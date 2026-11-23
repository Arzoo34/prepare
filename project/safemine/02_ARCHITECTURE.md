# SafeMine: Architecture & System Design

---

## 1. Architectural Pattern & Paradigm

SafeMine is structured as an **Event-Driven, Layered Cyber-Physical Monolith with Reactive Push Streaming**.

```
+----------------------------------------------------------------------------------------------------+
|                                    FIVE-TIER ARCHITECTURAL STACK                                   |
+----------------------------------------------------------------------------------------------------+
| Tier 1: Hardware & Sensing Layer        ESP32 C++ Firmware, FreeRTOS, I2C/ADC Bus, Wearable Sensor |
| Tier 2: Ingestion & Gateway Layer       Express HTTP Endpoint, Embedded Aedes MQTT Broker          |
| Tier 3: Business Logic & Optimizer      Sensor Fusion, Oxygen Depletion Model, Dijkstra Pathfinding|
| Tier 4: Persistence & Storage Layer     sql.js WebAssembly SQLite Engine, Sync Disk Exporter       |
| Tier 5: Presentation & Client Layer     HTML5 Canvas 2D Engine, Leaflet GPS, WebSockets Client     |
+----------------------------------------------------------------------------------------------------+
```

### Architectural Classification
1. **Event-Driven**: Sensor readings, MQTT topic publications, and threshold infractions emit events that trigger asynchronous callbacks, state updates, and real-time WebSocket broadcasts.
2. **Modular Monolith**: All server-side components (MQTT broker, REST API server, WebSocket hub, database interface, and analytical optimization engine) execute concurrently within a single Node.js runtime process.
3. **Pipeline-Based Data Processing**: Telemetry undergoes a strict linear pipeline:
   $$\text{Ingestion} \longrightarrow \text{Normalization} \longrightarrow \text{Validation/Clamping} \longrightarrow \text{State Mutation} \longrightarrow \text{Sensor Fusion} \longrightarrow \text{Escalation / Downlink} \longrightarrow \text{Client Broadcast}$$
4. **Client-Server with Full-Duplex Push**: While setup and administrative configurations use standard REST request-response cycles, real-time telemetry uses continuous WebSocket push streaming, eliminating client polling.

---

## 2. Layer-by-Layer Architectural Breakdown

### Tier 1: Hardware & Sensing Layer (`ESP32.ino`)
- **Responsibilities**:
  - Interfacing with physical hardware pins: GPIO 34 (MQ-2 analog ADC), GPIO 4 (DHT11 1-wire), I2C pins GPIO 21/22 (Wire bus for MPU-6050 and BMP085/180), and GPIO 25 (piezoelectric buzzer).
  - Executing local edge processing: computing 3D motion differential ($\Delta = |\Delta x| + |\Delta y| + |\Delta z|$), calculating shock magnitude ($\sqrt{x^2+y^2+z^2} > 3.0\text{g}$), and evaluating fall impact windows ($5000\text{ms}$).
  - Executing local oxygen depletion calculations mirroring the backend algorithm.
  - Packaging readings into a JSON payload and dispatching HTTP POST requests every $3000\text{ms}$.

### Tier 2: Ingestion & Gateway Layer (`server.js`, `lora-gateway.js`)
- **Responsibilities**:
  - Providing multi-protocol ingestion:
    - **HTTP REST**: `POST /api/telemetry` for Wi-Fi-enabled nodes and `POST /api/wristband` for wearable distress bands.
    - **MQTT Broker**: Embedded Aedes server listening on port 1883, handling subscriptions for topics `mineguard/helmet/#`, `mineguard/wristband/#`, and `mineguard/anchor/#`.
  - Rate limiting incoming HTTP requests via `express-rate-limit` ($300\text{ req/minute}$).
  - Managing downlink feedback: Publishing commands back to physical nodes on `mineguard/helmet/:id/downlink`.

