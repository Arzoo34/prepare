# SafeMine: Architectural Decision Records (ADRs) & Trade-off Analysis

---

## 1. Overview & Decision Framework

In industrial engineering and mission-critical software design, no architecture is universally "best"—every architectural choice represents a conscious trade-off between **development velocity**, **runtime performance**, **operational complexity**, **hardware portability**, and **safety-critical reliability**.

This document captures the 12 pivotal architectural decisions made in the SafeMine codebase, formalized as **Architectural Decision Records (ADRs)**.

---

## 2. Architectural Decision Records (ADRs)

### ADR-01: In-Memory WebAssembly SQLite (`sql.js`) vs. Native SQLite (`better-sqlite3`) vs. Client-Server RDBMS (PostgreSQL)

- **Decision**: Use `sql.js` (SQLite compiled to WebAssembly via Emscripten) wrapped in a custom synchronous interface that writes exported memory buffers to `mineguard.db`.
- **Problem Solved**: Cross-platform execution on developer and edge machines without requiring native C++ build toolchains (node-gyp, Python, MSVC build tools).
- **Alternatives Considered**:
  1. `better-sqlite3`: Native C++ binding offering extreme performance ($100,000+\text{ ops/sec}$), zero-copy reads, and true OS file locking.
  2. `PostgreSQL`: Enterprise client-server relational database with concurrent read/write scaling, robust indexing, and point-in-time recovery.
- **Why Alternatives Were Not Chosen**:
  - `better-sqlite3` frequently fails installation on non-configured Windows environments due to missing Python and C++ compilation environments.
  - PostgreSQL introduces severe operational overhead (managing separate database daemons, connection pools, migrations, network ports, and container orchestration), complicating air-gapped mine-site setups.
- **Trade-offs Introduced**:
  - `sql.js` maintains the entire database in WebAssembly heap memory.
  - On every mutating query (`INSERT`, `UPDATE`, `DELETE`), `db.js` calls `_sqlDb.export()` and executes `fs.writeFileSync(DB_PATH, Buffer.from(data))`. This writes the *entire database file* to disk synchronously, introducing $O(N)$ disk I/O cost on every telemetry packet.
- **When the Alternative Becomes Better**:
  - `better-sqlite3` is preferable immediately once a standard deployment pipeline (Docker / Linux containers) is available.
  - PostgreSQL is mandatory if more than 50 concurrent miners stream telemetry simultaneously or if multi-master clustering is required across surface facilities.
- **Enterprise Scale Verdict**: Replace immediately with PostgreSQL with TimescaleDB extension for time-series telemetry storage.

---

### ADR-02: Embedded In-Process MQTT Broker (`aedes`) vs. Standalone Broker (`Mosquitto` / `EMQX`)

- **Decision**: Integrate the `aedes` MQTT broker directly inside the Node.js process listening on TCP port 1883 when `DUMMY_MODE=false`.
- **Problem Solved**: Provides out-of-the-box MQTT connectivity for LoRa gateways and ESP32 hardware without forcing users to install and configure external system daemons.
- **Alternatives Considered**:
  1. `Eclipse Mosquitto`: Lightweight, battle-tested C-based MQTT broker.
  2. `EMQX`: Erlang-based distributed industrial MQTT broker capable of millions of concurrent connections.
- **Why Alternatives Were Not Chosen**:
  - Requiring an external Mosquitto installation creates setup friction, configuration errors, and background service management issues during demonstration and evaluation.
- **Trade-offs Introduced**:
  - Aedes runs on the same Node.js single-threaded event loop as the Express API and WebSocket server. A flood of incoming MQTT packets can monopolize the CPU, causing latency spikes in HTTP requests and WebSocket broadcasts.
- **When the Alternative Becomes Better**:
  - When deployment targets more than 100 physical hardware nodes. An external Mosquitto daemon running in a separate process isolates message brokering from application crashes.
