# Sessions Claude Code — archive

Ce dossier archive les sessions Claude Code importantes du projet, ainsi que
la **mémoire projet** que Claude utilise comme contexte permanent.

## Contenu

| Fichier | Rôle |
|---|---|
| `setup-autre-pc.md` | **Comment reprendre le projet depuis un autre PC** (cf. ci-dessous) |
| `2026-04-28_session1_transcript.md` | Transcript complet de la session inaugurale (Phase 0+1+2) |
| `memory/MEMORY.md` | Index des memories de Claude pour ce projet |
| `memory/project_banking.md` | Mémoire stable : banque Desjardins, 5 comptes, formats |

## Pour reprendre depuis un autre PC

→ Lis [`setup-autre-pc.md`](setup-autre-pc.md). Étapes résumées :

1. Installer prérequis (Git, Docker, Ollama, Node, gh)
2. `git clone` les 5 repos `hub-*` depuis github.com/MoKarade
3. Restaurer le `~/.claude/CLAUDE.md` global de Marc (depuis OneDrive ou copie directe)
4. Restaurer la mémoire projet via `memory/*.md`
5. Lancer la stack : `docker compose up`
6. Re-importer les données réelles via les CSV/PDF (privacy : jamais sur GitHub)
7. Lancer Claude Code dans `C:\hub` et lui demander de lire `JOURNAL.md`

## Privacy

Les transcripts de sessions peuvent contenir des **données personnelles
réelles de Marc** (montants, marchands, dates, adresses, soldes…) car le
projet exclut explicitement la fake data (règle 4 du CLAUDE.md global).

Le repo `hub-docs` est **PRIVÉ** sur GitHub — vérifié explicitement avant
chaque push. Ne JAMAIS le rendre public.