### Tier 3: Business Logic & Processing Pipeline
- **Responsibilities**:
  - **Dynamic Sensor Validation & Clamping**: Mitigating sensor glitches by rejecting impossible values (e.g., $\text{CH}_4 > 5\%$, $\text{Temp} > 80^\circ\text{C}$).
  - **Hardware Exemption Registry**: Maintaining the `realHardwareWorkers` Map to ensure real hardware data is not overwritten by internal simulation routines.
  - **Sensor Health Monitor**: Detecting flatlined/stuck sensors ($0$ or max streaks) and telemetry dropouts ($10\text{s}$ warning, $30\text{s}$ critical).
  - **Sensor Fusion Engine**: Evaluating cross-sensor patterns (Gas Accumulation, Fire Signatures, Incapacitated Miners).
  - **Multi-Stage Escalation Engine**: Managing timers for automatic state transitions from Warning ($10\text{s}$) to Critical ($20\text{s}$) to Panic.
  - **Optimization Suite (`optimizer.js`)**: Executing Dijkstra shortest-path evacuation routing, proportional ventilation fan control, worker redistribution, and predictive fatigue modeling.

### Tier 4: Persistence & Storage Layer (`db.js`, `mineguard.db`)
- **Responsibilities**:
  - Providing a synchronous, SQLite-compatible interface wrapped around WebAssembly `sql.js`.
  - Maintaining relational schemas: `workers`, `telemetry`, `alerts`, `devices`, `sensor_faults`, `anchors`, `incidents`, `evacuations`, `geofences`, `refuges`, `drones`, and `settings`.
  - Exporting the in-memory SQLite binary array to `mineguard.db` on disk upon every mutating SQL operation.

### Tier 5: Presentation & Client Layer (`index.html`, `app.js`, `admin-dashboard.js`)
- **Responsibilities**:
  - Maintaining active full-duplex WebSocket communication with automatic reconnection.
  - Rendering subterranean spatial positions on an HTML5 Canvas 2D engine with animated miner vectors, anchor signal circles, depth gradients, and pulsing hazard zones.
  - Rendering surface GPS telemetry using Leaflet.js map tiles.
  - Visualizing real-time telemetry time-series using Chart.js.
  - Generating synthesized audio tones using the Web Audio API for emergency notifications.
  - Interfacing with the Groq AI Copilot via `/api/chat`.

---

## 3. High-Level System Architecture Diagram

```mermaid
flowchart TB
    subgraph Subterranean["Underground Mine Environment"]
        H1["Smart Helmet (ESP32)<br/>DHT11, MQ2, MPU6050, BMP180"]
        W1["Smart Wristband<br/>Movement, SOS, Battery"]
        A1["Anchor Infrastructure (x10)<br/>LoRa/BLE RSSI Beacons"]
    end

    subgraph Ingestion["Ingestion & Gateway Tier"]
        HTTP_IN["Express HTTP Gateway<br/>POST /api/telemetry<br/>POST /api/wristband"]
        MQTT_IN["Embedded Aedes MQTT Broker<br/>Port 1883<br/>Topics: mineguard/#"]
        LORA_BR["LoRa Gateway Bridge<br/>lora-gateway.js"]
    end

    subgraph CoreEngine["SafeMine Node.js Core Engine (server.js)"]
        HW_REG["Real Hardware Registry<br/>realHardwareWorkers Map"]
        CLAMP["Input Sanitization &<br/>Sensor Range Clamping"]
        HEALTH["Sensor Health Monitor<br/>Stuck / Drift / Timeout"]
        O2_MOD["O2 Respiration Model<br/>o2Calculation.js"]
        FUSION["Sensor Fusion Engine<br/>Fire, Gas, Incapacitation"]
        ESCAL["Alert Escalation Engine<br/>Warning -> Critical -> Panic"]
        OPTIM["Safety Optimizer<br/>Dijkstra, Vent, Fatigue"]
    end

    subgraph DataStorage["Persistence Tier (db.js)"]
        WASM_DB["sql.js WebAssembly Database<br/>(In-Memory Virtual SQLite)"]
        DISK_FILE[("mineguard.db<br/>(Synchronous Disk File)")]
    end

    subgraph ExternalServices["External Cloud & Telephony Services"]
        TWILIO["Twilio Telephony API<br/>Automated Voice Call & SMS"]
        GROQ["Groq AI API<br/>Llama 3.3 70B Versatile"]
    end

    subgraph Presentation["Surface Command Center"]
        WS_HUB["WebSocket Broadcast Server<br/>ws://localhost:3001"]
        SPA["Browser Dashboard (app.js)<br/>Single Page Application"]
        CANVAS["Subterranean Canvas 2D HUD<br/>Tunnel Topology & Miners"]
        LEAFLET["Surface Leaflet GPS Map<br/>Georeferenced Marker Overlay"]
        AUDIO["Web Audio Synthesizer<br/>Emergency Audio Alarms"]
    end

    H1 -->|HTTP POST| HTTP_IN
    H1 -.->|MQTT Pub| MQTT_IN
    W1 -->|HTTP POST| HTTP_IN
    A1 -->|MQTT Pub| LORA_BR --> MQTT_IN

    HTTP_IN --> HW_REG --> CLAMP
    MQTT_IN --> HW_REG --> CLAMP

    CLAMP --> HEALTH
    CLAMP --> O2_MOD
    O2_MOD --> FUSION
    HEALTH --> FUSION
    FUSION --> ESCAL
    FUSION --> OPTIM

    CoreEngine <-->|execQuery / execRun| WASM_DB
    WASM_DB -->|fs.writeFileSync| DISK_FILE

    ESCAL -->|Trigger Emergency Alert| TWILIO
    CoreEngine <-->|Context-Injected Query| GROQ

    CoreEngine -->|Event Push| WS_HUB
    WS_HUB -->|Live JSON Packets| SPA

    SPA --> CANVAS
    SPA --> LEAFLET
    SPA --> AUDIO
```