- **Enterprise Scale Verdict**: Deploy a clustered EMQX or Mosquitto container behind an NGINX reverse proxy.

---

### ADR-03: Monolithic Node.js Application vs. Microservices Architecture

- **Decision**: Implement all system capabilities (telemetry ingestion, optimization, alerting, chat proxy, and file serving) within a single Node.js monolith.
- **Problem Solved**: Eliminates inter-service network latency, complex RPC serialization, service discovery requirements, and distributed failure states.
- **Alternatives Considered**:
  1. Multi-service microservices: Ingestion Service (Go), Optimization Engine (Python/NetworkX), Alerting Service (Node.js), Gateway (Kong/Envoy).
- **Why Alternatives Were Not Chosen**:
  - Underground mine safety requires deterministic simplicity. Managing container meshes (Kubernetes/Docker Compose) underground increases downtime risk due to network partitions between nodes.
- **Trade-offs Introduced**:
  - Tight coupling. A memory leak or unhandled exception in the optimization loop (`optimizer.js`) can crash the telemetry ingestion server.
- **When the Alternative Becomes Better**:
  - When separate engineering teams manage the optimization math, firmware protocols, and frontend interfaces independently.
- **Enterprise Scale Verdict**: Decouple into two services: a high-throughput Ingestion & Telemetry Gateway (Go/Rust) and an Operations & Analytics Core (Node.js/Python).

---

### ADR-04: Dual-Protocol Ingestion (HTTP REST + MQTT) vs. Pure MQTT

- **Decision**: Expose both `POST /api/telemetry` (HTTP) and MQTT topics (`mineguard/helmet/#`) for sensor data submission.
- **Problem Solved**: Enables direct integration with standard ESP32 HTTP client firmware while maintaining native compatibility with industrial LoRaWAN gateways.
- **Alternatives Considered**:
  1. Pure MQTT: Force all hardware and gateways to communicate via pub/sub.
  2. Pure HTTP: Forbid MQTT and use standard REST endpoints only.
- **Why Alternatives Were Not Chosen**:
  - While MQTT is ideal for constrained networks (smaller packet overhead), many educational and prototype ESP32 sketches use standard HTTPClient due to simpler debugging and lack of keep-alive/ping overhead.
- **Trade-offs Introduced**:
  - Code duplication in packet normalization: `server.js` maintains both `handleHelmetUpdate` (MQTT) and the `POST /api/telemetry` handler with field name aliasing.
- **When the Alternative Becomes Better**:
  - Pure MQTT becomes superior in production because persistent TCP/MQTT connections avoid the overhead of recurring TCP 3-way handshakes and HTTP header retransmission.
- **Enterprise Scale Verdict**: Standardize entirely on MQTT / CoAP for edge nodes, exposing HTTP strictly for web client operations.

---

### ADR-05: HTML5 Canvas 2D Schematic + Leaflet GPS vs. 3D WebGL / Three.js

- **Decision**: Build the subterranean view on a custom HTML5 Canvas 2D engine and the surface view on Leaflet.js tiles.
- **Problem Solved**: High-performance, 60 FPS rendering on low-spec control room laptops without requiring discrete GPUs.
- **Alternatives Considered**:
  1. Three.js / WebGL 3D Mine Model: Full 3D rendering of geological scans, declines, and stopes.
- **Why Alternatives Were Not Chosen**:
  - 3D models require complex CAD/BIM assets, high GPU fill rates, complex camera controls, and higher battery/CPU overhead. In an emergency, a 2D planar tactical schematic provides faster cognitive comprehension.
- **Trade-offs Introduced**:
  - Tunnels overlapping in the vertical plane (different depths) must be represented schematically rather than through true spatial geometry.
- **When the Alternative Becomes Better**:
  - When the mine operates deep multi-level ore bodies where vertical shaft clearance, incline gradients, and structural pillars must be inspected in 3D.
