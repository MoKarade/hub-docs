# ADR-0002 — Event sourcing pour l'ingest

**Date :** 2026-04-28
**Statut :** Acceptée

## Contexte

Le hub ingère des données depuis plusieurs sources externes (banque, Google, etc.). Question : on stocke directement en DB normalisée, ou on garde aussi les data brutes ?

## Décision

**Event sourcing partiel : on stocke les data BRUTES telles que reçues, en plus de la DB normalisée.**

Structure :
```
raw_events/
├── bank_csv/
│   └── 2026-04-28/
│       └── 143052_124000.json     ← raw export, immutable
├── google_gmail/
│   └── 2026-04-28/
│       └── ...
└── ...
```

Et la DB Postgres a la "vue" normalisée, reconstructible à partir des raw events si besoin.

## Pourquoi

1. **Robustesse contre les bugs de pipeline.** Si un connecteur a un bug, on garde quand même la raw data. On peut re-runner la pipeline corrigée.
2. **Robustesse contre les changements de schéma.** Si on ajoute un champ dans le modèle, on peut re-projeter l'historique depuis les raw events.
3. **Robustesse contre l'API source qui change/disparaît.** Plaid coupe demain ? On a déjà tout l'historique en local. Google deprecate Photos API ? Idem.
4. **Auditabilité.** On voit exactement ce que la source a envoyé.
5. **Pas si cher.** Compression naturelle (zstd via restic), volume gérable.

## Trade-offs acceptés

- **Volume disque plus élevé.** OK : 250 GB total estimé sur SSD, pas un souci.
- **Complexité légèrement accrue.** Chaque connecteur doit `dump_raw()` ET `transform_and_insert()`. Mitigation : interface claire dans `connectors/base.py`.
- **Devoir gérer la dédup.** Si on rejoue, ne pas insérer en double. Mitigation : idempotence via clé naturelle (hash, ID source).

## Alternatives rejetées

- **DB seule, pas de raw events.** Plus simple mais on perd la robustesse. Si on découvre un bug 6 mois plus tard, la data est déjà polluée.
- **Event sourcing pur (Kafka, EventStoreDB).** Overkill pour usage perso. Le filesystem suffit.

## Conséquences

- Chaque connecteur appelle `self.dump_raw(payload)` avant de transformer.
- Les pipelines peuvent être re-runnées : `python -m src.replay <connector> <date_range>`.
- Backup couvre `raw_events/` (déjà inclus dans le restic config).
