# SafeMine: End-to-End Operational Execution Flows

---

## 1. Overview of Execution Flows

Understanding SafeMine requires tracing data flow across distributed boundaries: from physical sensor pins and microcontroller registers, through network sockets and event loops, across database tables and mathematical optimization algorithms, and finally onto browser canvas pixels and cellular telephony voice channels.

This chapter provides **step-by-step numbered execution traces** for the system's 7 primary operational workflows.

---

## 2. Operational Trace 1: System Boot & Client Synchronization

```
+---------------------------------------------------------------------------------------------------+
| BOOT SEQUENCE                                                                                     |
| 1. Node Runtime Init -> 2. WASM SQLite Load -> 3. Schema Migrate -> 4. HTTP & WSS Bind ->         |
| 5. Mode Select (Dummy / MQTT) -> 6. Client Connect -> 7. Snapshot Push -> 8. Map Initialization  |
+---------------------------------------------------------------------------------------------------+
```

### Detailed Numbered Step-by-Step Flow:
1. **Node Process Launch (`package.json` $\to$ `server.js`)**:
   - Operator executes `npm start` (or `npm run dev` with nodemon).
   - Node.js runtime boots, loads environment variables via `dotenv` from `backend/.env`.
2. **Database Initialization (`db.js`, lines 248–298)**:
   - `initSqlJs()` loads the WebAssembly binary asynchronously.
   - It checks if `backend/mineguard.db` exists on disk. If found, reads raw binary buffer into memory; if not, instantiates an empty database.
   - Executes the DDL `SCHEMA` script, ensuring all 12 relational tables exist (`workers`, `telemetry`, `alerts`, etc.).
   - Executes non-destructive `ALTER TABLE` statements to ensure backward compatibility for columns (`wristband_battery`, `flammable_gas`, `lat`, `lng`).
   - Resolves the `db.ready` Promise, unblocking the server bootloader.
3. **HTTP & WebSocket Server Binding (`server.js`, lines 43–66, 1941–1953)**:
   - Express server attaches to HTTP port 3001 (or `$PORT`).
   - An `EADDRINUSE` error handler monitors for port conflicts, providing clear terminal troubleshooting commands (`netstat`, `taskkill`) if another instance is running.
   - `WebSocket.Server({ server })` attaches to the same HTTP server instance, listening for upgrade handshakes on `ws://localhost:3001`.
4. **Broker & Ingestion Engine Mode Selection (lines 1954–1975)**:
   - Inspects `process.env.DUMMY_MODE`:
     - If `true` (default): Starts `startDummyEngine()`, initiating internal timers for 12 simulated miners (every $3000\text{ms}$) and 10 anchors (every $15000\text{ms}$).
     - If `false`: Boots the embedded Aedes MQTT broker on port 1883, connects the internal MQTT subscriber client, and launches `startSimulator(1883)`.
5. **Browser Client Connection (`index.html` $\to$ `api.js` $\to$ `app.js`)**:
   - Operator opens `http://localhost:3001`.
   - Browser executes `frontend/utils/api.js`:
     - `resolveLocalBackendOrigin()` detects the host and configures `API_BASE` (`http://localhost:3001/api`) and `WS_BASE` (`ws://localhost:3001`).
     - `wsConnect()` initiates an HTTP GET request with `Upgrade: websocket` headers.
6. **Handshake & Initial Snapshot Push (`server.js`, lines 178–184)**:
   - Server `wss.on('connection')` fires.
   - Server queries `SELECT * FROM workers` from SQLite.
   - Server pushes an initial WebSocket message: `{ type: 'SNAPSHOT', payload: { workers }, ts: ... }`.
   - Browser `api.js` fires `ws.onmessage`, dispatching the snapshot to `app.js`.
7. **Client HUD Rendering (`app.js`, lines 67–100)**:
   - Browser populates state array `state.workers = snapshot.workers`.
   - `renderWorkerTable()` populates the dashboard DOM.
   - `drawMap()` initializes the HTML5 2D Canvas context, drawing tunnel geometries and positioning miners at their subterranean coordinates.

---

## 3. Operational Trace 2: Live Telemetry Ingestion, Clamping & Broadcasting

