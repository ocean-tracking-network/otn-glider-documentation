### End‑to‑End GDP Processing — Master Sequential Diagram (Implementation‑Agnostic)

This master sequence is a canonical reference for maintainers re‑implementing the pipeline while preserving or
referencing behaviours. It spans: mission sync/selection, pre‑processing, core processing (binary→ASCII→NetCDF),
post‑processing (ERDDAP XML + uploads), and external master `datasets.xml` assembly. It abstracts away specific
libraries (GUTILS, pyglider, etc.) and focuses on responsibilities, inputs/outputs, and coordination.

```mermaid
sequenceDiagram
    autonumber
    participant O as "Operator / CRON / Rundeck"
    participant CLI as "Pipeline CLI"
    participant CFG as "Settings / Env"
    participant ST as "Sensor Tracker API"
    participant DB as "DB Models"
    participant PM as "Pipeline Manager"
    participant MB as "Mission Builder"
    participant RUN as "Mission Runner"
    participant PRE as "Before Steps"
    participant CORE as "Process Steps"
    participant ENG as "Slocum Engine"
    participant POST as "After Steps"
    participant ERF as "ERDDAP XML Generator"
    participant UP as "Uploaders"
    participant BK as "Backup"
    participant SLK as "Slack Notifier"
    participant FS as "Filesystem"
    participant EXT as "External Joiner"
    participant ERD as "ERDDAP Server"
    O ->> CLI: Invoke with CLI options
    CLI ->> CFG: Load settings
    CLI ->> PM: start(platform, options)
    PM ->> ST: Fetch deployments, instruments, parameters
    ST -->> PM: Deployment records
    PM ->> DB: Mission.update_or_create_from_dict
    PM ->> MB: build_missions(platform, options)
    MB ->> DB: Query selected missions
    DB -->> MB: Selected mission rows
    MB -->> PM: mission_list
    PM ->> RUN: Create runner

    loop For each mission
        RUN ->> PRE: Execute before steps
        PRE ->> CFG: Resolve directory roles
        PRE ->> DB: Save ProcessDirectory

        alt Reprocess or maintenance enabled
            PRE ->> DB: Update ProcessFiles flags
        end

        alt Realtime mode
            PRE ->> FS: Locate SBD / TBD / MBD / NBD files
        else Delayed mode
            PRE ->> FS: Locate DBD / EBD files
        end

        PRE ->> DB: Create BINARY / ASCII ProcessFiles
        RUN ->> CORE: Execute process steps
        CORE ->> DB: Get unprocessed BINARY files

        alt Binaries found
            CORE ->> ENG: Convert binary to ASCII
            ENG -->> CORE: Processed binaries and ASCII files
            CORE ->> DB: Update ProcessFiles
        else No binaries
            CORE -->> RUN: No-op
        end

        CORE ->> DB: Get unprocessed ASCII files

        alt ASCII found
            CORE ->> CFG: Read filters
            CORE ->> ENG: Create NetCDF files
            ENG -->> CORE: NetCDF results
            CORE ->> DB: Update ProcessFiles
        else No ASCII
            CORE -->> RUN: No-op
        end

        RUN ->> POST: Execute after steps
        POST ->> ERF: Build draft XML from NetCDF
        ERF -->> POST: draft_xml
        POST ->> ERF: Refine XML
        ERF -->> POST: final_xml_path
        POST ->> DB: Create ERDDAP_CONFIG ProcessFiles

        alt Backup enabled
            POST ->> BK: Backup ERDDAP NC directory
            BK -->> POST: backup_ok
        end

        alt Upload enabled
            POST ->> UP: Upload XML
            UP -->> ERD: Write datasets config
            POST ->> UP: Upload NetCDF files
            UP -->> ERD: Write NetCDF products
        end

        alt Cleanup intermediates
            POST ->> FS: Remove temporary files
        end

        POST ->> SLK: Send summary
        SLK -->> POST: ack
        RUN ->> RUN: Increment counter and check limit
        RUN -->> PM: Mission result
    end

    O ->> EXT: Run join_datasets
    EXT ->> FS: Read templates and individual XML files
    EXT ->> EXT: Validate and format XML
    EXT ->> EXT: Apply rules and token substitutions
    EXT ->> FS: Write datasets.xml, backups, and logs
    EXT -->> O: status
    O ->> ERD: Run GenerateDatasetsXml.sh
    ERD -->> O: Updated datasets.xml
```

---

### Notes & Contracts (for Re‑implementation)

- Inputs/Outputs
    - All file paths must be resolved via a directory role abstraction (e.g., `ProcessDirectory`), not hardcoded
      strings.
    - All artifact paths (raw, ASCII, NC, XML, META_JSON) are recorded in a file catalog (e.g., `ProcessFiles`) to
      ensure idempotence.

- Modes & Selection
    - Real‑time processes current mission (no `end_time`), delayed processes completed missions; explicit lists
      override.
    - Auto‑delayed runner should support a `PROCESS_LIMIT` to bound batch runs.

- Engine Interchangeability
    - Engine API must expose: `convert_bin_to_ascii(...)` and `create_netcdf(...)` equivalents that accept filters, meta
      dir, and time bounds.
    - Return structures must include success/failure lists for accurate DB updates and logs.

- ERDDAP XML
    - Two‑stage generation (draft + refine) or an equivalent single stage that guarantees: datasetID, title,
      ERDDAP-visible NC dir, and correct variable/global attributes.

- External Joiner
    - Validates individual XML, assembles master, applies minimal rules/patches; aim to migrate those rules upstream
      into XML generation.

- Observability
    - Structured logs per stage; summaries per mission; explicit lists of skipped/failed files; diffs for master
      `datasets.xml`.

- Failure Policy
    - Steps are individually fault‑tolerant (continue over per‑file errors), with final aggregation and non‑zero exit on
      fatal conditions.

This diagram and notes are meant as the definitive behavioural contract irrespective of internal libraries.
Re‑implementations should preserve sequence boundaries, side‑effects, and observable outputs to remain fully compatible
with existing operations and ERDDAP publication workflows.