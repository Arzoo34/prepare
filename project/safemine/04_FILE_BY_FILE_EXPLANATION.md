# SafeMine: Exhaustive File-by-File Technical Deep Dive

---

## 1. Directory Tree & File Inventory

```
SafeMine/
│
├── ESP32.ino                          # C++ firmware for ESP32 Smart Helmet hardware
├── ESP32.info                         # Firmware documentation & configuration variant
├── index.html                         # Primary single-page application (SPA) shell
├── login.html                         # Operator authentication interface
├── signup.html                        # New operator registration interface
├── underground-tracker.html           # Hardware integration workbench & wiring guide
├── package.json                       # Top-level workspace orchestration manifest
│
├── backend/                           # Node.js backend services & data layer
│   ├── server.js                      # Core monolithic application engine
│   ├── db.js                          # WebAssembly SQLite layer (sql.js)
│   ├── auth.js                        # JWT & Supabase authentication gate
│   ├── o2Calculation.js               # Oxygen depletion & gas displacement physics model
│   ├── optimizer.js                   # Dijkstra evacuation routing & multi-objective optimizer
│   ├── emergency-contact.js           # Twilio SMS & voice telephony dispatcher
│   ├── dummy-engine.js                # Internal 12-worker telemetry simulation loop
│   ├── lora-gateway.js                # LoRaWAN gateway bridge & MQTT publisher
│   ├── esp32_simulator.js             # Standalone HTTP telemetry simulation script
│   ├── simulator.js                   # MQTT telemetry simulator for Aedes broker
│   ├── seed.js                        # Database initialization & mock data seeder
│   ├── reset-db.js                    # Database cleanup script
│   ├── patch-gps.js                   # GPS coordinate patching script
│   ├── _db_query.js                   # CLI database inspection utility
│   ├── _db_runner.js                  # Batch SQL runner for database migrations
│   ├── package.json                   # Backend dependencies & script definitions
│   └── README.md                      # Backend service documentation
│
└── frontend/                          # Client-side presentation assets
    ├── app.js                         # Main UI controller & Canvas 2D render engine
    ├── admin-dashboard.js             # Executive dashboard & optimization controller
    ├── styles.css                     # Primary design system stylesheet
    ├── admin-dashboard.css            # Executive dashboard styling
    ├── admin-dashboard-fixes.css      # Layout & overflow bug fixes (Audit: Unlinked)
    ├── new-features.css               # Ancillary feature styling
    ├── auth-pages.css                 # Login & registration styling
    ├── tour.js                        # Intro.js interactive walkthrough (Audit: Unlinked)
    ├── tour.css                       # Walkthrough styling (Audit: Unlinked)
    ├── auth/
    │   ├── login-app.js               # Login client logic (Supabase integration)
    │   └── signup-app.js              # Signup client logic (Supabase integration)
    ├── bot/
    │   └── chatbot.js                 # Floating AI assistant widget
    ├── data/
    │   └── mockData.js                # Legacy file audit (contains CSS, unreferenced)
    └── utils/
        ├── api.js                     # REST API client & WebSocket manager
        └── chartHelpers.js            # Chart.js defaults & color palettes
```

---

## 2. Hardware & Firmware Layer

### `ESP32.ino` (and `ESP32.info`)
- **File Path**: `/ESP32.ino` (331 lines)
- **Role in Architecture**: Edge firmware executing on the physical ESP32 Smart Helmet. Responsible for sensor sampling, local hazard classification, audio buzzer alarm actuation, edge O2 calculation, and HTTP telemetry transmission.
- **Key Dependencies & Hardware Libraries**:
  - `<WiFi.h>`: ESP32 802.11 b/g/n wireless network stack.
  - `<HTTPClient.h>`: Synchronous HTTP client for REST POST dispatch.
  - `<DHT.h>`: Single-bus timing protocol for DHT11 temperature and humidity.
  - `<Wire.h>`: Hardware I2C bus master driver (SDA on GPIO 21, SCL on GPIO 22).
  - `<MPU6050.h>`: 6-axis I2C accelerometer and gyroscope.
  - `<Adafruit_BMP085.h>`: I2C digital barometric pressure sensor (BMP085/BMP180).
