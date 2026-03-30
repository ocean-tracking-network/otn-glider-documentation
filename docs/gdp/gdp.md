---
title: Glider Data Pipeline
---

# Glider Data Pipeline

This section outlines the high-level stages of the OTN Glider Data Pipeline.

## Overview

```mermaid
sequenceDiagram
  participant G as Glider
  participant S as Shore Station
  participant P as Processing
  participant R as Repository
  G->>S: Transmit data
  S->>P: Ingest
  P->>P: QC & Transform
  P->>R: Publish artifacts
```

## Stages

1. Ingest: acquire raw data and metadata
2. Process: quality control, calibration, and transformation
3. Publish: expose datasets for consumers and downstream systems
