# As-Built Schema & reparsers-v2 Seam (G6 / Seam 3)

This is the shared contract between the deploy tool (`deploy/`) and the as-built document
generator (`reparsers/`, a.k.a. reparsers-v2). It defines what the deploy tool emits at check-out
and how reparsers-v2 consumes it. Written from an archaeology pass over reparsers-v2 (2026-06-13).

## 1. How reparsers-v2 works today

reparsers-v2 is a Word (.docm) **as-built document generator**. Data flow (`cli.py` →
`document_builder.build_final_document`):

1. **Per-product parsers** ingest *tech-support packages* and config captures and emit a
   product-specific parsed dict:
   | Product | Parser | Input | Output shape (hostname source) |
   |---|---|---|---|
   | PowerEdge | `poweredge_parser.py` | iDRAC **TSR** zip (CIM XML + JSON) | CIM instances grouped by CLASSNAME (`DCIM_SystemView[0].HostName`) + `FirmwareSummary` |
   | PowerStore | `powerstore_parser.py` | config capture | `appliance[0].name` |
   | PowerScale | `powerscale_parser.py` | config backup | `cluster.name` |
   | PowerVault | `powervault_parser.py` | store logs | `system-information[0].system-name` |
   | PowerSwitch | `powerswitch_parser.py` | `show run` / sosreport | `SystemSummary[0].Hostname` |
   | VMware/ESXi | `parsers/esxi_parser.py` | esxi `.txt` | (fixed "VMware Infrastructure") |
2. **`--metadata <json>`** supplies engagement context, NOT device config — customer block, dell
   contacts (validated roles: Solutions Architect / Delivery Resource / Project Manager), engagement
   notes. Drives the cover page (`@1..@6`), Appendix A (contacts), Appendix B (customer), Appendix D
   (notes).
3. **`document_builder`** maps each product's parsed dict through per-product `.docx` templates
   (`mapper.render_template`), composes them with `docxcompose`, appends global + dynamic appendices,
   renumbers tables, and saves the merged `.docm`.

**Key fact:** reparsers-v2's parsers expect *vendor-native* tech-support formats (CIM XML TSRs, config
backups). It has **no source today for the deployment record** — what was actually deployed, the final
management IPs, the credentials set, and whether post-deploy validation passed.

## 2. The seam decision

The deploy tool already produces exactly that missing record. Rather than reshape the deploy tool's
SSH/REST captures into vendor-native TSR formats (fragile — `show vlt all` text is not an iDRAC CIM
TSR), the seam is **complementary**:

```
reparsers-v2 TSR parsers   ─►  detailed per-product CONFIG sections   ┐
deploy tool as-built JSON  ─►  DEPLOYMENT SUMMARY + credentials/        ├─►  one as-built .docm
                                validation appendix                    ┘
diagrammer draw.io export  ─►  topology figure (manual insert for now) ┘
```

- **deploy tool owns**: what we deployed + verified (models, serials, final IPs, credentials,
  validation pass/fail, captured-artifact inventory).
- **reparsers-v2 owns**: detailed config extraction from TSRs + document assembly.
- **importer bridges**: a new reparsers-v2 input type (`asbuilt`) that turns the deploy package into
  a Deployment Summary section + a credentials/validation appendix, using the existing
  render/compose machinery.

The deploy tool's `validation_artifacts/<tenant>/` captures (Seam 2 / G5) remain useful as *raw TSR
substitutes* for the existing parsers where formats line up (e.g. the OS10 `<switch>_diagrammer.zip`
already feeds the diagrammer; switch `show run` captures can feed `powerswitch_parser`), but that is
a per-product opportunistic mapping, not the primary seam.

## 3. The as-built package (deploy tool → reparsers-v2)

At check-out the deploy tool writes `checkout_<tenant>/` containing two versioned files. Both carry
`schema_version` (currently **1**; bump on any breaking change to the device shape).

### 3a. `asbuilt_credentials_<tenant>.json` — the credential record (PLAINTEXT)
Produced by `vault.ExportPlaintext`. **Sensitive**: merged into the as-built then deleted; never
committed.
```json
{
  "schema_version": 1,
  "tenant": "project-gotham",
  "engineer": "Drew Smith",
  "generated_at": "2026-06-13T21:40:00Z",
  "devices": [
    {
      "device": "project-gotham-os10-01",
      "model": "PowerSwitch S5248F-ON (OS10)",
      "serial": "CN0ABC1",
      "username": "admin",
      "password": "<factory/login>",
      "os_password": "<password the deploy SET, if any>",
      "final_mgmt_ip": "10.20.1.10",
      "updated_at": "2026-06-13T21:30:00Z",
      "validation": {
        "overall": "pass",
        "checked_at": "2026-06-13T21:38:00Z",
        "checks": [
          {"name": "VLT 100 peer status", "status": "pass", "detail": "peer up"}
        ]
      }
    }
  ]
}
```