- **Pin Allocations**:
  - `DHTPIN 4`: DHT11 data pin.
  - `MQ2_PIN 34`: Analog input connected to MQ-2 gas sensor (ADC1 channel 6).
  - `BUZZER 25`: Digital output driving the active piezoelectric buzzer.
- **Critical Algorithms & Implementation Details**:
  1. **Edge Oxygen Estimation (`calculateOxygenLevel`, lines 63–87)**:
     ```cpp
     float calculateOxygenLevel(int workerCount, float gasPercent) {
       float workerDrop = workerCount * 0.12;
       if (workerDrop > 2.5) workerDrop = 2.5;
       float gasDrop = gasPercent * 1.2;
       if (gasDrop > 4.0) gasDrop = 4.0;
       float noise = ((millis() % 100) / 1000.0) - 0.05;
       float o2 = 20.9 - workerDrop - gasDrop + noise;
       if (o2 < 15.0) o2 = 15.0;
       if (o2 > 21.0) o2 = 21.0;
       return o2;
     }
     ```
     *Engineering Explanation*: Implements the exact same mathematical formula on the microcontroller as the server-side `o2Calculation.js`. This allows the helmet to trigger its buzzer if O2 drops below $18.0\%$ even during a wireless disconnection.
  2. **Fall & Impact Detection Engine (lines 211–235)**:
     - Normalizes raw 16-bit accelerometer counts to g-forces: $x = ax / 16384.0$.
     - Computes vector magnitude: $\text{magnitude} = \sqrt{x^2 + y^2 + z^2}$.
     - Detects severe impact: $\text{shock} = (\text{magnitude} > 3.0\text{g})$.
     - Detects post-impact prone orientation: $\text{tilt} = (z < 0.3\text{g})$.
     - Asserts fall: If shock and tilt occur within an impact window of $5000\text{ms}$ and are followed by complete absence of motion ($\Delta \le 0.15\text{g}$).
  3. **Atmospheric Barometric Drop Detection (lines 258–276)**:
     - Detects sudden atmospheric depressurization: If $\Delta P = P_{\text{prev}} - P_{\text{current}} > 20\text{ hPa}$, triggers a `PRESSURE_DROP` alert, indicating a rapid ventilation fan failure or blast overpressure wave.
- **Differences with `ESP32.info`**:
  - `ESP32.info` is an identical firmware backup containing an alternate target backend IP (`http://192.168.50.88/api/telemetry` vs `http://192.168.50.12:3001/api/telemetry`).
- **Potential Bugs & Edge Cases**:
  - `WiFi.status() != WL_CONNECTED` uses a blocking `while` loop with `delay(500)` during reconnection (line 133), stalling sensor sampling and local fall detection for up to 10 seconds.
  - `millis()` rolls over after approximately 49.7 days, which could cause arithmetic errors in timing comparisons if unsigned subtraction is not strictly followed.
- **Sample Technical Interview Question**:
  - *Q: Why did you divide `ax` by 16384.0 in your MPU-6050 reading?*
  - *A: The MPU-6050 defaults to an accelerometer full-scale range of $\pm 2\text{g}$. Since the onboard analog-to-digital converter has 16-bit resolution ($2^{16} = 65,536$ states), the sensitivity scale factor for $\pm 2\text{g}$ is $\frac{65536}{4\text{g}} = 16,384\text{ LSB/g}$. Dividing raw 16-bit integers by 16,384 yields acceleration directly in units of earth gravity ($g$).*

---

## 3. Backend Services Layer