```
[ESP32 Hardware] ---> (HTTP POST /api/telemetry) ---> [server.js Express Handler]
                                                               |
    +----------------------------------------------------------+
    v
1. realHardwareWorkers.set(workerId, now)   <-- Exempt from simulator
2. Normalize field names (ch4, co, o2, hr)
3. Input Sanitization & Clamping (0 <= ch4 <= 5.0)
4. Sensor Health Flatline/Jump Check
5. Calculate Dynamic O2 (o2Calculation.js)
6. Calculate Risk Score (calcRisk: 0 - 10)
7. Upsert to 'workers' & Insert to 'telemetry'
8. fs.writeFileSync(mineguard.db)           <-- Disk Sync
9. wss.broadcast('WORKER_UPDATE', worker)
    |
    v
[Browser WebSocket Client] ---> Canvas Repaint & DOM Table Update
```

### Detailed Numbered Step-by-Step Flow:
1. **Hardware Transmission (`ESP32.ino`, lines 319–327)**:
   - ESP32 hardware finishes sampling MQ-2, DHT11, BMP180, and MPU-6050.
   - Serializes JSON payload and initiates an HTTP POST to `http://192.168.50.12:3001/api/telemetry`.
2. **Server Route Ingestion (`server.js`, lines 972–988)**:
   - Express rate limiter verifies IP has not exceeded 300 requests/minute.
   - Route resolves `workerId = d.worker_id || d.device_id || 'MNR-001'`.
   - **Hardware Exemption Check**: Stamps `realHardwareWorkers.set(workerId, Date.now())`. This informs the dummy engine that a physical device is active, suppressing simulated data overwrite for the next 10 seconds.
3. **Packet Normalization (lines 990–1020)**:
   - Handles multi-source field aliasing: maps `d.methane` / `d.mq4` to `ch4`, `d.bpm` / `d.pulse` to `heart_rate`, and `d.vbat` to `battery`.
4. **Input Sanitization & Range Clamping (`handleHelmetUpdate`, lines 265–351)**:
   - Validates that values fall within acceptable physical bounds:
     - If $\text{CH}_4 < 0$ or $> 5.0\%$, flags a `critical` sensor failure alert and clamps value to the previous valid reading.
     - If $\text{Temp} < -20^\circ\text{C}$ or $> 80^\circ\text{C}$, flags a temperature fault and clamps value.
5. **Sensor Health & Drift Evaluation (lines 291–302, 352–417)**:
   - Increments `ch4ZeroStreak` if $\text{CH}_4 = 0.0$. If streak reaches 5, flags `warning`; if 10, flags `critical` (stuck sensor).
   - Computes $\Delta\text{CH}_4 = |\text{CH}_{4,\text{current}} - \text{CH}_{4,\text{previous}}|$. If $\Delta\text{CH}_4 \ge 1.0\%$, increments `ch4JumpStreak`. If streak $\ge 2$, flags sensor instability.
6. **Dynamic Oxygen Calculation (lines 278–284)**:
   - If incoming payload does not supply a dedicated physical O2 sensor reading, queries current active worker headcount ($N$) from database.
   - Invokes `calculateOxygenLevel(N, ch4)` from `o2Calculation.js`:
     $$\text{O}_2 = 20.9 - \min(N \times 0.12, 2.5) - \min(\text{CH}_4 \times 1.2, 4.0)$$
   - Obtains safety classification label (`SAFE`, `WARNING`, `DANGER`) and UI color code.
7. **Multi-Variable Risk Scoring (`calcRisk`, lines 828–841)**:
   - Evaluates composite risk score on a scale of $0.0\text{--}10.0$:
     - Adds $+3.0$ if $\text{CH}_4 > 1.0\%$ (or $+1.5$ if $> 0.5\%$).
     - Adds $+2.5$ if $\text{CO} > 50\text{ ppm}$ (or $+1.0$ if $> 25\text{ ppm}$).
     - Adds $+2.0$ if $\text{O}_2 < 19.5\%$ (or $+1.5$ if $< 18.0\%$).
     - Adds $+4.0$ if panic button is pressed; $+3.0$ if fall detected.
     - Adds $+1.5$ if heart rate $> 120\text{ bpm}$ (tachycardia) or $< 45\text{ bpm}$ (bradycardia).
8. **Relational Database Persistence (`db.js`, lines 496–516)**:
   - Executes parameterized `INSERT INTO workers ... ON CONFLICT(id) DO UPDATE SET ...` to update real-time position, status, and telemetry.
   - Inserts immutable time-series record into `telemetry` table for historical audit logs.
   - Calls `saveDB()`: Serializes virtual memory buffer and writes to `mineguard.db` on disk.
