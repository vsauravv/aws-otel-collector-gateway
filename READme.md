```mermaid
flowchart TD
    subgraph AWS["AWS Cloud (ap-south-1)"]
        CW[CloudWatch Metrics<br/>(EC2, RDS, S3, EKS, etc.)]
    end

    subgraph EKS["EKS Cluster: vivek-aws-otel-test<br/>Namespace: monitoring"]
        subgraph YACE_Group["YACE (Yet Another CloudWatch Exporter)"]
            YACE[YACE Deployment<br/>quay.io/.../yace:v0.65.0<br/>1 replica<br/>ServiceAccount: yace-sa]
            YACE_SVC[YACE Service<br/>port: 5000<br/>Prometheus endpoint]
            YACE_CM[YACE ConfigMap<br/>discovery jobs for multiple AWS services]
        end

        subgraph SCRAPER_Group["ADOT Scraper"]
            SCRAPER[ADOT Scraper Deployment<br/>aws-otel-collector:latest<br/>1 replica<br/>ServiceAccount: adot-sa]
            SCRAPER_CM[ADOT Scraper ConfigMap<br/>Prometheus receiver + resource processor]
        end

        subgraph FORWARDER_Group["ADOT Forwarder (HA)"]
            FORWARDER[ADOT Forwarder Deployment<br/>aws-otel-collector:latest<br/>2 replicas<br/>ServiceAccount: adot-sa]
            FORWARDER_SVC[ADOT Forwarder Service<br/>port: 4317<br/>OTLP gRPC]
            FORWARDER_CM[ADOT Forwarder ConfigMap<br/>OTLP receiver + batch processor]
        end
    end

    NR[New Relic<br/>OTLP/HTTP endpoint<br/>https://otlp.nr-data.net:443]

    %% Data & Permission Flow
    CW --"Pulls metrics via AWS APIs<br/>(IRSA = YACE-IRSA-Role)"--> YACE
    YACE --> YACE_SVC
    YACE_SVC --"Prometheus scrape<br/>every 60s"--> SCRAPER
    SCRAPER --"OTLP gRPC (insecure)<br/>to adot-forwarder:4317"--> FORWARDER_SVC
    FORWARDER_SVC --> FORWARDER
    FORWARDER --"OTLP/HTTP + API key"--> NR

    %% IRSA for both ADOT components
    SCRAPER -.->|"IRSA = ADOT-IRSA-Role"| CW
    FORWARDER -.->|"IRSA = ADOT-IRSA-Role"| CW

    classDef aws fill:#FF9900,stroke:#232F3E,color:white;
    classDef component fill:#E6F3FF,stroke:#0066CC,color:#003366;
    classDef nr fill:#00CC66,stroke:#006600,color:white;
    class CW aws;
    class YACE,YACE_SVC,YACE_CM,SCRAPER,SCRAPER_CM,FORWARDER,FORWARDER_SVC,FORWARDER_CM component;
    class NR nr;
```