### Node & Arrow Explanation
1. **`H1` $\to$ `HTTP_IN` & `MQTT_IN`**: The physical ESP32 helmet pushes JSON telemetry over either direct Wi-Fi HTTP or MQTT.
2. **`W1` $\to$ `HTTP_IN`**: The fallback wristband submits vital signs and emergency SOS state to `/api/wristband`.
3. **`HW_REG` (Real Hardware Registry)**: Stamps the hardware ID in a volatile memory map to exempt it from simulator overwrites.
4. **`CLAMP` (Input Sanitization)**: Validates incoming readings against physiological and physical limits, preventing sensor glitches from triggering false safety alarms.
5. **`O2_MOD` (Oxygen Calculation Engine)**: Calculates dynamic ambient oxygen based on worker count and gas displacement.
6. **`FUSION` $\to$ `ESCAL`**: When combined hazard criteria are met, the escalation state machine initiates a staged alert timer.
7. **`ESCAL` $\to$ `TWILIO`**: High-severity emergencies trigger automated SMS and voice phone calls with synthesized TwiML voice alerts.
8. **`WASM_DB` $\to$ `DISK_FILE`**: Mutating SQL operations trigger synchronous byte serialization from WebAssembly memory to `mineguard.db`.
9. **`WS_HUB` $\to$ `SPA`**: All processed telemetry, alerts, and device status updates are pushed immediately to all connected browsers.

---

## 4. Component Interaction Diagram

```mermaid
flowchart LR
    subgraph ClientLayer["Frontend Runtime"]
        UI["app.js / admin-dashboard.js"]
        API_JS["api.js (HTTP / WS Client)"]
    end

    subgraph ServerLayer["Backend Runtime"]
        EXPRESS["Express Router & Middleware"]
        WS_SRV["WebSocket Server (ws)"]
        AEDES["Aedes MQTT Broker"]
        OPT_ENG["optimizer.js Engine"]
        EMERG["emergency-contact.js"]
        DB_LAYER["db.js Wrapper"]
    end

    subgraph StorageLayer["Data Storage"]
        SQLJS["sql.js Virtual Machine"]
        FILE_SYSTEM["Host OS File System"]
    end

    UI <--> API_JS
    API_JS -->|REST API Calls| EXPRESS
    API_JS <-->|Real-Time WebSockets| WS_SRV

    EXPRESS --> DB_LAYER
    WS_SRV --> DB_LAYER
    AEDES --> EXPRESS

    EXPRESS --> OPT_ENG
    EXPRESS --> EMERG

    DB_LAYER <--> SQLJS
    SQLJS -->|export buffer| FILE_SYSTEM
```

---

## 5. End-to-End Data Flow Pipeline