9. **WebSocket Client Broadcast (lines 535–536)**:
   - Executes `broadcast('WORKER_UPDATE', worker)`.
   - Browser client receives event, updates local state, recalculates dashboard KPI aggregations, and triggers Canvas 2D frame re-render.

---

## 4. Operational Trace 3: Critical Gas Spike, Sensor Fusion & 3-Stage SOS Escalation

```
1. Gas Ingestion (CH4 = 1.2%, CO = 55 ppm in Zone C)
      |
      v
2. Sensor Fusion Detection (checkSensorFusion)
      --> Identifies Compound Hazard Pattern
      --> Inserts Critical Alert to DB
      --> WebSocket broadcast: 'NEW_ALERT'
      |
      v
3. Auto-Trigger Escalation (startEscalation, Stage 1: WARNING, 0-10s)
      --> Broadcast 'ESCALATION_STAGE' (countdown: 10s)
      --> Downlink to Helmet: Beep Warning Buzzer
      |
      v
4. Unacknowledged Timeout (10 seconds elapse)
      --> Stage 2: CRITICAL (10-30s)
      --> Inserts Escalated Alert to DB
      --> Downlink to Helmet: High Priority Buzzer
      |
      v
5. Unacknowledged Timeout (30 seconds total elapse)
      --> Stage 3: PANIC / EMERGENCY
      --> Worker status set to 'emergency', risk = 10
      --> notifyAdmin() triggers Twilio Voice Call & SMS
      --> Downlink to Helmet: Continuous Siren ("EVACUATE IMMEDIATELY")
```

### Detailed Numbered Step-by-Step Flow:
1. **Hazard Telemetry Arrives (`server.js`, line 852)**:
   - Miner `MNR-003` in Zone C registers $\text{CH}_4 = 1.20\%$ and $\text{CO} = 55\text{ ppm}$.
   - `checkSafetyThresholds(w)` asserts both readings exceed critical safety limits ($\text{CH}_{4,\text{crit}} = 1.0\%$, $\text{CO}_{\text{crit}} = 50\text{ ppm}$).
2. **Compound Sensor Fusion (`checkSensorFusion`, line 1545)**:
   - Detects concurrent flammable gas and toxic gas buildup:
     `((w.ch4 >= 0.5) && (w.co >= 25))` triggers `Pattern 1: Gas Accumulation`.
   - Generates a critical sensor fusion alert: `"💨 SENSOR FUSION: Gas Accumulation — Zone C. Compound hazard — possible ventilation failure."`
   - Checks alert deduplication cache (`_lastAlert`): if not reported within `ALERT_TTL_MS` (60s), writes alert to SQLite `alerts` table and broadcasts `NEW_ALERT` over WebSockets.
3. **Escalation Engine Initialization (`startEscalation`, line 1487)**:
   - Checks if worker has an active escalation timer in `escalationTimers['MNR-003']`.
   - **Stage 1 (Warning, $0\text{--}10\text{s}$)**:
     - Sets `stage = 'warning'`.
     - Broadcasts `ESCALATION_STAGE` event with `countdown: 10`.
     - Dispatches MQTT downlink to helmet: `{ type: 'early-warning', msg: 'CH4 detected (rising)', beep: true }`.
     - Operator dashboard flashes amber, and client audio synthesizer emits a low-frequency alert tone.
4. **Stage Transition to Critical ($10\text{s}$ mark, line 1495)**:
   - If neither operator nor worker has acknowledged the alert, `timer1` fires after $10,000\text{ms}$.
   - Sets `stage = 'critical'`.
   - Inserts `⚠️ ESCALATED CRITICAL — Amit Singh` alert into SQLite database.
   - Broadcasts `ESCALATION_STAGE` with `countdown: 20`.
   - Emits dual-tone siren on operator console via Web Audio API.
5. **Stage Transition to Emergency / Panic ($30\text{s}$ mark, line 1506)**:
   - After an additional $20,000\text{ms}$ (30 seconds total from onset), `timer2` fires.
   - Automatically mutates worker state in database:
     `UPDATE workers SET panic=1, status='emergency', risk=10 WHERE id='MNR-003'`.
   - Inserts `🚨 PANIC ESCALATION — Amit Singh (No response after 30s. IMMEDIATE RESCUE REQUIRED)`.
   - Broadcasts `ESCALATION_STAGE` with `countdown: 0`.
   - Sends aggressive MQTT downlink: `{ type: 'emergency', msg: 'EVACUATE IMMEDIATELY', beep: true, priority: 3 }`.
