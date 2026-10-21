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

1. Ingestion: The PCAP importer or traffic simulator supplies input data.
2. Extraction: The flow extractor converts packet data into normalized flows.
3. Feature Computation: The feature extractor calculates features and publishes events to Kafka.
4. Stream Processing: The stream processor analyzes events in time windows.
5. Detection: The detection engine identifies anomalies using rules and a second analytical method.
6. Classification: The alert and classification service assigns severity, confidence, and explanatory evidence.
7. Persistence: PostgreSQL persists flows, detections, configuration, and history.
8. Visualization: The Flask REST API exposes the data to the React dashboard.


---

# Component Diagrams

One diagram per component of the main diagram. Rounded nodes `([ ])` are neighbouring components, cylinders `[( )]` are data stores, and rectangles are subcomponents.

## A. PCAP Importer (C++)

Main Purpose: Ingests prepared static PCAP or PCAPNG network traffic files and validates input data before stream processing.

Key Sub-components:
	CLI / File Loader & Packet Reader: Supplies the initial interface to load capture files from disk and read individual packets.
	Format Detector & Input Validator: Detects file encodings and validates incoming packet structures to ensure data integrity.
	Rejection Logger: Captures and logs malformed or invalid input files safely.
	Replay Rate Controller & Import Job Tracker: Controls the replay speed of packets and tracks active import jobs and processing counts.

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

Main Purpose: Generates safe, controlled synthetic normal traffic and predefined anomalous scenarios without interacting with live production networks.
 
Key Sub-components:
	Scenario & Load Controllers: Manages YAML-based configuration settings and dynamic load parameters for testing.
	Traffic Generators: Implements specialized generators for normal traffic, Port Scans, DoS-like floods, Brute Force-like behavior, and Abnormal Patterns.
	Packet Builder & Output Writer: Constructs synthetic packets and writes generated flows downstream.
	Label Writer: Outputs ground-truth labels tied to scenario phase changes for evaluation metrics.

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

Main Purpose: Normalizes raw packet streams coming from importers or simulators into a uniform network flow model.

Key Sub-components:
	Packet Source & Decoder: Decodes incoming packet payloads and logs malformed errors.
	Flow Key Builder & Flow Table: Creates unique flow keys and maintains an active table of ongoing connections.
	Flow Updater & Expiry Manager: Updates active flow states and evicts finished or timed-out flows.
	Flow Normalizer: Standardizes attributes (timestamps, durations, endpoints, counts, flags) before sending them to feature extraction.

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

Main Purpose: Computes high-level temporal, volume, and connection-related metrics from normalized flows.

Key Sub-components:
	Flow Input Queue & Per-Flow Feature Calculator: Manages incoming flow streams and calculates metrics such as packets/sec, bytes/sec, and unique ports.
	Event Serializer & Kafka Producer: Serializes calculated feature events and publishes them directly to Apache Kafka.
	Delivery Error Handler: Handles failed network sends, managing automatic retries and dead-letter routing.

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

Main Purpose: Acts as the mandatory centralized event transport and message broker ensuring loose coupling across distributed platform components.

Key Sub-components:
	Primary Feature Topic (flows.features): Distributes serialized feature events keyed by source IP.
	Dead-Letter Topic: Captures failed or invalid events for diagnostic review.
	Consumer Groups: Manages offset commits and reliable stream consumption for downstream processing.

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

Main Purpose: Continuously consumes and processes event streams across multiple time windows.

Key Sub-components:
	Consumer, Deserializer & Schema Validator: Ingests Kafka messages, validates structures, and routes invalid data to the dead-letter queue.
	De-duplicator & Key Router: Cleans duplicate flows and routes keys appropriately.
	Window Manager & Aggregator: Aggregates streaming data across mandatory 1 s, 10 s, and 60 s windows before emitting results to detection.

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

Main Purpose: Identifies network anomalies and attacks using dual independent analytical methodologies.

Key Sub-components:
	Rule Engine: Deterministically detects Port Scans, DoS-like floods, Brute Force attempts, and Abnormal Connection Patterns using configurable parameters.
	Statistical / ML Detector: Implements secondary methods (e.g., Z-scores, moving statistics, or Isolation Forest) to catch anomalies.
	Config Loader, Baseline Store & Result Combiner: Manages system configuration, references historical baselines, and combines rule and statistical outputs into structured detections.

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

Main Purpose: Validates, classifies, and enriches raw anomalies with severity ratings, confidence levels, and human-readable explanations.

Key Sub-components:
	Validator & Correlator/De-duplicator: Validates incoming alerts and groups duplicate or related events.
	Classifier & Severity Mapper: Assigns standardized categories and maps severity levels (LOW/MEDIUM/HIGH/CRITICAL).
	Confidence Calculator & Explanation Builder: Computes confidence scores ($0-1$) and builds textual explanations containing key feature evidence.
	Persistence Writer: Securely writes classified alerts into the PostgreSQL database.

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

Main Purpose: Serves as the relational persistence layer for alerts and the detection information behind them, written by the Alert & Classification Service and read by the REST API.

Key Sub-components:
	Alerts Table: Stores each operator-facing alert, which groups one or more related detections (same source and type in consecutive windows).
	Detections Table: Stores individual detections with their ID, timestamp, type, severity, confidence, source, destination, time window, detection method, and textual explanation.
	Detection Evidence Table: Stores the key feature values behind each detection, together with the threshold or baseline they were compared against.

```mermaid
flowchart LR
    T1["ALERTS"] --> T2["DETECTIONS"]
    T2 --> T3["DETECTION_EVIDENCE"]
```

## J. REST API (Flask)

Main Purpose: Exposes platform data, history, statistics, and administrative endpoints securely to the frontend dashboard.

Key Sub-components:
	Security Modules: Rate limiters, JWT authentication modules, and role guards enforcing user and admin separation.
	Input Validators & Resource Blueprints: Validates external payloads and exposes blueprints for /auth, /traffic, /flows, /detections, /history, /stats, /config, and /health.
	Data Access Layer & Audit Logger: Interfaces with PostgreSQL and logs important administrative actions securely.

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

Main Purpose: Provides security operators and analysts with a responsive web-based visual interface for monitoring traffic, alerts, and historical data.

Key Sub-components:
	Authentication & Client Management: Login pages, token stores, and centralized API clients communicating with backend services.
	View Router & Monitoring Panels: Routers directing views to Live Monitoring, Alert Feeds, History/Filter Panels, and Statistics.
	Investigation Tools: Detection Details, Trace Views, and restricted Admin Configuration pages.

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