- **Enterprise Scale Verdict**: Hybrid interface: Canvas 2D as the primary tactical HUD, with an optional Three.js Digital Twin viewport.

---

### ADR-06: Raw WebSockets (`ws`) vs. Socket.io vs. Server-Sent Events (SSE)

- **Decision**: Use the minimalist `ws` library on the server and native browser `WebSocket` API on the client.
- **Problem Solved**: Minimalist memory footprint, zero client-library dependencies, and standard RFC 6455 compliance.
- **Alternatives Considered**:
  1. `Socket.io`: Adds auto-reconnection, HTTP long-polling fallback, and room multiplexing.
  2. `Server-Sent Events (SSE)`: Native browser unidirectional HTTP streaming.
- **Why Alternatives Were Not Chosen**:
  - `Socket.io` introduces proprietary packet framing, requires custom client libraries, and adds memory bloat.
  - SSE is strictly unidirectional (server-to-client), preventing bidirectional client pinging and downlink triggers over the same socket.
- **Trade-offs Introduced**:
  - Reconnection and message routing logic had to be implemented manually in `api.js` (`wsConnect`, `wsReconnectTimer`).
- **When the Alternative Becomes Better**:
  - Socket.io is preferable if clients frequently operate behind restrictive corporate firewalls that block standard WebSocket handshakes.
- **Enterprise Scale Verdict**: Raw WebSockets over TLS (`wss://`) managed behind an API Gateway / Envoy proxy.

---

### ADR-07: Vanilla ES6 JavaScript vs. Modern Frontend Framework (React / Vue)

- **Decision**: Author the frontend interface in pure Vanilla JavaScript (ES6+), semantic HTML5, and native CSS variables without a build or bundling step.
- **Problem Solved**: Zero build step (`npm run build`), no webpack/Vite configuration, instant file-edit reloading, and complete freedom from framework churn.
- **Alternatives Considered**:
  1. `React.js` + `Tailwind CSS`: Componentized UI with virtual DOM reconciliation.
  2. `Vue.js`: Declarative template bindings and reactive state management.
- **Why Alternatives Were Not Chosen**:
  - Eliminating a bundler allows the application to be served directly from any static web server or the Node.js backend root without transpilation.
- **Trade-offs Introduced**:
  - Imperative DOM manipulation (`document.getElementById`, `innerHTML`).
  - High risk of DOM desynchronization as `app.js` grew to over 2,200 lines.
- **When the Alternative Becomes Better**:
  - When the team expands beyond 2 developers and complex component state requires strict lifecycle boundaries.
- **Enterprise Scale Verdict**: React or Svelte with TypeScript for type safety, modular components, and automated testing.

---

### ADR-08: Physics-Inspired Respiration Model vs. Static Threshold Checking

- **Decision**: Formulate oxygen depletion dynamically:
  $$\text{O}_2 = 20.9\% - \min(N_{\text{workers}} \times 0.12, 2.5) - \min(\text{Gas}\% \times 1.2, 4.0) + \text{noise}$$
- **Problem Solved**: Replaces unrealistic static O2 values with an environmental model reflecting actual underground atmospheric dynamics.
- **Alternatives Considered**:
  1. Static scalar thresholding: Treating O2 as an isolated sensor reading with a constant default of $20.9\%$.
  2. Computational Fluid Dynamics (CFD): Full 3D finite-element simulation of airflow and gas dispersion.
- **Why Alternatives Were Not Chosen**:
  - Static fallback values hide the reality that multiple workers consuming oxygen in unventilated headings cause rapid depletion.
  - Full CFD modeling requires supercomputing clusters and cannot run in real time on microcontrollers or edge servers.
- **Trade-offs Introduced**:
  - Requires active worker count tracking across tunnel sections to maintain accurate estimates.
- **When the Alternative Becomes Better**:
  - Hardware with dedicated, calibrated electrochemical O2 sensors makes software estimation redundant.
