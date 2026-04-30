### GDP Feature Catalog — Comprehensive Reference for Re‑implementation

This document enumerates the full feature set of the Glider Data Pipeline (GDP). It is designed as a contract:
maintainers can re‑implement the system (e.g., with different engines or frameworks) while preserving behaviour. Each
feature lists purpose, inputs/outputs, dependencies, and acceptance criteria.

---

### 1) Mission & Mode Orchestration

- Mission discovery & selection
    - Purpose: Select which deployments to process.
    - Modes: real‑time (current mission; `end_time=None`) and delayed (completed missions).
    - Inputs: Sensor Tracker deployments; CLI flags (`--live/--delayed`, `-ml` lists, ranges).
    - Outputs: Ordered list of mission dicts (platform, deployment_number, start/end time, mode).
    - Acceptance: Correct mission chosen for each mode; explicit lists honored; order predictable (latest first for
      delayed by default).

- Builder chain (per mission)
    - Before → Process → After builders executed in order.
    - Pluggable via handler lists with conditional No‑Op steps.
    - Acceptance: Steps run in defined sequence; conditional steps respect CLI/options.

---

### 2) Sensor Tracker Sync

- DB synchronization
    - Purpose: Refresh `Mission` model from external Sensor Tracker.
    - Inputs: Sensor Tracker API (host, auth), platform type.
    - Outputs: Upserted `Mission` rows.
    - Acceptance: New missions appear; end_time updates reflected; failures logged with retry/backoff policy.

- Metadata inputs
    - Purpose: Pull instruments, sensors, parameters for metadata generation.
    - Acceptance: Generator works with expected field shapes; degrades gracefully if optional fields missing.

---

### 3) Directory Management

- Mission directory provisioning
    - Roles: `ascii_path`, `netcdf_path`, `meta_json_path`, `erddap_config_path`.
    - Inputs: mode, deployment_number, base roots.
    - Outputs: Created/validated directories; `ProcessDirectory` rows saved.
    - Acceptance: Idempotent creation; permissions validated; paths queryable during later steps.

- Resource roots
    - Real‑time: `<resource_root>/<sfmc_group>/gliders/<platform_lower>/from-glider`.
    - Delayed: `<resource_root>/<PLATFORM_UPPER>/<start_time_short>/**/raw/**` (+ `raw/cache`).
    - Acceptance: Finder discovers correct files given settings; respects extension lists.

---

### 4) File Catalog & Idempotence

- File tracking database
    - `ProcessFiles` manages (mission_type, deployment_number, file_type, file_path, processed).
    - Types: `BINARY`, `ASCII`, `NETCDF` (`NC`/`NC_RT`), `ERDDAP_CONFIG`, `META_JSON`.
    - Acceptance: Discover steps create rows; processing steps update `processed`; queries return unprocessed lists
      reliably.

- Unmanaged file intake
    - ASCII directory scan to add `.dat` not previously tracked.
    - Acceptance: Newly dropped files are integrated on next run.

---

### 5) Pre‑processing (Before*)

- Data discovery
    - Real‑time finder: locate `SBD/TBD/MBD/NBD` filtered by mission start day.
    - Delayed finder: locate `DBD/EBD` in deployment `raw/` trees.
    - Acceptance: Correct extension filters; ignores earlier deployments; handles missing dirs gracefully.

- Reprocess/maintenance controls
    - Rebuild ASCII, binary, metadata; re‑upload NC; re‑generate ERDDAP XML.
    - Acceptance: Flags trigger only their target work; safe if outputs already present.

- Metadata discovery/creation kick‑off
    - Create meta JSON or notify with meta file paths; save as `META_JSON`.
    - Acceptance: Downstream consumers find the meta directory.

---

### 6) Core Processing (Process*)

- Binary → ASCII conversion
    - Purpose: Convert `*.*bd` files to `.dat` ASCII with caching.
    - Inputs: binary dir, ASCII output dir, shared cache dir.
    - Outputs: ASCII files, processed/unprocessed binary lists.
    - Acceptance: Converts valid inputs; logs/returns failures; cache reused across runs; idempotent.

