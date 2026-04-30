---
title: Overall Class Diagram
---

### Class/Object Diagram — Key Relationships

Below is a `classDiagram` capturing the core classes, inheritance, and the most important associations across
the Glider Data Pipeline (GDP). It focuses on orchestration (manager → builders → runners), mission assembly, process
builders, and external integrations (Sensor Tracker → Mission model, ERDDAP steps).

```mermaid
classDiagram
%% =======================
%% High-level Orchestration
%% =======================
    class GliderPipelineManager {
        - glider_type: str
        - options: dict
        + run(): void
        + get_missions(): list[BaseMission]
        + get_mission_runner(missions): MissionRunner
        + update_from_sensor_tracker(): void
    }

    class MissionRunnerFactory {
        - missions: list[BaseMission]
        + build(): MissionRunner
    }

    class MissionRunner {
        - missions: list[BaseMission]
        + run(): void
        + raise_error(): void
        + should_break(): bool
        + mission_summary(): str
    }

    class AutoDelayedModeMissionProcessRunner {
        + should_break(): bool
    }

    class SlocumMissionRunner {
        + run(): void
    }

    class SlocumAutoDelayedModeMissionProcessRunner {
        + should_break(): bool
    }

    class WaveMissionRunner {
        + run(): void
    }

    GliderPipelineManager --> MissionRunnerFactory: uses
    MissionRunnerFactory --> MissionRunner: builds
    AutoDelayedModeMissionProcessRunner --|> MissionRunner
    SlocumMissionRunner --|> MissionRunner
    SlocumAutoDelayedModeMissionProcessRunner --|> SlocumMissionRunner
    WaveMissionRunner --|> MissionRunner
%% =======================
%% Missions & Builders
%% =======================
    class BaseMissionBuilder {
        <<abstract>>
        - options: dict
        - command: any
        - missions: list[BaseMission]
        + build(): list[BaseMission]
        + get_selected_mission_list(): QuerySet[Mission]
        + process_builder_list(): list[BaseProcessBuilder]
        + build_command(): Command
    }

    class SlocumMissionBuilder {
        + GLIDER_TYPE = "slocum"
    }

    class WaveMissionBuilder {
        + GLIDER_TYPE = "wave"
    }

    BaseMissionBuilder <|-- SlocumMissionBuilder
    BaseMissionBuilder <|-- WaveMissionBuilder

    class BaseMission {
        - mission_dict: dict
        - command: Command
        - builders: list[BaseProcessBuilder]
    }

    BaseMissionBuilder --> BaseMission: creates
    BaseMissionBuilder --> CommandFactory: uses
%% =======================
%% Process Builders & Steps
%% =======================
    class BaseProcessBuilder {
        <<abstract>>
        - mission_dict: dict
        - command: Command
        - step_manager: BaseStepManager
        + build(): void
        + add_step(handler): void
        + add_step_generator(gen): void
    }

    class BeforeSlocumProcessBuilder
    class SlocumProcessBuilder
    class AfterProcessBuilder

    BaseProcessBuilder <|-- BeforeSlocumProcessBuilder
    BaseProcessBuilder <|-- SlocumProcessBuilder
    BaseProcessBuilder <|-- AfterProcessBuilder
    BaseMission o-- BaseProcessBuilder: has many
    BaseProcessBuilder --> BaseStepManager: delegates to

    class BaseStepManager {
        + run(steps): void
    }

%% Representative contrib handlers (objects invoked by builders)
    class LocalDirectoriesCreationHandler
    class SlocumProcessorHandler
    class FileCleaningHandler
    class ErddapDatasetConfigHandler
    class ErddapDataUploadHandler
    class MetaGenerationHandler

    BeforeSlocumProcessBuilder --> LocalDirectoriesCreationHandler: adds step
    SlocumProcessBuilder --> SlocumProcessorHandler: adds step
    AfterProcessBuilder --> FileCleaningHandler: adds step
    AfterProcessBuilder --> ErddapDatasetConfigHandler: adds step
    AfterProcessBuilder --> ErddapDataUploadHandler: optional step
    AfterProcessBuilder --> MetaGenerationHandler: adds step
%% =======================
%% Commands, Components & Models
%% =======================
    class CommandFactory {
        + generate(options, glider_type): Command
    }

    class Command {
<<interface/placeholder>>
+ options: dict
}

class Mission {
<<DjangoModel>>
+ deployment_number: int
+ platform_type: str
+ start_time: datetime
+ end_time: datetime|None
+ update_or_create_from_dict(...)
}

class SensorTrackerProvider as STP {
+ get_glider_deployment(): dict
}

GliderPipelineManager --> STP: updates DB from
STP --> Mission: feeds deployments
BaseMissionBuilder --> Mission: selects from DB
CommandFactory --> Command: creates
BaseMission --> Command: has

%% =======================
%% Management Commands (entrypoint)
%% =======================
class SlocumRunCommand {
+ add_arguments(parser): void
+ handle(*args, **options): void
}

SlocumRunCommand --> GliderPipelineManager: invokes
```

#### Notes

- Class names and relationships reflect the modules observed in `gdp/core` and `gdp/contrib` (e.g.,
  `GliderPipelineManager`, mission builders, runners, process builders, step managers, and representative contrib
  handlers).
- Contrib handlers (e.g., `SlocumProcessorHandler`, `ErddapDatasetConfigHandler`) are shown as step objects added into
  builders via the step manager; the exact handler class names may vary slightly by file, but their roles and
  relationships are consistent.
- The `Mission` model and `SensorTrackerProvider` (via `gdp.component.stp`) illustrate how deployments are synchronized
  before selection.