### `backend/server.js`
- **File Path**: `/backend/server.js` (1,982 lines)
- **Role in Architecture**: Central monolithic application server. Hosts the Express HTTP server, WebSocket broadcast server, embedded Aedes MQTT broker, sensor health monitor, alert escalation timers, and background optimization loops.
- **Key Imports**: `express`, `cors`, `http`, `net`, `ws`, `express-rate-limit`, `uuid`, `./db`, `mqtt`, `./simulator`, `./dummy-engine`, `./optimizer`, `./emergency-contact`, `./o2Calculation`.
- **Key Subsystems & Execution Flows**:
  1. **Aedes Embedded MQTT Broker (lines 26–40)**:
     - Instantiated when `DUMMY_MODE=false`. Listens on TCP port 1883 via Node's native `net.createServer(aedes.handle)`.
  2. **WebSocket Broadcast Hub (lines 72–76)**:
     - `broadcast(type, payload)` iterates through all `wss.clients` in `WebSocket.OPEN` state and delivers JSON-stringified event messages.
  3. **Sensor Health & Drift Detector (`getSensorState`, lines 78–176)**:
     - Maintains in-memory streak counters per worker. Detects flatlines (`ch4ZeroStreak`, `ch4MaxStreak`), sudden unrealistic jumps ($\Delta\text{CH}_4 \ge 1.0\%$, $\Delta\text{Temp} \ge 15^\circ\text{C}$), and missing packets.
     - Logs faults into the `sensor_faults` table with `repair_status='required'`.
  4. **Hardware Exemption Registry (lines 940–952)**:
     - `realHardwareWorkers` Map records timestamps when real ESP32 devices POST to `/api/telemetry`. Prevents internal dummy engines from modifying real worker records for $10,000\text{ms}$.
  5. **Three-Stage SOS Escalation (`startEscalation`, lines 1485–1530)**:
     - Warning Stage ($0\text{s}$) $\to$ Critical Stage ($10\text{s}$) $\to$ Panic Stage ($30\text{s}$). Automatically marks worker `status='emergency'`, issues MQTT downlink alarm command, and calls `notifyAdmin()`.
  6. **Groq / Claude AI Assistant Gateway (lines 1278–1374)**:
     - Synthesizes current underground telemetry (active alerts, peak gas, danger geofences, high-risk miners) into a prompt injected into Groq's `llama-3.3-70b-versatile` model.
- **Critical Code Flaws & Architectural Vulnerabilities**:
  - `auth.js` is imported nowhere in `server.js`. The entire API gate is dormant.
  - Static file middleware `app.use(express.static(...))` serves the entire project root (`..`), exposing source code, config files, and SQLite database downloads if accessed via `/backend/mineguard.db`.
- **Sample Technical Interview Question**:
  - *Q: How does `server.js` handle race conditions between incoming real hardware telemetry and the internal simulation loop?*
  - *A: Through an in-memory TTL registry called `realHardwareWorkers`. When `/api/telemetry` receives a packet from a physical device (e.g. `MNR-001`), it records `realHardwareWorkers.set('MNR-001', Date.now())`. When the background dummy engine runs every 3 seconds, it executes `isRealHardwareWorker('MNR-001')`. If a real packet was received within the last 10 seconds (`REAL_HW_TTL_MS`), the dummy engine returns early, preserving physical hardware telemetry from simulation overwrites.*

---

### `backend/db.js`
- **File Path**: `/backend/db.js` (303 lines)
- **Role in Architecture**: Data access layer wrapping WebAssembly `sql.js` with an API compatible with `better-sqlite3`.
- **Mechanism**:
  - Initializes WebAssembly SQLite asynchronously: `const SQL = await initSqlJs()`.
  - Reads existing database binary from disk (`mineguard.db`) into memory.
  - Implements `db.prepare(sql)` returning an object with `all()`, `get()`, and `run()`.
  - Converts named SQLite parameters (`@worker_id`) to positional placeholders (`?`) via `_toPositional()`.
- **Disk Synchronization (`saveDB`, line 148)**:
  ```javascript
  function saveDB() {
    if (!_sqlDb) return;
    try {
      const data = _sqlDb.export();
      fs.writeFileSync(DB_PATH, Buffer.from(data));
    } catch (e) { /* ignore */ }
  }
  ```
- **Performance Evaluation**:
  - WebAssembly memory operations are fast, but `_sqlDb.export()` serializes the entire database into a Uint8Array, and `fs.writeFileSync` blocks the event loop while writing the multi-megabyte file to disk. Under high write loads, this is a major performance bottleneck.
- **Sample Technical Interview Question**:
  - *Q: Why wrap `sql.js` in a synchronous interface rather than using asynchronous promises?*
  - *A: To emulate the synchronous API design of `better-sqlite3`. This allowed legacy synchronous backend route handlers to query the database inline (`db.prepare(...).get(...)`) without refactoring hundreds of call sites into `async/await` chains. However, this creates synchronous event loop blocking during disk flushes.*

---

