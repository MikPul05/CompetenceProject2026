# NetGuard Architecture

```mermaid
flowchart TD
    A["PCAP Importer (C++)"]
    B["Traffic Simulator (Python)"]
    C["Flow Extractor (C++)"]
    D["Feature Extractor (C++)"]
    E[("Apache Kafka")]
    F["Stream Processor (Python)"]
    G["Detection Engine (Python)"]
    H["Alert & Classification Service (Python)"]
    I[("PostgreSQL")]
    J["REST API (Flask)"]
    K["Dashboard (React)"]

    A --> C
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I --> J
    J --> K
```

## Data flow

1. The PCAP importer or traffic simulator supplies input data.
2. The flow extractor converts packet data into normalized flows.
3. The feature extractor calculates features and publishes events to Kafka.
4. The stream processor analyzes events in time windows.
5. The detection engine identifies anomalies using rules and a second analytical method.
6. The alert and classification service assigns severity, confidence, and explanatory evidence.
7. PostgreSQL persists flows, detections, configuration, and history.
8. The Flask REST API exposes the data to the React dashboard.


---

# Component Diagrams

One diagram per component of the main diagram. Rounded nodes `([ ])` are neighbouring components, cylinders `[( )]` are data stores, and rectangles are subcomponents.

## A. PCAP Importer (C++)

```mermaid
flowchart LR
    X0(["PCAP / PCAPNG files"]) --> A1["CLI / File Loader"]
    A1 --> A2["Format Detector"]
    A2 --> A3{"Input Validator"}
    A3 -->|valid| A4["Packet Reader"]
    A3 -->|invalid| A6["Rejection Logger"]
    A4 --> A5["Replay Rate Controller"]
    A7["Import Job Tracker"] -->|import_job_id| A4
    A4 -->|counts| A7
    A5 --> X1(["Flow Extractor"])
```

## B. Traffic Simulator (Python)

```mermaid
flowchart LR
    B1["CLI + Scenario Config (YAML)"] --> B2["Scenario Controller"]
    B2 --> B3["Load Controller"]
    B3 --> B4["Traffic Generators<br/>Normal · Port Scan · DoS-like ·<br/>Brute Force-like · Abnormal Pattern"]
    B4 --> B5["Packet Builder"]
    B5 --> B6["Output Writer"]
    B6 --> X1(["Flow Extractor"])
    B2 -->|phase change| B7["Label Writer"]
    B7 --> X2[("Labels file")]
```

## C. Flow Extractor (C++)

```mermaid
flowchart LR
    X0(["PCAP Importer"]) --> C1["Packet Source"]
    X1(["Traffic Simulator"]) --> C1
    C1 --> C2["Packet Decoder"]
    C2 -->|malformed| C8["Error Log"]
    C2 --> C3["Flow Key Builder"]
    C3 --> C4["Flow Table"]
    C4 <--> C5["Flow Updater"]
    C6["Expiry Manager"] -->|evict finished flows| C4
    C4 --> C7["Flow Normaliser"]
    C7 --> X2(["Feature Extractor"])
```

## D. Feature Extractor (C++)

```mermaid
flowchart LR
    X0(["Flow Extractor"]) --> D1["Flow Input Queue"]
    D1 --> D2["Per-Flow Feature Calculator"]
    D2 --> D3["Event Serializer"]
    D3 --> D4["Kafka Producer"]
    D4 -->|flows.features| X1[("Apache Kafka")]
    D4 -->|failed send| D5["Delivery Error Handler"]
    D5 -->|retry| D4
    D5 -->|dead-letter| X1
```

## E. Apache Kafka

```mermaid
flowchart LR
    X0(["Feature Extractor"])
    subgraph K["Apache Kafka (single broker)"]
        T1["Topic: flows.features<br/>key: source IP"]
        T2["Topic: dead-letter"]
        G1["Consumer group:<br/>stream-processor"]
    end
    X1(["Stream Processor"])
    X0 -->|publish| T1
    X0 -->|failed events| T2
    T1 --> G1
    G1 --> X1
    X1 -->|invalid events| T2
```

