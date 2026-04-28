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
        CW[CloudWatch Metrics\n(EC2, RDS, S3, EKS, ElastiCache etc.)]
    end

    subgraph EKS["EKS Cluster: vivek-aws-otel-test\nNamespace: monitoring"]
        
        subgraph YACE_Group["1. YACE - CloudWatch Exporter"]
            YACE[YACE Deployment\n1 replica\nServiceAccount: yace-sa\nIRSA Role: YACE-IRSA-Role]
            YACE_SVC[YACE Service\nport: 5000\nPrometheus /metrics]
            YACE_CM[YACE ConfigMap\nDiscovery jobs for multiple AWS services]
        end

        subgraph SCRAPER_Group["2. ADOT Scraper"]
            SCRAPER[ADOT Scraper Deployment\n1 replica\nServiceAccount: adot-sa\nIRSA Role: ADOT-IRSA-Role]
            SCRAPER_CM[ADOT Scraper ConfigMap\nPrometheus Receiver]
        end

        subgraph FORWARDER_Group["3. ADOT Forwarder (HA)"]
            FORWARDER[ADOT Forwarder Deployment\n2 replicas\nServiceAccount: adot-sa\nIRSA Role: ADOT-IRSA-Role]
            FORWARDER_SVC[ADOT Forwarder Service\nport: 4317\nOTLP gRPC]
            FORWARDER_CM[ADOT Forwarder ConfigMap\nOTLP Receiver → New Relic]
        end
    end

    NR[New Relic\nOTLP/HTTP Endpoint]

    %% Data Flow
    CW --"Pull metrics using AWS APIs"--> YACE
    YACE --> YACE_SVC
    YACE_SVC --"Prometheus scrape (60s)"--> SCRAPER
    SCRAPER --"OTLP gRPC"--> FORWARDER_SVC
    FORWARDER_SVC --> FORWARDER
    FORWARDER --"OTLP/HTTP + API Key"--> NR

    classDef aws fill:#FF9900,stroke:#232F3E,color:white;
    classDef component fill:#E6F3FF,stroke:#0066CC,color:#003366;
    classDef nr fill:#00CC66,stroke:#006600,color:white;
    
    class CW aws;
    class YACE,YACE_SVC,YACE_CM,SCRAPER,SCRAPER_CM,FORWARDER,FORWARDER_SVC,FORWARDER_CM component;
    class NR nr;
```