- **Enterprise Scale Verdict**: Use real electrochemical O2 sensors on all helmets, using the mathematical model as a sensor-fault validation check.

---

### ADR-09: Dynamic Edge-Penalized Dijkstra vs. Standard A* vs. Floyd-Warshall

- **Decision**: Implement Dijkstra's algorithm with dynamic edge weights modified by hazard geofences ($+500$) and worker congestion ($+15/\text{miner}$).
- **Problem Solved**: Computes the true shortest evacuation route to safety while actively routing personnel around toxic gas zones and bottlenecks.
- **Alternatives Considered**:
  1. $A^*$ Search: Heuristic-guided search algorithm.
  2. Floyd-Warshall: All-pairs shortest path precomputation.
- **Why Alternatives Were Not Chosen**:
  - $A^*$ requires a consistent geometric Euclidean distance heuristic ($h(n)$) to the exit. In tunnel networks, Euclidean distance often misleads searches toward impassable rock walls.
  - Floyd-Warshall ($O(V^3)$) precomputes all paths, but SafeMine's edge weights mutate in real time as gas clouds drift and workers move, invalidating static matrices.
- **Trade-offs Introduced**:
  - Running Dijkstra for multiple workers concurrently has $O(W \cdot (V^2 + E))$ complexity in this array-based implementation.
- **When the Alternative Becomes Better**:
  - $A^*$ becomes superior if mine geometry is represented as a large-scale grid mesh ($10,000+$ nodes) with clear directional headings.
- **Enterprise Scale Verdict**: Dijkstra with a Min-Heap priority queue ($O((V+E)\log V)$) running on an in-memory graph service.

---

### ADR-10: Groq Cloud LLM (Llama 3.3) vs. Local Ollama vs. OpenAI GPT-4

- **Decision**: Integrate Groq API running `llama-3.3-70b-versatile` with prompt-injected live mine context, with Anthropic Claude as a fallback.
- **Problem Solved**: Achieves sub-second token generation speeds ($>250\text{ tokens/sec}$) essential for time-critical emergency queries, with zero on-premise GPU hardware requirements.
- **Alternatives Considered**:
  1. Local Ollama / vLLM: Running an open-source model on local server GPUs.
  2. OpenAI GPT-4o: Cloud API with high general intelligence.
- **Why Alternatives Were Not Chosen**:
  - Local LLMs require expensive dedicated enterprise GPUs (e.g., NVIDIA A100/RTX 4090) that standard mine workstations lack.
  - OpenAI API token generation latency ($1\text{--}3\text{ seconds}$) is too slow for interactive emergency dispatch operations.
- **Trade-offs Introduced**:
  - Requires active WAN/Internet connectivity. If external satellite links fail, the AI assistant goes offline.
- **When the Alternative Becomes Better**:
  - A local quantized model (e.g., Llama 3 8B via Ollama on an RTX 4060) is superior for strictly air-gapped mines with no Internet access.
- **Enterprise Scale Verdict**: Dual-engine setup: Local 8B LLM running on-premise for offline survival, with Groq cloud failover when online.

---

### ADR-11: Twilio Telecom APIs vs. AWS SNS/Pinpoint vs. Physical GSM Modem Hardware

- **Decision**: Integrate Twilio's REST API for automated voice calling (TwiML) and SMS dispatch to the Mine Safety Manager.
- **Problem Solved**: Delivers automated voice calls with text-to-speech alert synthesis and international SMS without dedicated telephony hardware.
- **Alternatives Considered**:
  1. AWS SNS / Pinpoint: Cloud notification service.
  2. Physical USB GSM / 4G Modem: Serial AT-command hardware connected to the edge server.
- **Why Alternatives Were Not Chosen**:
  - AWS SNS voice capabilities require complex Amazon Connect setups and lack rapid TwiML-style programmatic speech generation.
  - Physical GSM modems require SIM cards, signal coverage in the server room, and are prone to hardware driver lockups.