6. **Automated Telephony Notification (`notifyAdmin`, line 904 $\to$ `emergency-contact.js`)**:
   - `server.js` detects an `emergency` level alert and calls `notifyAdmin(alert, workers)`.
   - `emergency-contact.js` verifies `ADMIN_PHONE` is configured.
   - Checks throttling cache (`_lastNotify`): guarantees maximum 1 phone call per worker per 5 minutes.
   - Dispatches Twilio Voice API request: dials safety manager's mobile phone and speaks synthesized TwiML warning: `"Emergency alert from Mine Guard Pro. PANIC ESCALATION: Worker Amit Singh in Zone C requires immediate assistance."`
   - Concurrently dispatches an urgent SMS summary via Twilio Messaging API.

---

## 5. Operational Trace 4: Dynamic Dijkstra Evacuation Path Generation

```
1. Operator requests evacuation path for MNR-003 (GET /api/optimization/evacuation/MNR-003)
      |
      v
2. Retrieve worker coordinates (x=460, y=210) & all active geofences
      |
      v
3. Build In-Memory Adjacency List from TUNNEL_GRAPH & TUNNEL_EDGES
      |
      v
4. Map worker start node 'WORKER' to nearest tunnel junction (J-T3, dist: 60)
      |
      v
5. Dynamic Edge Weighting:
      - Base Euclidean distance (dist)
      - Intersects Hazard Geofence? Add +500 penalty
      - Other workers within 30 units? Add +15 per worker (congestion penalty)
      |
      v
6. Execute Dijkstra Priority Traversal (dist[WORKER] = 0, dist[v] = Infinity)
      |
      v
7. Evaluate Lowest-Cost Target among exits ('SURFACE') & refuges ('REFUGE-1', 'REFUGE-2')
      |
      v
8. Backtrack path via 'prev' map -> Construct ordered coordinate path array
      |
      v
9. Return JSON: { path: [...], target: 'REFUGE-1', dist: 280, estimatedTime: 336s, safe: true }
```

### Detailed Numbered Step-by-Step Flow:
1. **Route Request Inception**:
   - Operator clicks "Find Evac Route" in the UI, or the background optimizer flags a high-risk worker ($risk \ge 5.0$), querying `GET /api/optimization/evacuation/:workerId`.
2. **Graph Assembly (`optimizer.js`, lines 91–105)**:
   - Imports static topological definitions: `TUNNEL_GRAPH` (9 junctions and exits) and `TUNNEL_EDGES` (9 bidirectional tunnel segments).
   - Initializes an empty adjacency list: `adj[nodeId] = []`.
   - Appends a virtual origin node: `adj['WORKER'] = []`.
   - Calculates Euclidean distance from worker $(x, y)$ to all tunnel junctions; identifies `J-T3` as nearest node ($dist = 60$) and adds directed edge `WORKER -> J-T3`.
3. **Dynamic Edge Weight Calculation (lines 106–142)**:
   - For every edge in `TUNNEL_EDGES`:
     - **Hazard Avoidance Penalty**: Calculates midpoint $(mx, my)$ of the tunnel segment. Checks if distance from midpoint to any active hazard geofence center is less than geofence radius. If true, adds **$+500$ penalty** to edge weight.
     - **Worker Congestion Penalty**: Computes point-to-line-segment distance from all other miners in the mine to the tunnel edge. For every worker within 30 units, adds **$+15$ penalty** to model bottleneck delays during crowd evacuation.
     - Symmetrical insertion: adds augmented weight to both `adj[from]` and `adj[to]`.
4. **Dijkstra Traversal Execution (lines 143–164)**:
   - Initializes distance map: `dist['WORKER'] = 0`, all other nodes initialized to `Infinity`.
   - Initializes priority queue: `queue = [{ node: 'WORKER', d: 0 }]`.
   - Traversal loop:
     - Sorts queue by distance: `queue.sort((a, b) => a.d - b.d)`.
     - Extracts node with minimum distance: `node = queue.shift()`.
     - For each neighbor `to` connected by edge `weight`:
       - Calculates alternative distance: $alt = dist[node] + weight$.
       - If $alt < dist[to]$, updates $dist[to] = alt$, records predecessor pointer $prev[to] = node$, and pushes neighbor to queue.