### `backend/optimizer.js`
- **File Path**: `/backend/optimizer.js` (410 lines)
- **Role in Architecture**: Multi-objective safety optimization engine. Houses algorithmic implementations for risk scoring, Dijkstra evacuation routing, greedy proportional ventilation, worker distribution balancing, and predictive fatigue modeling.
- **Core Algorithms**:
  1. **`calcOptimizedRisk(w)` (lines 14–59)**:
     - Applies non-linear log penalties for flammable gas: $\min(4.0, \ln(1 + \text{fg}) \times 0.72)$.
     - Applies exponential penalties for oxygen deficiency: $\min(3.0, e^{(20.9 - \text{o2}) \times 1.2} \times 0.3)$.
     - Adds compound hazard penalties: If $\ge 2$ hazards co-occur, adds $+0.8 \times \text{compoundCount}$.
  2. **`dijkstraEvacRoute(workerX, workerY, workers, geofences)` (lines 91–192)**:
     - Graph nodes represent tunnel junctions (`J-SHAFT`, `J-T2`, etc.) and refuge chambers (`REFUGE-1`, `SURFACE`).
     - Maps worker $(x,y)$ to nearest tunnel node.
     - Adds a $+500$ penalty if the edge midpoint intersects an active hazard geofence.
     - Adds a $+15$ congestion penalty for every other worker located within 30 units of the edge.
     - Traverses via Dijkstra's shortest path to the lowest-cost exit/refuge.
  3. **`optimizeVentilation(workers, threshold)` (lines 199–237)**:
     - Computes zone hazard priority:
       $$\text{Priority} = \left(\frac{\text{FG}}{200} \times 55\right) + \left(\frac{\text{CH}_4}{1.0} \times 15\right) + \left(\frac{\text{CO}}{50} \times 15\right) + \left(\frac{\max(0, 19.5 - \text{O}_2)}{2.5} \times 15\right)$$
     - Allocates fan speeds proportionally: $\text{FanSpeed}\% = \min(100, 20 + \text{Priority} \times 0.8)$.
  4. **`predictFatigue(workers)` (lines 301–332)**:
     - Approximates hours underground from helmet battery drain: $\text{Hours} = \frac{100 - \text{battery}}{100} \times 8.0$.
     - Computes fatigue index ($0\text{--}10$) by weighting elevated heart rate, shift time, lack of motion late in shift, and ambient risk.
- **Complexity**: Dijkstra implementation uses an unsorted array with `queue.sort((a, b) => a.d - b.d)` inside the loop, resulting in $O(V^2 + E)$ complexity.
- **Sample Technical Interview Question**:
  - *Q: Why does the Dijkstra implementation penalize hazard edges rather than pruning them completely?*
  - *A: Hard-pruning an edge containing a gas hazard can partition the graph, making the destination mathematically unreachable and returning no route. By applying a large penalty ($+500$) instead of removal, the algorithm will prefer any available safe alternative, but can still return the hazardous passage as an absolute last resort if all other escape routes are blocked by cave-ins.*

---

### `backend/o2Calculation.js`
- **File Path**: `/backend/o2Calculation.js` (181 lines)
- **Role in Architecture**: Physics-inspired oxygen depletion calculation engine. Replaces static sensor dummy data with an environmental respiration and gas displacement model.
- **Mathematical Constants**:
  - Baseline atmospheric O2: $20.9\%$.
  - Respiration consumption rate: $0.12\%$ drop per active worker (capped at $2.5\%$).
  - Gas displacement coefficient: $1.2 \times \text{CH}_4\%$ (capped at $4.0\%$).
  - Jitter noise: $\pm 0.06\%$.
- **Helper Functions**:
  - `getO2Label(o2)`: Returns `'DANGER'` ($<18.0\%$), `'WARNING'` ($18.0\text{--}19.5\%$), or `'SAFE'` ($\ge 19.5\%$).
  - `getO2Status(o2)`: Returns structured object with numeric value, string label, and hex color code (`#ef4444`, `#f59e0b`, `#22c55e`).
- **Sample Technical Interview Question**:
  - *Q: What physical phenomenon does the 1.2 gas coefficient represent?*
  - *A: Dalton's Law of Partial Pressures. When methane ($\text{CH}_4$) or volatile hydrocarbons leak into a confined, unventilated subterranean stope, the foreign gas increases the total gas volume and displaces ambient air. Because air is roughly $21\%$ oxygen and $78\%$ nitrogen, methane accumulation dilutes the partial pressure of oxygen proportionally.*

