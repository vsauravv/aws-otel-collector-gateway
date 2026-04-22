# Central OTEL Gateway - Deduplication Explained

## Overview

We are pulling **CloudWatch metrics** through Yet Another CloudWatch Exporter (YACE) and sending it to AMP/New Relic via ADOT collector

**Goal**: High Availability (HA) + Minimal Duplication

---

## Architecture

```mermaid
flowchart TD
    A[CloudWatch Metrics] 
    -->|Pull| 
    B[YACE Exporter<br/>Replicas = 2]

    B -->|Expose /metrics| 
    C[ADOT Collector<br/>Replicas = 2]

    C -->|Remote Write| D[Amazon Managed Prometheus<br/>(AMP)]
    C -->|Remote Write| E[New Relic]