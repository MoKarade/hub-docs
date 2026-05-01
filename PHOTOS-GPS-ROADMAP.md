# Photos GPS / EXIF / Faces — Roadmap Phase 3c+

> Document de design pour l'enrichissement des photos avec geolocalisation,
> EXIF complet, face recognition et search semantique.

## Statut

- ✅ Modèle DB enrichi : champs `latitude`, `longitude`, `altitude_m`, `location_name`, `exif_data`, `faces_count`, `faces_data`, `clip_embedding`
- ✅ Index composite `(latitude, longitude)` pour bbox queries
- ⏸️ Implémentation : reportée (GPS extraction nécessite download des bytes)

## Phase 3c+ : Extraction GPS (à implémenter)

### Workflow

1. À l'import (Picker `_parse_picker_item`), pour chaque photo :
   - Download les bytes via `baseUrl + =d` (full resolution) avec Bearer token
   - Parse l'EXIF avec `exifread` (lib Python) ou `Pillow.ExifTags`
   - Extract `GPSInfo` : tag `34853` qui contient lat/lng/alt/timestamp
   - Convertir DMS → décimal
   - Reverse geocode optionnel via Nominatim (gratuit, OSM) pour `location_name`

### Code template

```python
# hub-core/src/services/photo_gps.py

import exifread
import httpx
from typing import Tuple

def _dms_to_decimal(dms: list, ref: str) -> float:
    """Convertit DMS (degrés, minutes, secondes) en décimal signé."""
    deg, mn, sec = dms
    val = deg + mn / 60 + sec / 3600
    return -val if ref in ('S', 'W') else val


async def extract_gps_from_photo(
    base_url: str, access_token: str
) -> tuple[float | None, float | None, float | None, dict | None]:
    """Telecharge les bytes + parse EXIF. Retourne (lat, lng, alt_m, exif_dict)."""
    async with httpx.AsyncClient(timeout=30.0) as client:
        # =d suffix = original resolution (vs =w200-h200-c qui resize)
        r = await client.get(
            f"{base_url}=d",
            headers={"Authorization": f"Bearer {access_token}"},
        )
        r.raise_for_status()
        bytes_data = r.content

    import io
    tags = exifread.process_file(io.BytesIO(bytes_data), details=False)

    lat = lng = alt = None
    if 'GPS GPSLatitude' in tags and 'GPS GPSLatitudeRef' in tags:
        lat_dms = [float(v.num) / v.den for v in tags['GPS GPSLatitude'].values]
        lat = _dms_to_decimal(lat_dms, str(tags['GPS GPSLatitudeRef']))
    if 'GPS GPSLongitude' in tags and 'GPS GPSLongitudeRef' in tags:
        lng_dms = [float(v.num) / v.den for v in tags['GPS GPSLongitude'].values]
        lng = _dms_to_decimal(lng_dms, str(tags['GPS GPSLongitudeRef']))
    if 'GPS GPSAltitude' in tags:
        v = tags['GPS GPSAltitude'].values[0]
        alt = float(v.num) / v.den

    # Stocke un sous-ensemble EXIF utile (pas le RAW complet)
    exif = {
        str(k): str(v) for k, v in tags.items()
        if k.startswith(('Image', 'EXIF', 'GPS')) and 'Thumbnail' not in k
    }
    return lat, lng, alt, exif
```

### Endpoint enrichissement

```python
@router.post("/photos/enrich-gps")
async def enrich_gps(db: Annotated[AsyncSession, Depends(get_db)]):
    """Pour chaque photo SANS GPS (latitude IS NULL), tente extraction EXIF."""
    photos = (
        await db.execute(
            select(Photo).where(Photo.latitude.is_(None), Photo.base_url.isnot(None))
        )
    ).scalars().all()

    access_token = await _resolve_token(db, "marc.richard4@gmail.com")
    enriched = 0
    for p in photos:
        try:
            lat, lng, alt, exif = await extract_gps_from_photo(p.base_url, access_token)
            if lat and lng:
                p.latitude = lat
                p.longitude = lng
                p.altitude_m = alt
                p.exif_data = exif
                enriched += 1
        except Exception as e:
            logger.warning("gps_extract_failed: %s err=%r", p.media_id, e)
    await db.commit()
    return {"enriched": enriched, "total": len(photos)}
```

### Frontend : carte + filtre par lieu

```tsx
// /photos avec filtre par bbox + Leaflet markers
import { MapContainer, Marker, TileLayer } from 'react-leaflet'

// Toggle "Vue carte"
<MapContainer center={[46.8, -71.2]} zoom={10}>
  <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
  {photos.filter(p => p.latitude && p.longitude).map(p => (
    <Marker position={[p.latitude, p.longitude]} key={p.id}>
      ...
    </Marker>
  ))}
</MapContainer>
```

Plus Filtre "À ce lieu" : reverse geocode au click d'un marker pour grouper par
ville/pays.

## Phase 7+ : Face recognition

Bibliothèque : `face_recognition` (Python, basé sur dlib). Modèle ~100MB,
inférence ~50ms/photo.

```python
import face_recognition
img = face_recognition.load_image_file(io.BytesIO(bytes_data))
encodings = face_recognition.face_encodings(img)
locations = face_recognition.face_locations(img)
```

Ensuite clustering DBSCAN sur les `encodings` pour grouper par personne.
UI : cluster preview + click pour nommer la personne. Update `Person` table.

## Phase 7+ : CLIP embeddings (search sémantique)

Bibliothèque : `open_clip` (PyTorch) ou `clip-anytorch`. Modèle ~600MB.
Pour chaque photo : encode l'image → vecteur 512 floats. Stocker dans
`clip_embedding` (en pgvector idéalement, ou JSON pour MVP).

Search : encode la query texte → cosine similarity vs tous les embeddings.

```python
import open_clip
model, _, preprocess = open_clip.create_model_and_transforms('ViT-B-32')
tokenizer = open_clip.get_tokenizer('ViT-B-32')

# Index
img = preprocess(Image.open(io.BytesIO(bytes_data))).unsqueeze(0)
img_emb = model.encode_image(img).detach().numpy().tolist()

# Query
text = tokenizer(['photo de vacances Maroc'])
text_emb = model.encode_text(text).detach().numpy()
# Cosine similarity vs photos.clip_embedding
```

Avec pgvector : `ORDER BY clip_embedding <=> :query_emb LIMIT 50`.

## Coûts / dépendances

| Phase | Effort | Modèle | Storage | Privacy |
|---|---|---|---|---|
| GPS only | 1-2h | exifread (free) | +EXIF dict / photo | Local |
| Reverse geocode | +30min | Nominatim free OSM | +location_name | Local |
| Face recognition | +1 jour | dlib 100MB | +faces array (1KB/face) | Local |
| CLIP | +1 jour | OpenCLIP 600MB | +512 floats / photo | Local |

## TODO ordre

1. **GPS extraction** (la base, déclenche tout le reste)
2. **Reverse geocode** (pour `location_name` lisible)
3. **Frontend filtre par lieu** + carte
4. **CLIP indexing** au sync
5. **CLIP search UI** dans /search
6. **Face recognition** + clustering personnes (le plus lourd, à la fin)
