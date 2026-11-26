# GDP1 architecture

GDP1's Slocum workflow starts at the Django `slocum_run` command. It uses PostgreSQL to track missions and files, while the data itself lives in configured directories. At the start of a run, the manager attempts to refresh mission records from Sensor Tracker. If that refresh fails, it logs the error and can still select missions already in PostgreSQL. Metadata generation makes a separate Sensor Tracker request when that step runs.

## Component boundaries

```mermaid
flowchart LR
    CLI["Slocum command"] --> MANAGER["Pipeline manager"]
    ST["Sensor Tracker"] -->|Mission refresh| MANAGER
    MANAGER <--> DB[(PostgreSQL records)]
    MANAGER --> RUNNER["Mission builder and runner"]
    RUNNER --> BEFORE["Preparation steps"]
    BEFORE --> CORE["Conversion steps"]
    CORE --> AFTER["After-process steps"]
    ST -->|Deployment hierarchy when needed| BEFORE
    RAW["Configured Slocum source directory"] --> BEFORE
    BEFORE <--> DB
    CORE <--> DB
    AFTER <--> DB
    BEFORE --> LOCAL["Local metadata and data files"]
    CORE --> LOCAL
    AFTER --> LOCAL
    AFTER -.->|Selected transfers| HOST["Configured ERDDAP host paths"]
```

The mission builder selects Slocum records and attaches a command plus three ordered process builders to each mission. The runner processes missions one at a time. Each process builder supplies step handlers; command options and existing records determine whether a handler performs work. For steps that save state, the step manager runs the step before calling its Django model manager to record the result.

## Per-mission execution

This sequence shows the order of the major interactions for a mission that reaches its eligible steps. An error can stop the remaining steps for that mission. Each optional block runs only when its command option, required inputs, and recorded state allow it. The [Processing lifecycle page](processing-lifecycle.md) shows the main decisions in more detail.

```mermaid
sequenceDiagram
    participant CLI as Slocum command
    participant MAN as Pipeline manager
    participant ST as Sensor Tracker
    participant DB as PostgreSQL
    participant RUN as Mission runner
    participant STEPS as Mission steps
    participant FILES as Local files
    participant HOST as ERDDAP host paths
    CLI->>MAN: Run with selected options
    MAN->>ST: Request deployment list
    opt Refresh succeeds
        ST-->>MAN: Deployment records
        MAN->>DB: Update or create mission records
    end
    MAN->>DB: Select Slocum missions
    DB-->>MAN: Mission candidates
    MAN->>RUN: Run selected missions
    loop Each mission until runner stops
        RUN->>DB: Get or create mission-mode record
        RUN->>STEPS: Run preparation handlers
        STEPS->>FILES: Prepare local work directories
        STEPS->>DB: Record directory paths
        opt Metadata generation runs
            STEPS->>ST: Request deployment hierarchy
            ST-->>STEPS: Hierarchy metadata
            STEPS->>FILES: Write JSON metadata
            STEPS->>DB: Record metadata paths
        end
        opt Binary discovery runs
            STEPS->>FILES: Scan configured source directory
            STEPS->>DB: Record previously unseen binary paths
        end
        RUN->>STEPS: Run conversion handlers
        opt Binary conversion has eligible files
            STEPS->>FILES: Convert binary files to ASCII
            STEPS->>DB: Update binary and record ASCII paths
        end
        opt ASCII conversion has eligible files
            STEPS->>FILES: Write profile NetCDF from ASCII and metadata
            STEPS->>DB: Update ASCII and record NetCDF paths
        end
        RUN->>STEPS: Run after-process handlers
        opt XML generation is eligible
            STEPS->>FILES: Generate and refine XML fragment
            STEPS->>DB: Record XML fragment path
        end
        opt Selected files are transferred
            STEPS->>HOST: Copy XML and/or NetCDF files
            STEPS->>DB: Update recorded file flags
        end
        opt Cleanup is enabled
            STEPS->>FILES: Remove processed ASCII and NetCDF files
        end
    end
```

The file finder compares source paths with paths already recorded for that mission and mode. It does not compare file contents or hashes when deciding whether a path is new. Binary conversion selects existing, unprocessed source files; ASCII conversion can also pick up unprocessed `.dat` files added directly to the mission's ASCII directory. The ASCII step uses JSON metadata to write individual profile NetCDF files.

When requested, the after-process stage can draft an ERDDAP XML fragment from an available NetCDF file and refine it. Separate steps can copy the fragment and NetCDF files to configured host paths. A NetCDF transfer also writes a reload flag; it does not check whether ERDDAP later loads or serves the dataset.

## Stored state and limits

These four Django models form GDP1's processing record in PostgreSQL:

| Model | Role |
| --- | --- |
| `Mission` | Local deployment details refreshed from Sensor Tracker, including number, platform, and start/end times. |
| `MissionProcess` | One record per mission and processing mode; parent of directory and file records. |
| `ProcessDirectory` | The purpose and path of a directory associated with a mission process. |
| `ProcessFiles` | A file type and path associated with a mission process, plus processing and failure flags. |

The `ProcessFiles.processed` flag has a stage-specific meaning. For binary and ASCII inputs, conversion steps update it; for NetCDF and XML files, transfer steps update it. It is not a scientific quality result or a universal mission-complete flag. The `MissionProcess.status` field is present, but the Slocum workflow does not maintain it as an end-to-end run status.

The database stores paths and flags, not copies or fingerprints of the files. A record alone does not prove a file still exists or has the same contents. Existing JSON metadata may be reused, and a changed source file at an already recorded path is not rediscovered simply because its contents changed. Reprocessing can clear selected tracking records, but it is not a complete reset of every related record.

The runner catches errors from one mission and can continue to another; it later logs those errors without reliably returning a nonzero command exit. Individual conversion failures may also be logged or skipped. To assess a run, inspect its logs, recorded paths, actual output files, and the live ERDDAP service where publication matters.

## Where to look in the code

| Source | Responsibility |
| --- | --- |
| `gdp/management/commands/slocum_run.py` and `gdp/core/pipeline/manager.py` | Command options, Sensor Tracker mission refresh, and pipeline startup. |
| `gdp/core/mission/` and `gdp/core/pipeline/runners.py` | Mission selection, per-mission builders, execution, and runner errors. |
| `gdp/core/process/`, `gdp/contrib/step_handlers/`, and `gdp/core/steps/manager.py` | Preparation, conversion, after-process order, and step dispatch. |
| `gdp/contrib/step_implementation/file_finder/`, `gdp/contrib/step_implementation/slocum_processor_handler/`, and `gdp/engine/slocum/` | Source discovery and Slocum conversion. |
| `gdp/contrib/step_implementation/meta/`, `gdp/contrib/step_implementation/errdap_dataset_config/`, and `gdp/contrib/step_implementation/errdap_data_file_upload/` | JSON metadata, XML creation, and optional transfers. |
| `gdp/models.py` | Mission, directory, and file-tracking models. |