- ASCII → NetCDF generation
    - Purpose: Produce NetCDF files from ASCII using metadata and filters.
    - Inputs: ASCII file list; `meta_json_path`; filters (`filter_distance`, `filter_points`, `filter_time`, `filter_z`,
      `tsint`).
    - Outputs: NetCDF files in `netcdf_path`; DB updates linking ASCII→NC.
    - Acceptance: CF‑compliant NCs with expected variables/attrs; filters applied; stable ordering.

- Progress & resilience
    - Progress bar/logging over N files; continues on per‑file failures; aggregates errors.

---

### 7) Post‑processing (After*)

- ERDDAP Dataset XML generation
    - Draft creation from actual NCs (tool/script/logic) + refinement with mission context.
    - Inject datasetID, title, ERDDAP‑visible NC directory, variable/global attributes.
    - Save final XML under `erddap_config_path`; register as `ERDDAP_CONFIG`.
    - Acceptance: Valid XML per ERDDAP; datasetID uniqueness; correct NC path for container.

- Uploads
    - Upload dataset XML to ERDDAP datasets/ dir.
    - Upload NetCDFs to ERDDAP data/ dir (per mission/mode subdirs).
    - Acceptance: Permission/ownership correct; re‑uploads safe; retries/backoff on transient errors.

- Backups & cleanup
    - Backup ERDDAP NC dir prior to refresh (configurable retention).
    - Cleanup intermediates unless `--keep_middle_process_files`.
    - Acceptance: Backups timestamped; cleanup respects policy; space reclaimed.

- Notifications
    - Slack summary of mission processing (success/failures, file counts, paths).
    - Acceptance: Single message per run/mode; sensitive data not leaked.

---

### 8) Metadata System

- Generation
    - From Sensor Tracker instruments/sensors/parameters via template renderers.
    - Output `META_JSON` files in `meta_json_path` with combined and/or per‑section docs.
    - Acceptance: Contains canonical variable semantics and global attrs used by NC/ERDDAP.

- Consumption
    - NC writer maps ASCII channels to CF variables with correct units/standard_names.
    - ERDDAP XML refiner aligns with same semantics (destinationName, ioos_category, axis roles).
    - Acceptance: NetCDF and XML remain consistent; no drift in naming/attrs.

- Evaluation/validation
    - Optional `--evaluate_nc_file` step to validate/fix NC outputs.

---

### 9) ERDDAP Join & Master datasets.xml (External Orchestration)

- Individual XML emission
    - GDP writes 1 XML per dataset/mission.

- External join (Rundeck)
    - Concatenate `start.xml` + all individual XMLs (validated/formatted) + `end.xml`.
    - Apply token substitutions and temporary CF/CDM patches.
    - Diff with previous `datasets.xml`; optional run of ERDDAP’s `GenerateDatasetsXml.sh addFillValueAttributes`.
    - Acceptance: Master `datasets.xml` valid; diffs reviewed; patches converge to zero over time as GDP improves XML.

- Reimplementation note
    - Provide a Python joiner (see separate doc) for deterministic, testable operation.

---

### 10) Configuration & Settings Contract

- FILE_TYPE, DIRECTORY_TYPE, MISSION_TYPE, REALTIME/DELAYED extensions.
- ERDDAP_CONFIG (container NC root, datasets dir, data dir).
- SLOCUM_SHARED_CACHE_DIR.
- SENSOR_TRACKER_HOST (+ auth), SLACK_*.
- Acceptance: All configurable via settings/env; documented defaults; validated at startup.

---

### 11) CLI & User Experience