### 3b. `manifest.json` — the deployment manifest (non-secret)
Produced by `buildCheckoutManifest`. Safe to keep; the importer's primary input.
```json
{
  "schema_version": 1,
  "tenant": "project-gotham",
  "engineer": "Drew Smith",
  "generated_at": "2026-06-13T21:40:00Z",
  "devices": [
    {
      "name": "project-gotham-os10-01",
      "model": "PowerSwitch S5248F-ON (OS10)",
      "serial": "CN0ABC1",
      "final_mgmt_ip": "10.20.1.10",
      "deploy_status": "done",
      "manual": false,
      "attempts": 1,
      "last_update": "2026-06-13T21:30:00Z",
      "validation_overall": "pass",
      "validation_checks": ["VLT 100 peer status: pass (peer up)"]
    }
  ],
  "artifacts": [
    {"path": "project-gotham/project-gotham-os10-01/cli_output/show-vlt-all.txt", "bytes": 412, "mod_time": "..."},
    {"path": "project-gotham/project-gotham-os10-01_diagrammer.zip", "bytes": 2048, "mod_time": "..."}
  ]
}
```
Artifact `path` values are relative to `validation_artifacts/`.

### Field provenance
| Field | Source in deploy tool |
|---|---|
| `model`, `serial` | NetBox device (carried in the session bundle) |
| `final_mgmt_ip` | `vault.RecordDeployment` (per-product intent IP) |
| `username` / `password` | pull-tag creds entered onsite (vault) |
| `os_password` | generated unique password the deploy set (G3a) |
| `deploy_status`, `manual`, `attempts` | `deployment_state_<tenant>.json` (G4) |
| `validation_*` | Phase 5 `ValidateCluster` results (per-check) |
| `artifacts[]` | `validation_artifacts/<tenant>/` tree (Seam 2 + G5) |

## 4. Planned reparsers-v2 importer

New module `reparsers/asbuilt_importer.py` + CLI flag `--asbuilt <checkout_dir>`:

1. Load `manifest.json` (+ optionally `asbuilt_credentials_<tenant>.json` when a credentials appendix
   is wanted). Validate `schema_version`.
2. Emit a parsed-data entry `{"type": "asbuilt", "data": {...}}` into `all_parsed_data`, ahead of the
   product config sections.
3. `document_builder` gains an `asbuilt` branch that renders two new templates:
   - **`templates/asbuilt/01_DeploymentSummary.docx`** — per-device table: hostname, model, serial,
     final mgmt IP, deploy status (incl. "manual"), validation result. One row per device.
   - **`templates/asbuilt/Appendix_Credentials.docx`** (opt-in, gated on the credentials file being
     present) — hostname, login user/password, OS password, final IP. Carries the same
     handle-and-delete warning the deploy tool prints.
4. The manifest's `artifacts[]` inventory is rendered as an evidence list; where an artifact format
   matches an existing parser (e.g. switch `show run`), a later enhancement can route it through that
   parser for a full config section.

### Why a new type rather than `--metadata`
`--metadata` is a single engagement block (customer/contacts/notes), not a per-device array. The
deployment record is per-device and validation-bearing, so it belongs as its own input type with its
own templates — and it must be optional so reparsers-v2 keeps working from TSRs alone.

## 5. Status & next steps
- [x] Archaeology of reparsers-v2 data flow (this doc).
- [x] Deploy tool emits both files with `schema_version` (deploy: `vault.go`, `checkout.go`).
- [ ] `reparsers/asbuilt_importer.py` + `--asbuilt` flag.
- [ ] `templates/asbuilt/01_DeploymentSummary.docx` and `Appendix_Credentials.docx`.
- [ ] `document_builder` `asbuilt` branch + ordering.
- [ ] Round-trip test: deploy check-out package → reparsers `--asbuilt` → section present in the .docm.

The importer + templates are the build work; gating them behind the field test is deliberate, since
the first real check-out package will confirm the field shapes before templates are cut.
