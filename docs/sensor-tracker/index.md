# Sensor Tracker

Sensor Tracker provides the deployment and instrument information used by the Glider Data Pipeline. Both pipeline
versions read its data, but they use it in different ways.

## Information used by GDP

- **Deployments:** platform identity, deployment number, start and end times, and mission status. These help the
  pipelines select a mission and distinguish ongoing from completed deployments.
- **Instruments, data loggers, and parameters:** the inventory and variable details used to describe observations in
  metadata and output files.