```mermaid
sequenceDiagram
    autonumber
    participant HW as ESP32 Helmet
    participant SRV as SafeMine Server (server.js)
    participant O2 as O2 Physics Engine (o2Calculation.js)
    participant DB as SQLite (sql.js / WASM)
    participant OPT as Optimizer (optimizer.js)
    participant WS as WebSocket Clients (Browsers)
    participant TW as Twilio Voice/SMS API

    HW->>SRV: HTTP POST /api/telemetry (CH4, CO, Temp, Accelerometer)
    Note over SRV: 1. Record Hardware ID in realHardwareWorkers<br/>2. Validate & Clamp Readings
    SRV->>O2: calculateOxygenLevel(workerCount, ch4)
    O2-->>SRV: Dynamic O2 % (e.g. 19.2% - WARNING)
    SRV->>SRV: calcRisk(telemetry) -> Computes Risk Score (0-10)
    SRV->>DB: Upsert into workers & Insert into telemetry table
    Note over DB: Synchronously export WASM memory & write mineguard.db

    alt Hazard Detected (e.g., CH4 > 1.0% or Panic Pressed)
        SRV->>SRV: checkSafetyThresholds() & checkSensorFusion()
        SRV->>DB: INSERT INTO alerts (Critical / Emergency)
        SRV->>WS: Broadcast NEW_ALERT & WORKER_UPDATE
        SRV->>OPT: dijkstraEvacRoute(workerX, workerY)
        OPT-->>SRV: Optimal Evacuation Path avoiding hazards
        SRV->>WS: Broadcast ESCALATION_STAGE & Evacuation Route
        SRV->>TW: notifyAdmin() -> Outbound Voice Call (TwiML) & SMS
        SRV->>HW: Downlink MQTT: EVACUATE IMMEDIATELY (Beep buzzer)
    else Normal Operating Conditions
        SRV->>WS: Broadcast WORKER_UPDATE
    end
```

---

## 6. Evacuation Routing & Optimization Workflow

The optimization subsystem (`optimizer.js`) executes every $30\text{ seconds}$ or on demand:

```mermaid
flowchart TD
    START["Run Optimization Routine"] --> FETCH["Fetch Active Workers, Anchors, and Active Geofences"]
    
    FETCH --> RISK["Calculate Optimized Multi-Criteria Risk<br/>calcOptimizedRisk(w)"]
    RISK --> UPDATE_RISK["Persist Updated Risk Scores to DB workers table"]
    
    FETCH --> VENT["Optimize Zone Ventilation<br/>optimizeVentilation()"]
    VENT --> VENT_METRICS["Compute Gas Density & Allocate Dynamic Fan Speeds (20-100%)"]
    
    FETCH --> DIST["Evaluate Worker Distribution<br/>optimizeWorkerDistribution()"]
    DIST --> CHECK_CAP{"Zone Headcount > 5 OR<br/>Zone Risk > 6?"}
    CHECK_CAP -- Yes --> GEN_REDIST["Generate Relocation Orders to Safe Zones"]
    CHECK_CAP -- No --> BALANCED["Mark Distribution as Balanced"]
    
    FETCH --> FATIGUE["Predict Worker Fatigue<br/>predictFatigue()"]
    FATIGUE --> FAT_METRICS["Synthesize Shift Time (Battery Proxy), HR & Motion"]
    
    FETCH --> EVAC["Identify High-Risk Workers (Risk >= 5 or Emergency)"]
    EVAC --> DIJKSTRA["Run Dynamic Dijkstra Algorithm<br/>dijkstraEvacRoute()"]
    DIJKSTRA --> PENALTY["Inject Hazard Penalty (+500) & Congestion (+15/worker)"]
    PENALTY --> BEST_EXIT["Identify Nearest Safe Exit or Refuge Chamber"]
    
    VENT_METRICS --> BUNDLE["Compile Optimization Report & Safety Score (0-100%)"]
    GEN_REDIST --> BUNDLE
    FAT_METRICS --> BUNDLE
    BEST_EXIT --> BUNDLE
    
    BUNDLE --> BROADCAST["Broadcast OPTIMIZATION_UPDATE via WebSockets"]
    BROADCAST --> FINISH["Cache Report in lastOptimResult"]
```

---

