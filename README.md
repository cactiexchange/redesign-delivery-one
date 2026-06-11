# redesign-delivery-one

Integration workspace tying together three projects into a single end-to-end delivery pipeline.

## Subprojects

| Folder | Repo | Role |
|--------|------|------|
| `deploy/` | [DrewsDatacenterDeployV5](https://github.com/cactiexchange/DrewsDatacenterDeployV5) | Go orchestrator — NetBox intake, encrypted credential vault, phase-ordered deployment (OS10 / SONiC / ESXi / PowerStore / ME5 / PowerScale / Unity), post-deployment validation, as-built JSON export |
| `diagrammer/` | [os10-diagrammer-v2](https://github.com/cactiexchange/os10-diagrammer-v2) | React+Vite + Flask — 6-step design Wizard, OS10 support-bundle parser, draw.io/N2G export, NetBox read/write |
| `reparsers/` | [reparsers-v2](https://github.com/cactiexchange/reparsers-v2) | As-built document generator — parses tech-support packages and log files into a final customer deliverable |

## Intended pipeline

```
diagrammer Wizard
       │  (design intent → NetBox)
       ▼
   NetBox (source of truth)
       │  (FetchNetboxData / workbook intake)
       ▼
   deploy tool
       │  deploy + validate + asbuilt_credentials_<tenant>.json
       ▼
   reparsers-v2
       │  (structured JSON + TSRs + draw.io figure → Word/PDF)
       ▼
   Final as-built document
```

## Integration seams (active work)

### Seam 1 — Diagrammer Wizard → seeder
The diagrammer Wizard already collects switches, L3 config, and cabling — the same data the intake
workbook captures. A `--from-json` path in `deploy/devops_netbox_seeder.py` allows the diagrammer
to export a project-intent JSON that flows through the seeder's validation/dry-run/push pipeline
without bypassing DAG checks.

### Seam 2 — Validation artifacts → diagrammer ✅ DONE
During Phase 5 switch validation the deploy tool captures the TSR-equivalent CLI set (11 files for
OS10, 5 for SONiC) into `validation_artifacts/<tenant>/<switch>/cli_output/` and packs a
`<switch>_diagrammer.zip` whose entry layout matches what `diagrammer/backend/archive_handler.py`
ingests. Drop the zip into the diagrammer for an as-deployed topology diagram — no tech-support
bundle pull needed. Handoff verified end-to-end against the diagrammer's real parsers
(deploy commit `f068184`).

### Seam 3 — Deploy tool → reparsers-v2
`asbuilt_credentials_<tenant>.json` (produced at end of every deployment run) feeds reparsers-v2
as structured input alongside parsed TSRs. The shared schema is defined in `docs/asbuilt_schema.md`
(TBD — requires reading reparsers-v2 template internals).

## Getting started

```powershell
git clone --recurse-submodules https://github.com/cactiexchange/redesign-delivery-one.git
cd redesign-delivery-one
```

To update all submodules to their latest upstream commits:

```powershell
git submodule update --remote --merge
```

Each subproject has its own README for build/run instructions.