5. **Target Evaluation & Path Reconstruction (lines 166–191)**:
   - Filters graph nodes where `type === 'exit'` (`SURFACE`) or `type === 'refuge'` (`REFUGE-1`, `REFUGE-2`).
   - Identifies candidate target with minimum cumulative cost: `REFUGE-1` ($dist = 280$).
   - Backtracks from target using `prev` map:
     `REFUGE-1` $\longleftarrow$ `J-T2` $\longleftarrow$ `J-T3` $\longleftarrow$ `WORKER`.
   - Reconstructs coordinate path array:
     `[{ id: 'J-T3', x: 400, y: 210 }, { id: 'J-T2', x: 280, y: 160 }, { id: 'REFUGE-1', x: 280, y: 160 }]`.
   - Computes estimated walking time at industrial evacuation pace ($50\text{ units/min}$):
     $$\text{EstimatedTime} = \frac{280}{50} \times 60 = 336\text{ seconds}$$
6. **Payload Delivery**:
   - Returns JSON response to client.
   - Client Canvas 2D engine renders an animated glowing green path from miner `MNR-003` to `REFUGE-1`, visually bypassing the contaminated Zone C methane pocket.

---

## 6. Operational Trace 5: Fallback Wristband Handover (Degraded Tracking Mode)

```
1. Miner removes helmet or helmet battery dies (No telemetry for > 15 seconds)
      |
      v
2. Background Health Check (checkDeviceHealth, runs every 3000ms)
      --> Detects (Date.now() - helmet.last_seen) > 15000ms
      --> Marks helmet_status = 'offline'
      |
      v
3. Wristband telemetry arrives (POST /api/wristband)
      --> { worker_id: "MNR-001", battery: 92, movement: true, sos: false, signal: -78 }
      |
      v
4. State Evaluation in handleWristbandUpdate()
      --> Helmet is offline, BUT wristband is online!
      --> Sets worker status = 'degraded_mode'
      --> Calculates wristband fallback risk score (calcWristbandRisk)
      |
      v
5. Database Upsert & WebSocket Broadcast
      --> Updates workers table: status='degraded_mode', wristband_status='online'
      --> Broadcasts 'DEVICE_STATUS_UPDATE' and 'WORKER_UPDATE'
      |
      v
6. UI Renders Amber Warning Halo & Status Indicator on Tactical HUD
```

### Detailed Numbered Step-by-Step Flow:
1. **Primary Device Failure**:
   - Miner `MNR-001` removes helmet during a break, or the helmet battery depletes to $0\%$.
   - HTTP packets cease arriving at `/api/telemetry`.
2. **Timeout Detection (`server.js`, lines 696–786)**:
   - Background monitor `checkDeviceHealth()` executes every $3000\text{ms}$.
   - Computes age of last packet: $ageMs = Date.now() - parseIsoMs(helmet.last\_seen)$.
   - When $ageMs > 15,000\text{ms}$ (`DEVICE_OFFLINE_SEC * 1000`), asserts helmet is offline.
   - Updates `devices` table: `status='offline'`.
3. **Secondary Wearable Ingestion (`handleWristbandUpdate`, lines 600–695)**:
   - Worker's fallback wristband transmits telemetry packet to `POST /api/wristband`:
     `{ worker_id: "MNR-001", battery: 90, movement: true, sos: false, signal: -75 }`.
   - Records wristband status as `online` in `devices` table.
4. **Degraded Mode Handover Logic (lines 678–684)**:
   - Inspects dual-device operational state:
     - `helmetOnline === false` AND `wristOnline === true`.
   - Verifies worker is not in active emergency.
   - Enters **Degraded Mode**:
     - Calculates wristband-only fallback risk score (`calcWristbandRisk`):
       - If no movement, adds $+3.0$.
       - If wristband battery $< 20\%$, adds $+2.0$.
       - If signal $RSSI < -90\text{ dBm}$, adds $+2.0$.
     - Executes SQL: `UPDATE workers SET status='degraded_mode', risk=? WHERE id='MNR-001'`.
