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
    subgraph AWS["AWS Cloud (ap-south-1)"]
        CW[CloudWatch Metrics\n(EC2, RDS, S3, EKS etc.)]
    end

    subgraph EKS["EKS Cluster - vivek-aws-otel-test\nNamespace: monitoring"]
        
        subgraph YACE["1. YACE - CloudWatch Exporter"]
            YACE_DEP[YACE Deployment\n1 replica\nyace-sa]
            YACE_SVC[YACE Service :5000]
            YACE_CM[YACE ConfigMap]
        end

        subgraph SCRAPER["2. ADOT Scraper"]
            SCRAPER_DEP[ADOT Scraper\n1 replica\nadot-sa]
            SCRAPER_CM[Scraper ConfigMap]
        end

        subgraph FORWARDER["3. ADOT Forwarder (HA)"]
            FORWARDER_DEP[ADOT Forwarder\n2 replicas\nadot-sa]
            FORWARDER_SVC[Forwarder Service :4317]
            FORWARDER_CM[Forwarder ConfigMap]
        end
    end

    DT[Dynatrace\nOTLP/HTTP Endpoint]

    %% Data Flow
    CW --"Pull metrics via AWS APIs"--> YACE_DEP
    YACE_DEP --> YACE_SVC
    YACE_SVC --"Prometheus scrape (60s)"--> SCRAPER_DEP
    SCRAPER_DEP --"OTLP gRPC"--> FORWARDER_SVC
    FORWARDER_SVC --> FORWARDER_DEP
    FORWARDER_DEP --"OTLP/HTTP + API Key"--> DT

    classDef aws fill:#FF9900,stroke:#232F3E,color:white;
    classDef component fill:#E6F3FF,stroke:#0066CC,color:#003366;
    classDef dest fill:#00CC66,stroke:#006600,color:white;
    
    class CW aws;
    class YACE_DEP,YACE_SVC,YACE_CM,SCRAPER_DEP,SCRAPER_CM,FORWARDER_DEP,FORWARDER_SVC,FORWARDER_CM component;
    class DT dest;
```