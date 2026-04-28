# Central OTEL Gateway - Deduplication Explained

## Overview

We are pulling **CloudWatch metrics** through Yet Another CloudWatch Exporter (YACE) and sending it to Dynatrace via ADOT collector

**Goal**: High Availability (HA) + Minimal Duplication

---

## Flow

1. CloudWatch emits metrics
2. 1 pod of YACE pull metrics from CloudWatch --> For Duplication
3. ADOT (2 replicas) scrapes metrics from YACE Pods. Every ADOT pod has replica label added with the metric name.
4. Metrics flow into Dynatrace

### IMPORTANT:
1. YACE has one replicas for deduplication, in case this pod gets down, other will start and serve the ttaffic

## Architecture

```mermaid
flowchart TD
    %% --- AWS Layer ---
    subgraph AWS["AWS Cloud (ap-south-1)"]
        CW["CloudWatch Metrics<br/>(EC2 • RDS • S3 • EKS)"]
    end

    %% --- EKS Cluster ---
    subgraph EKS["EKS Cluster: vivek-aws-otel-test<br/>Namespace: monitoring"]

        %% YACE
        subgraph YACE["YACE (CloudWatch Exporter)"]
            YACE_DEP["Deployment<br/>1 replica"]
            YACE_SVC["Service :5000"]
            YACE_CM["Config"]
        end

        %% Scraper
        subgraph SCRAPER["ADOT Scraper"]
            SCRAPER_DEP["Deployment<br/>1 replica"]
            SCRAPER_CM["Config"]
        end

        %% Forwarder
        subgraph FORWARDER["ADOT Forwarder (HA)"]
            FORWARDER_DEP["Deployment<br/>2 replicas"]
            FORWARDER_SVC["Service :4317"]
            FORWARDER_CM["Config"]
        end
    end

    %% --- Destination ---
    DT["Dynatrace<br/>OTLP/HTTP Endpoint"]

    %% --- Flow ---
    CW -->|Pull via AWS APIs| YACE_DEP
    YACE_DEP --> YACE_SVC
    YACE_SVC -->|Prometheus scrape (60s)| SCRAPER_DEP
    SCRAPER_DEP -->|OTLP gRPC| FORWARDER_SVC
    FORWARDER_SVC --> FORWARDER_DEP
    FORWARDER_DEP -->|OTLP/HTTP + API Key| DT
```