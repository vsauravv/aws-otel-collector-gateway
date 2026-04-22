# Central OTEL Gateway - Deduplication Explained

## Overview

We are pulling **CloudWatch metrics** through Yet Another CloudWatch Exporter (YACE) and sending it to AMP/New Relic via ADOT collector

**Goal**: High Availability (HA) + Minimal Duplication

---

## Flow

1. CloudWatch emits metrics
2. 2 Pods of YACE pull metrics from CloudWatch --> Duplication possible
3. ADOT (2 replicas) scrapes metrics from YACE Pods. Every ADOT pod has replica label added with the metric name.
4. Metrics flow into both AMP and New Relic
5. Duplications are managed automatically at AMP - replica label + timestamp basis

# IMPORTANT:
1. YACE has two replicas for HA, in case one pod gets down, other will still serve the ttaffic
2. AMP manages any kind of duplication automatically.

## Architecture

```mermaid
flowchart TD
    %% CloudWatch
    CW[CloudWatch Metrics<br/>EC2, S3, RDS, ALB, NLB...]

    %% YACE Layer
    subgraph YACE_Layer ["YACE Exporter (2 Replicas)"]
        Y1[YACE Pod 1]
        Y2[YACE Pod 2]
    end

    %% ADOT Layer
    subgraph ADOT_Layer ["ADOT Collector (2 Replicas)"]
        A1[ADOT Pod 1<br/>+ replica label]
        A2[ADOT Pod 2<br/>+ replica label]
    end

    %% Destination
    subgraph Destination ["Destinations"]
        AMP[Amazon Managed Prometheus<br/>AMP<br/>Deduplication happens here]
        NR[New Relic]
    end

    %% Flow
    CW -->|Pull Metrics| Y1
    CW -->|Pull Metrics| Y2

    Y1 -->|Expose /metrics| A1
    Y1 -->|Expose /metrics| A2
    Y2 -->|Expose /metrics| A1
    Y2 -->|Expose /metrics| A2

    A1 -->|Remote Write| AMP
    A2 -->|Remote Write| AMP
    A1 -->|Remote Write| NR
    A2 -->|Remote Write| NR

    %% Styling
    classDef yace fill:#90EE90,stroke:#2E8B57,stroke-width:2px
    classDef adot fill:#FFB6C1,stroke:#333,stroke-width:2px
    classDef dest fill:#ADD8E6,stroke:#333,stroke-width:2px

    class Y1,Y2 yace
    class A1,A2 adot
    class AMP,NR dest
```