---

### `backend/emergency-contact.js`
- **File Path**: `/backend/emergency-contact.js` (199 lines)
- **Role in Architecture**: Emergency communication gateway interfacing with the Twilio REST API for automated voice calling and SMS dispatch.
- **Execution Mechanism**:
  - Inspects environment variables: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`, and `ADMIN_PHONE`.
  - If keys are present, dispatches HTTP POST requests to `https://api.twilio.com/2010-04-01/Accounts/.../Calls.json` with dynamic TwiML text-to-speech XML.
  - If keys are absent, runs in simulated mode, appending outbound messages to an in-memory `callLog` ring buffer (max 50 entries) and broadcasting updates over WebSockets.
- **Throttling Protection**: `server.js` throttles automated notifications to at most one call/SMS per worker per 5 minutes to prevent alert storms and excessive telecom charges.
- **Sample Technical Interview Question**:
  - *Q: How does the system prevent sensitive telecom credentials or personal phone numbers from leaking to browser clients?*
  - *A: Through the `maskPhone()` utility and `getAdminConfig()`. The server masks phone numbers (e.g. `+123****890`) and returns only boolean configuration flags (`twilioConfigured: true/false`) to the frontend rather than raw API tokens or full phone numbers.*

---

### `backend/auth.js`
- **File Path**: `/backend/auth.js` (254 lines)
- **Role in Architecture**: Authentication router and JWT validation middleware. Supports both local SQLite user accounts (bcrypt hashing) and external Supabase JWT validation.
- **Key Functions**:
  - `createAuthRouter(db)`: Exposes `POST /register`, `POST /login`, and `GET /me`. First registered user receives role `admin`, subsequent users receive role `operator`.
  - `createRequireAuth(db)`: Middleware inspecting `Authorization: Bearer <token>`.
  - `createAuthApiGate(db)`: Intercepts all `/api/*` requests when `AUTH_ENABLED=true`, permitting access to public endpoints (`/health`, `/telemetry`, `/wristband`, `/auth/*`).
- **Code Audit Finding**:
  - **Unmounted Router**: `server.js` does not mount `auth.js`. Consequently, `/api/auth/register`, `/api/auth/login`, and `/api/auth/public-config` return 404/index.html responses. This represents a known integration bug in the current release.

---

### Ancillary Backend Scripts
- **`backend/dummy-engine.js` (281 lines)**: Internal telemetry generator simulating 12 miners with random walk drift, positional movement, and periodic methane spikes in Zone C.
- **`backend/lora-gateway.js` (172 lines)**: Connects to the MQTT broker and simulates hardware helmets and repeater anchors by publishing packets to `mineguard/helmet/:id`.
- **`backend/esp32_simulator.js` (67 lines)**: Standalone test script sending HTTP POST payloads to `/api/telemetry` every 3000ms.
- **`backend/seed.js` (246 lines)**: Seeds SQLite database with 12 miners, 10 anchors, 6 geofences, and initial incidents.
- **`backend/patch-gps.js` (99 lines)**: Utility setting mine GPS center coordinates ($6.2041^\circ\text{ N}, -1.6747^\circ\text{ W}$, Obuasi Gold Mine, Ghana) and assigning worker coordinate offsets.
- **`backend/reset-db.js` (19 lines)**: Deletes `mineguard.db` from disk to allow a clean re-initialization.

---

## 4. Frontend Client Layer

### `index.html`
- **File Path**: `/index.html` (1,339 lines)
- **Role in Architecture**: Single-page application markup shell.
- **Structure**:
  - Left navigation sidebar (`<nav class="nav">`) defining 19 views: Monitoring (`dashboard`, `map`, `workers`, `alerts`), Safety (`gas`, `health`, `evacuation`), Network (`network`, `incidents`, `aiassist`), Advanced (`analytics`, `simulation`, `drones`, `muster`, `geofence`, `admin`, `maintenance`), and Command (`admindash`, `optimization`).
  - Top action marquee displaying real-time safety status, active worker count, shift timer, and WebSocket connection indicator.
  - Modals for adding workers, initiating zone evacuations, logging incidents, defining geofences, and dispatching mine-wide alerts.
- **External CDN Dependencies**:
  - Chart.js (`v4.4.0`): Time-series and bar chart rendering.
  - Leaflet.js (`v1.9.4`): Surface GPS tile rendering and marker mapping.

