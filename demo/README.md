# Demo packet

Self-contained material for demonstrating the deployment ecosystem **before the first hardware field
test** — everything here runs (or is shown) without datacenter hardware.

Open **[walkthrough.html](walkthrough.html)** — a presenter script with a suggested 10-minute cut.

## Artifacts

| File | What | How it was made |
|------|------|-----------------|
| `sample_peq.html` | Customer Pre-Engagement Requirements doc (11-device project) | Real `peq_generator.py` output |
| `sample_intent.json` | Diagrammer wizard export (Seam 1) | Validated through the seeder — seeds cleanly |
| `sample_checkout/manifest.json` | Check-out deployment manifest (non-secret) | Real deploy-tool check-out output |
| `sample_checkout/asbuilt_credentials_project-gotham.json` | The plaintext credential record | Real output shape |

All for the fictional tenant **project-gotham** (2 OS10 switches, 1 customer switch, 3 PowerEdge R770,
1 PowerStore 3200T, 4 PowerScale H7100).

> **The credentials in `asbuilt_credentials_project-gotham.json` are FABRICATED demo data** — well-known
> factory defaults (`calvin`, `admin`, `Password123#`) and made-up strings. No real secret is in this
> folder. It's included precisely to show what the sensitive file looks like and why the real one must be
> merged into the as-built and deleted.

The **diagrammer wizard** demo (design → export → seed) needs a Node build (`npm run dev` in
`diagrammer/`) — run it on a machine with Node; the file it exports matches `sample_intent.json`.