- Management command: `slocum_run`
    - Modes: `--process_realtime_mission`, `--process_delayed_mission`.
    - Selection: `-ml/--mission_list`, `-m/--missions` (single/range).
    - Processing: `-a/--asciiProcess`, `-b/--binProcess`.
    - Filters: `--filter_distance`, `--filter_points`, `--filter_time`, `--filter_z`, `--tsint`.
    - ERDDAP/ops: `-c/--config_erddap`, `--upload`, `--keep_middle_process_files`, `--slack_notification`, `--test_run`,
      `--hide_process_bar`.
    - Acceptance: Options validated; invalid combos rejected; help text complete.

---

### 12) Logging & Error Handling

- Structured logging per step with mission context.
- Summarized errors via runner’s `raise_error`.
- Separate logs for skipped/bad XML (ERDDAP join), failed input files, etc.
- Acceptance: Operators can diagnose failures from logs; non‑zero exit codes on fatal conditions.

---

### 13) Database Model Behaviour

- Mission: upserts from Sensor Tracker.
- MissionProcess: per‑mode status/notification gating.
- ProcessDirectory: canonical dirs per mission/mode/role.
- ProcessFiles: file catalog + status with helpers (`get_unprocessed_file`, `save_*`).
- Acceptance: Migration‑safe; constraints enforced; admin scripts available.

---

### 14) Engines & Extensibility

- Engine abstraction
    - Current: GUTILS_mod + CEOTR wrappers.
    - Future: pluggable (e.g., `pyglider`) via stable interface (`process_bin`, `process_ascii_file`‑like contracts).
    - Acceptance: Swap engines without changing factories; outputs remain functionally equivalent.

- Step handler cookbook
    - Handlers + factories to add/remove steps; No‑Op when not applicable.
    - Acceptance: New steps integrate with `BaseProcessBuilder` and `BaseStepManager` lifecycles.

---

### 15) Performance & Scaling (Expectations)

- Auto‑delayed PROCESS_LIMIT (e.g., 1 mission/run) to bound batch time.
- Idempotent re‑runs; incremental progression across cron/timers.
- Acceptance: Real‑time cadence matches telemetry arrival; delayed completes within maintenance windows.

---

### 16) Security & Ops

- Permissions on ERDDAP volumes (UID/GID alignment); paths visible inside container.
- Secret handling for Sensor Tracker and Slack.
- Acceptance: No secrets in logs; minimal privileges for uploads; clear failure on permission errors.

---

### 17) Testing & Samples

- Unit tests for factories/handlers (including ERDDAP config generation with resources).
- Integration tests: sample missions for ASCII→NC and XML generation.
- Acceptance: CI green on representative fixtures; reproducible outputs across environments.

---

### 18) Documentation Deliverables

- Architecture overview (with Mermaid diagrams)
- Pre/Core/Post processing deep‑dives
- Real‑time vs Delayed runbooks
- DB & external API guide (Sensor Tracker, ERDDAP, Slack)
- Metadata lifecycle & schema
- ERDDAP layer & external join analysis
- Configuration & Deployment Matrix (this doc)
- GUTILS_mod integration & migration (to `pyglider`)

---

### 19) Acceptance Criteria Summary (per Stage)

- Pre: correct discovery, dirs created, `META_JSON` saved, file catalog updated, reprocess flags honored.
- Core: binary→ASCII idempotent; ASCII→NC CF‑compliant; filters applied; DB updates consistent.
- Post: valid ERDDAP XML with correct datasetID and paths; uploads succeed; backups & cleanup per policy; Slack posted.
- External: master `datasets.xml` valid; diffs reviewed; minimal/no ad‑hoc patches over time.

---

### 20) Re‑implementation Guidance

- Maintain DB model semantics (or equivalent stores) for idempotence and traceability.
- Keep interfaces stable for engine calls (`process_bin`, `process_ascii_file` contract) to allow drop‑in swaps.
- Drive all paths via settings/env + CLI; avoid hardcoded filesystem paths.
- Move any “join” or “patch” logic upstream into generation steps over time to reduce ops friction.

This catalog is intended to be exhaustive enough that a team can re‑implement GDP with different internals while keeping
all external behaviours and operational guarantees intact. If you want, I can turn this into a checklist with test cases
per feature for acceptance testing in a new implementation.