---

### `frontend/app.js`
- **File Path**: `/frontend/app.js` (2,260 lines)
- **Role in Architecture**: Primary client controller. Manages WebSocket event listeners, DOM state reconciliation, and the HTML5 Canvas 2D underground rendering engine.
- **Key Modules**:
  1. **WebSocket Handler (`wireWebSocket`, lines 104–185)**:
     - Subscribes to events: `WORKER_UPDATE`, `NEW_ALERT`, `ANCHOR_UPDATE`, `ESCALATION_STAGE`, `OPTIMIZATION_UPDATE`, and `SETTINGS_UPDATED`.
  2. **Canvas 2D Subterranean Map Engine (`drawMap`, lines 500–810)**:
     - Renders tunnel paths between anchor coordinates on a $900 \times 520$ virtual coordinate system.
     - Draws dynamic depth color gradients (surface green $\to$ deep orange $\to$ abyssal red).
     - Renders worker nodes with status halos, pulsing emergency circles, and vector trail paths.
     - Implements collision detection on canvas click events to open the worker detail modal.
  3. **Leaflet Surface GPS Integration (`initSurfaceMap`, lines 843–913)**:
     - Initializes Leaflet map centered on `mineLat` / `mineLng`.
     - Maps workers with real GPS coordinates, rendering custom SVG helmet markers.
  4. **Web Audio API Sound Synthesizer (`notifyWorker`, lines 55–65)**:
     - Generates emergency siren tones directly in the browser using an `AudioContext` oscillator, alternating frequencies between $880\text{Hz}$ and $440\text{Hz}$ without requiring audio MP3 files.

---

### `frontend/admin-dashboard.js`
- **File Path**: `/frontend/admin-dashboard.js` (647 lines)
- **Role in Architecture**: Command dashboard controller. Manages executive overview metrics, communication log feeds, and the optimization view.
- **Key Functions**:
  - `loadAdminDash()`: Fetches `/api/admin/dashboard` aggregating mine-wide statistics.
  - `renderOptimizationPage(data)`: Renders Dijkstra evacuation route paths, ventilation fan recommendations, and fatigue score distribution cards.
  - `handleContactLogUpdate(payload)`: Appends incoming voice/SMS dispatch events to the executive comms log table.

---

### `frontend/utils/api.js`
- **File Path**: `/frontend/utils/api.js` (196 lines)
- **Role in Architecture**: Network transport abstraction layer for the browser client.
- **Capabilities**:
  - Dynamically resolves backend origins (`resolveLocalBackendOrigin`), automatically switching between port 3001 and local development servers (Vite, Live Server).
  - Maintains `SafeMineAuth` helper reading JWT bearer tokens from `localStorage` under key `safemine_supabase_session`.
  - Establishes full-duplex WebSocket connections (`wsConnect`) with automated 5-second backoff reconnection logic.

---

### `frontend/bot/chatbot.js`
- **File Path**: `/frontend/bot/chatbot.js` (356 lines)
- **Role in Architecture**: Floating conversational safety copilot.
- **Mechanism**:
  - Self-contained IIFE injecting a floating chat bubble and modal panel into the DOM.
  - Dispatches queries to `POST /api/chat`.
  - Renders markdown formatting, code blocks, quick-prompt action chips ("Evacuation Status", "Zone C Gas", "MSHA Ventilation Limits"), and typing indicators.

---

### `frontend/data/mockData.js` (Codebase Anomaly Audit)
- **File Path**: `/frontend/data/mockData.js` (1,513 lines)
- **Codebase Anomaly**:
  - Despite the `.js` file extension, this file contains **1,513 lines of pure CSS code** (`/* MineGuard Pro — PREMIUM UI v3.0 ... */`).
  - An earlier developer appears to have backed up or renamed a stylesheet into the `data/` folder.
  - It is completely unreferenced by `index.html` (comment on line 1332 explicitly confirms all live data flows through `api.js`).

---

### Unlinked Assets Audit
1. **`frontend/tour.js` & `tour.css`**: Intro.js onboarding walkthrough tour implemented but not linked in `index.html`.
2. **`frontend/admin-dashboard-fixes.css`**: Contains layout overflow and styling fixes for the command dashboard, but is not linked in `index.html`.