## F. Stream Processor (Python)

```mermaid
flowchart LR
    X0[("Kafka: flows.features")] --> F1["Kafka Consumer"]
    F1 --> F2{"Deserializer +<br/>Schema Validator"}
    F2 -->|invalid| X2[("Kafka: dead-letter")]
    F2 -->|valid| F3["De-duplicator"]
    F3 --> F4["Key Router"]
    F4 --> F5["Window Manager<br/>1 s · 10 s · 60 s"]
    F5 --> F6["Window Aggregator"]
    F6 --> F7["Window Emitter"]
    F7 --> X1(["Detection Engine"])
    F7 -.->|commit offsets| F1
```

## G. Detection Engine (Python)

```mermaid
flowchart LR
    X0(["Stream Processor"]) --> G1["Window Input"]
    G1 --> G2["Rule Engine<br/>Port Scan · DoS-like ·<br/>Brute Force-like · Abnormal Pattern"]
    G1 --> G3["Statistical / ML Detector"]
    G4["Config Loader"] --> G2
    G4 --> G3
    G5["Baseline Store"] --> G2
    G5 --> G3
    G2 --> G6["Result Combiner"]
    G3 --> G6
    G6 --> G7["Detection Builder"]
    G7 --> X1(["Alert & Classification Service"])
```

## H. Alert & Classification Service (Python)

```mermaid
flowchart TD
    X0(["Detection Engine"]) --> H1["Detection Input"]
    H1 --> H2["Validator"]
    H2 --> H3["Correlator / De-duplicator"]
    H3 --> H4["Classifier"]
    H4 --> H5["Severity Mapper"]
    H5 --> H6["Confidence Calculator"]
    H6 --> H7["Explanation Builder"]
    H7 --> H8["Persistence Writer"]
    H8 --> X1[("PostgreSQL")]
```

## I. PostgreSQL

```mermaid
erDiagram
    DATASETS ||--o{ IMPORT_JOBS : "imported as"
    IMPORT_JOBS ||--o{ FLOWS : produces
    SIMULATION_RUNS ||--o{ FLOWS : produces
    DATASETS ||--o{ GROUND_TRUTH_LABELS : "labelled by"
    SIMULATION_RUNS ||--o{ GROUND_TRUTH_LABELS : "labelled by"
    FLOWS }o--o{ DETECTIONS : "evidence for"
    DETECTIONS ||--o{ DETECTION_EVIDENCE : has
    ALERTS ||--o{ DETECTIONS : groups
    ROLES ||--o{ USERS : assigned
    USERS ||--o{ AUDIT_LOG : performs
    USERS ||--o{ DETECTION_CONFIG : updates
```

## J. REST API (Flask)

```mermaid
flowchart LR
    X0(["Dashboard"]) --> J1["Rate Limiter"]
    J1 --> J2["Auth Module (JWT)"]
    J2 --> J3["Role Guard"]
    J3 --> J4["Input Validator"]
    J4 --> J5["Resource Blueprints<br/>/auth · /traffic · /flows · /detections<br/>/history · /stats · /config · /health"]
    J5 --> J6["Data Access Layer"]
    J6 --> X1[("PostgreSQL")]
    J5 -->|JSON response| X0
    J5 -->|admin actions| J7["Audit Logger"]
    J7 --> X1
```

## K. Dashboard (React)

```mermaid
flowchart TD
    K1["Login Page"] --> K2["Token Store"]
    K2 --> K3["API Client"]
    K3 --> X0(["REST API"])
    K4["Router"] --> K5["Live Monitoring"]
    K4 --> K6["Alert Feed"]
    K4 --> K7["History + Filter Panel"]
    K4 --> K8["Statistics"]
    K4 --> K9["Admin Configuration<br/>(admin only)"]
    K6 --> K10["Detection Details"]
    K7 --> K10
    K10 --> K11["Trace View"]
    K5 & K6 & K7 & K8 & K9 & K10 & K11 -.-> K3
```
