# Tutoriel : Ajouter une nouvelle source de données

> Cible : ajouter une source X (banque, service web, capteur IoT, etc.) au hub. Ce tuto décrit le pattern complet utilisé pour Gmail, Calendar, Drive, Garmin, etc.

## TL;DR

Une source de données = 5 morceaux de code à créer :

1. **Modèle SQLAlchemy** dans `hub-core/src/db/models/<source>.py`
2. **Migration Alembic** dans `hub-core/alembic/versions/<hash>_add_<source>.py`
3. **Endpoints API** dans `hub-core/src/api/v1/<source>.py`
4. **Job scheduler** dans `hub-core/src/scheduler.py` (auto-sync) ou connecteur ingest dans `hub-ingest/src/connectors/<source>.py`
5. **Page frontend** dans `hub-frontend/app/<source>/page.tsx`

## Étape 1 — Modèle SQLAlchemy

Fichier : `hub-core/src/db/models/streaming_activity.py`

```python
"""Modele StreamingActivity : 1 visionnage de film/episode trace via Trakt.tv."""

from __future__ import annotations
from datetime import UTC, datetime
from uuid import UUID, uuid4

from sqlalchemy import DateTime, Integer, String, func
from sqlalchemy.orm import Mapped, mapped_column

from src.db.base import Base


class StreamingActivity(Base):
    __tablename__ = "streaming_activities"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    source: Mapped[str] = mapped_column(String(20), nullable=False, index=True)
    """trakt | plex | jellyfin | manual"""

    external_id: Mapped[str] = mapped_column(String(100), unique=True, index=True)
    item_type: Mapped[str] = mapped_column(String(20), nullable=False)  # movie | episode
    title: Mapped[str] = mapped_column(String(300), nullable=False)
    year: Mapped[int | None] = mapped_column(Integer, nullable=True)
    season: Mapped[int | None] = mapped_column(Integer, nullable=True)
    episode: Mapped[int | None] = mapped_column(Integer, nullable=True)
    runtime_minutes: Mapped[int | None] = mapped_column(Integer, nullable=True)
    watched_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)

    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(UTC),
        server_default=func.now(),
    )
```

Puis ajoute l'import dans `hub-core/src/db/models/__init__.py` (sinon Alembic ne le détecte pas).

## Étape 2 — Migration Alembic

```powershell
cd C:\hub\hub-core
.venv\Scripts\Activate.ps1
alembic revision --autogenerate -m "add_streaming_activities"
```

Vérifie le fichier généré dans `alembic/versions/`. Adjuste si besoin (down_revision pointe vers la dernière migration).

```powershell
alembic upgrade head
```

## Étape 3 — Endpoints API

Fichier : `hub-core/src/api/v1/streaming.py`

Pattern minimum :

```python
from fastapi import APIRouter, Depends, HTTPException, status
from src.db.session import get_db
from src.db.models import StreamingActivity

router = APIRouter(prefix="/streaming", tags=["streaming"])


class SyncRequest(BaseModel):
    days_back: int = 30


@router.post("/sync", response_model=SyncResponse)
async def sync_streaming(
    payload: SyncRequest,
    db: Annotated[AsyncSession, Depends(get_db)],
) -> SyncResponse:
    """Pull la history Trakt et upsert en DB."""
    # 1. recupere access_token via OAuth (cf. oauth_google.py pattern)
    # 2. GET https://api.trakt.tv/sync/history
    # 3. upsert chaque item via external_id
    # 4. broadcast SSE 'streaming_synced'
    ...


@router.get("/history", response_model=list[StreamingActivityOut])
async def list_history(...):
    ...
```

Puis ajoute le router dans `hub-core/src/api/v1/__init__.py` :

```python
from src.api.v1 import (..., streaming, ...)
router.include_router(streaming.router)
```

## Étape 4 — Job scheduler (auto-sync)

Dans `hub-core/src/scheduler.py` :

```python
async def _job_streaming(db: AsyncSession) -> str:
    from src.api.v1.streaming import SyncRequest, sync_streaming
    res = await sync_streaming(SyncRequest(days_back=7), db=db)
    return f"ingested={res.ingested} updated={res.updated}"

# Dans start_scheduler() :
_add_job("streaming", _job_streaming, settings.scheduler_streaming_minutes)

# Dans run_job_now() :
factories = {..., "streaming": _job_streaming}
```

Ajoute `scheduler_streaming_minutes: int = 60` dans `hub-core/src/core/config.py`.

## Étape 5 — Page frontend

Fichier : `hub-frontend/app/streaming/page.tsx`

```tsx
'use client'

import useSWR from 'swr'
import { Widget } from '@/components/widget'
import { getBaseUrl } from '@/lib/api'

export default function StreamingPage() {
  const { data } = useSWR(
    `${getBaseUrl()}/v1/streaming/history`,
    (url) => fetch(url).then(r => r.json())
  )
  return (
    <div className="p-6">
      <h1 className="text-2xl font-semibold">Streaming</h1>
      {/* ... */}
    </div>
  )
}
```

Puis ajoute la nav dans `hub-frontend/components/sidebar.tsx` et `mobile-bottom-nav.tsx` si pertinent.

## Étape bonus — Connector ingest (alternative au scheduler hub-core)

Si la source nécessite des tâches lourdes (parsing PDF, scraping, etc.), préfère un connecteur dans `hub-ingest/` plutôt qu'un job dans `hub-core/scheduler.py`.

Pattern : voir `hub-ingest/src/connectors/insights_alerts.py` ou `privacy_reminders.py`.

## Règles à respecter (rappel `CLAUDE.md` global)

1. **Aucune fake data** — pas de fixtures inventées dans les tests, pas de `PREVIEW_*` hardcodés en frontend
2. **Local d'abord** — la source ne doit JAMAIS envoyer les données du hub vers un tiers
3. **Tout gratuit** — vérifie que l'API est gratuite ou a un free tier suffisant
4. **Event sourcing** — pour les sources externes lourdes (Timeline, Takeout), dump raw avant transform (cf. ADR-0002)
5. **Idempotence** — `external_id` UNIQUE partout. 2 syncs back-to-back = 0 doublon

## Checklist avant merge

- [ ] Modèle + migration testés en local (Postgres et SQLite)
- [ ] Endpoints répondent aux 3 verbes minimum (POST sync, GET list, GET stats)
- [ ] Tests pytest async pour au moins le sync
- [ ] Job scheduler ne crash pas si la source est down (try/except + log)
- [ ] Page frontend gère loading + error + empty states
- [ ] Mobile-friendly (touch targets ≥40px, tables overflow-x-auto)
- [ ] Variables d'env documentées dans `.env.example`
- [ ] ADR si choix structurant (vendor lock-in, schema major)