## 7. Subterranean Evacuation Topology (Graph Representation)

The evacuation routing engine models the underground mine as an undirected, weighted graph:

```mermaid
graph LR
    SURFACE["SURFACE<br/>(Exit Portal)"]
    J_SHAFT["J-SHAFT<br/>(Main Shaft Junction)"]
    J_T2["J-T2<br/>(T2 Junction)"]
    J_T3["J-T3<br/>(T3 Junction)"]
    J_T4["J-T4<br/>(T4 Junction)"]
    J_T5["J-T5<br/>(T5 Junction)"]
    J_T6["J-T6<br/>(T6 Deep Heading)"]
    REFUGE1["REFUGE-1<br/>(Muster Point T2)"]
    REFUGE2["REFUGE-2<br/>(Refuge Chamber B)"]

    SURFACE ---|dist: 60| J_SHAFT
    J_SHAFT ---|dist: 100| J_T2
    J_SHAFT ---|dist: 230| J_T4
    J_T2 ---|dist: 130| J_T3
    J_T2 ---|dist: 0| REFUGE1
    J_T3 ---|dist: 260| J_T5
    J_T4 ---|dist: 350| J_T6
    J_T4 ---|dist: 80| REFUGE2
    J_T5 ---|dist: 140| J_T6

    classDef exit fill:#00f5a0,stroke:#000,color:#000,font-weight:bold;
    classDef refuge fill:#00d8ff,stroke:#000,color:#000,font-weight:bold;
    classDef junction fill:#161b22,stroke:#ffb300,color:#fff;

    class SURFACE exit;
    class REFUGE1,REFUGE2 refuge;
    class J_SHAFT,J_T2,J_T3,J_T4,J_T5,J_T6 junction;
```

---

## 8. Deployment Architecture

```mermaid
flowchart TD
    subgraph PhysicalMine["Physical Mine Site"]
        HELMET["12x Smart Helmets (ESP32)"]
        WRIST["Wearable Wristbands"]
        GATEWAY["Industrial LoRaWAN Gateway / WiFi AP"]
    end

    subgraph EdgeServer["On-Premises Mine Edge Server (Local Workstation / Rack)"]
        NODE["Node.js Application Runtime"]
        subgraph InternalComponents["Node.js Internal Architecture"]
            EX["Express Web Server (:3001)"]
            AED["Aedes MQTT Server (:1883)"]
            WSS["WebSocket Server (:3001)"]
            WASM["sql.js WASM Virtual SQLite"]
        end
        DISK[("mineguard.db<br/>(Local Storage)")]
    end

    subgraph LocalLAN["Surface Command Network (LAN)"]
        OP1["Operator Workstation 1 (Chrome)"]
        OP2["Safety Manager Laptop (Edge)"]
        WALL["Emergency Overview Wall Display"]
    end

    subgraph CloudWAN["External Cloud Infrastructure (WAN)"]
        TW_API["Twilio Telecom Gateway"]
        GROQ_API["Groq Cloud AI (Llama 3.3)"]
        SUPA_API["Optional Supabase Cloud Auth"]
    end

    HELMET -->|Wi-Fi 802.11 b/g/n| GATEWAY
    WRIST -->|LoRaWAN / Wi-Fi| GATEWAY
    GATEWAY -->|Ethernet LAN| EX
    GATEWAY -->|MQTT TCP 1883| AED

    NODE --- DISK

    EX <--> OP1
    EX <--> OP2
    EX <--> WALL
    WSS -.->|WebSocket Push| OP1
    WSS -.->|WebSocket Push| OP2
    WSS -.->|WebSocket Push| WALL

    NODE -->|HTTPS Outbound| TW_API
    NODE -->|HTTPS Outbound| GROQ_API
    NODE -.->|HTTPS Outbound| SUPA_API
```

### Deployment Rationale
Underground safety systems **must prioritize local survivability**. If an underground explosion or storm severs the surface facility’s external Internet connection:
- The local on-premises edge server continues to broker MQTT packets, execute sensor fusion, calculate Dijkstra evacuation routes, sound browser HUD sirens, and actuate physical helmet buzzers.
- Only external cloud enrichments (Groq AI Chatbot and Twilio outbound PSTN calling) degrade gracefully.