5. **Client Notification**:
   - Broadcasts `DEVICE_STATUS_UPDATE` and `WORKER_UPDATE` over WebSockets.
   - Operator HUD shifts miner indicator from solid green (`online`) to striped amber (`degraded_mode`), indicating the miner is tracked via wristband motion and SOS fallback, but atmospheric gas sensing is offline.

---

## 7. Operational Trace 6: AI Safety Copilot Query with Context Injection

```
1. Operator types prompt in AI Assistant panel: "What is the status of Zone C?"
      |
      v
2. Client dispatches HTTP POST /api/chat with message history
      |
      v
3. Server executes Context Injection Assembly (server.js, lines 1285-1323)
      - Queries workers table: total, emergency, critical counts
      - Queries peak CH4, peak CO, minimum O2
      - Queries active geofence breaches (workers inside hazard zones)
      - Queries last 5 unresolved safety alerts
      |
      v
4. Format System Prompt:
      "You are SafeMine AI... LIVE MINE STATUS: Miners: 12 | Peak CH4: 1.20% |
       Active alerts: [CRITICAL] Methane Critical Zone C..."
      |
      v
5. Dispatch HTTP POST to Groq Cloud API (https://api.groq.com/openai/v1/chat/completions)
      - Model: llama-3.3-70b-versatile
      - Auth: Bearer GROQ_API_KEY
      |
      v
6. Groq LPU generates tokens at > 250 tokens/sec
      |
      v
7. Server returns JSON response: { content: [{ type: 'text', text: "..." }], provider: 'groq' }
      |
      v
8. Client chatbot.js parses Markdown and renders response in chat panel
```

### Detailed Numbered Step-by-Step Flow:
1. **User Interaction (`chatbot.js` or `#page-aiassist`)**:
   - Operator enters natural language query: *"What is the status of Zone C and what actions should we take?"*
   - Client sends POST request to `/api/chat` with `{ messages: [...] }`.
2. **Dynamic Context Assembly (`server.js`, lines 1285–1323)**:
   - Server intercepts request and queries live SQLite state:
     - `SELECT * FROM workers ORDER BY risk DESC`.
     - `SELECT * FROM alerts WHERE created_at > datetime('now', '-1 hour') LIMIT 10`.
     - `SELECT * FROM geofences WHERE active=1`.
   - Computes active aggregate metrics: total miners, emergency count, critical count, peak $\text{CH}_4$, peak $\text{CO}$, minimum $\text{O}_2$.
   - Computes geofence breach collisions: identifies miners whose current coordinates lie within hazard geofence radii.
3. **Prompt Composition**:
   - Combines safety guidelines (MSHA standards, Lower Explosive Limits, evacuation protocols) with live operational telemetry into a comprehensive `systemPrompt`:
     ```
     You are SafeMine AI, an expert underground mine safety assistant...
     LIVE MINE STATUS (2026-11-23T16:00:00.000Z):
     - Miners: 12 total | 1 emergency | 1 critical
     - Peak CH4: 1.20% | Peak CO: 55ppm | Min O2: 19.4%
     - MQTT: LIVE | Active alerts (1h): 3
     - GEOFENCE BREACHES: MNR-003 (Amit Singh)
       [CRITICAL] Methane Critical (CH4): MNR-003: 1.20% - EXPLOSIVE RISK in Zone C
     HIGH-RISK WORKERS:
       MNR-003 (Amit Singh): Risk 8.5/10, CH4 1.20%, CO 55ppm, Zone: Zone C, Status: emergency
     ```
4. **Groq Cloud API Execution (lines 1324–1350)**:
   - Dispatches HTTPS POST request to Groq’s LPU inference endpoint.
   - Passes model parameter `llama-3.3-70b-versatile`, system context, and recent conversation turns.
   - Groq’s tensor architecture processes the context and streams completion tokens in under $500\text{ms}$.
5. **Fallback Mechanism (lines 1352–1369)**:
   - If `GROQ_API_KEY` is missing or the Groq API returns a 5xx error, the server falls back to the Anthropic Claude API (`claude-sonnet-4-20250514`) if configured.
6. **Client Delivery & UI Rendering**:
   - Client receives JSON response.
   - `chatbot.js` renders formatted response, advising operator: *"Zone C is currently at explosive risk. Methane is at 1.20% and Amit Singh (MNR-003) is in emergency status. Immediate actions: 1. Verify evacuation order to REFUGE-1. 2. Boost auxiliary ventilation fan to 100%. 3. Restrict entry via geofence."*