- **Trade-offs Introduced**:
  - Recurring per-call/per-SMS API costs and dependence on external cloud connectivity.
- **When the Alternative Becomes Better**:
  - Physical GSM modems or dedicated Leaky Feeder VHF radio repeaters are superior when the surface facility experiences frequent WAN outages.
- **Enterprise Scale Verdict**: Twilio for cloud notifications backed by a physical industrial GSM/VHF paging gateway on-premise.

---

### ADR-12: Dual-Tier Wearable Architecture (Smart Helmet + Wristband)

- **Decision**: Implement a two-tiered redundancy architecture: primary telemetry via Smart Helmet (`/api/telemetry`), secondary vital sign & SOS tracking via Smart Wristband (`/api/wristband`).
- **Problem Solved**: Resolves the single-point-of-failure vulnerability of smart helmets. If a worker removes their helmet to rest or if the helmet is knocked off during a rockfall, the system retains vital sign monitoring and SOS capability.
- **Alternatives Considered**:
  1. Helmet-Only Tracking: Single hardware device per worker.
  2. Environmental Anchor Monitoring Only: Fixed tunnel wall sensors with no personal wearables.
- **Why Alternatives Were Not Chosen**:
  - Helmet-only systems go completely blind when a miner removes equipment.
  - Fixed wall sensors cannot track individual heart rates, personal falls, or localized toxic pockets in working stopes.
- **Trade-offs Introduced**:
  - Requires maintaining two separate battery charging cycles, hardware states, and data ingestion routes per worker.
- **When the Alternative Becomes Better**:
  - Helmet-only systems are suitable only for budget-constrained exploratory deployments.
- **Enterprise Scale Verdict**: Industry-standard best practice. Dual-device redundancy is mandated by modern tier-1 mining safety specifications.

---

## 3. Comprehensive Architecture Decision Matrix

| Dimension | Chosen Approach | Alternative Considered | Trade-off Made | Enterprise Recommendation |
| :--- | :--- | :--- | :--- | :--- |
| **Database** | `sql.js` (WASM SQLite) | `better-sqlite3` / PostgreSQL | Easy setup vs synchronous disk I/O | PostgreSQL + TimescaleDB |
| **MQTT Broker** | Embedded Aedes | Mosquitto / EMQX | Single process vs event loop contention | Clustered EMQX in Docker |
| **Architecture** | Modular Monolith | Microservices | Simplicity vs failure isolation | Ingestion Gateway + Core Service |
| **Ingestion** | HTTP REST + MQTT | Pure MQTT | Prototype flexibility vs protocol overhead | Pure MQTT / CoAP |
| **Visualization** | Canvas 2D + Leaflet | Three.js WebGL | Fast 2D rendering vs 3D fidelity | Hybrid 2D Tactical + 3D View |
| **Streaming** | Raw WebSockets (`ws`) | Socket.io / SSE | Standards-based vs manual reconnect logic | WebSockets over TLS |
| **Frontend** | Vanilla ES6 JavaScript | React / TypeScript | Zero build step vs imperative DOM complexity | Svelte / TypeScript |
| **O2 Model** | Respiration Formula | Static Scalar Checking | Model tracking vs dedicated sensor reliance | Electrochemical Sensor + Model Validation |
| **Pathfinding** | Edge-Penalized Dijkstra | $A^*$ / Floyd-Warshall | Real-time dynamic weights vs $O(V^2)$ cost | Dijkstra with Min-Heap |
| **AI Assistant** | Groq API (Llama 3.3) | Local Ollama | Fast inference vs cloud dependence | Hybrid: Local LLM + Cloud Failover |
| **Telephony** | Twilio REST / TwiML | GSM Serial Modems | Zero hardware vs Internet dependence | Twilio + Hardware VHF Radio Bridge |
| **Wearables** | Helmet + Wristband | Helmet Only | Double device overhead vs zero-blindspot safety | Dual Wearable Architecture |
