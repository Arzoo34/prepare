# SafeMine: Project Overview & Problem Domain

---

## 1. High-Level Explanation (Plain English)

### What Does SafeMine Do?
Imagine working hundreds of meters beneath the earth's surface in dark, narrow tunnels. You cannot rely on cellular networks, GPS satellites do not penetrate the solid rock, and the air around you can turn deadly in minutes due to invisible, odorless gases or sudden oxygen depletion. If you fall, pass out, or become trapped behind a cave-in, surface operators might not realize anything is wrong until hours later when you fail to return from your shift.

**SafeMine** is an intelligent safety ecosystem designed to protect subterranean miners. Every worker wears a smart helmet equipped with atmospheric and motion sensors. The helmet continuously tracks environmental conditions (toxic carbon monoxide, explosive methane, barometric pressure, temperature) and worker vitals (movement, impact, heart rate). 

This information is beamed to a surface command center in real time. If a dangerous gas buildup occurs, a worker collapses, or someone presses their emergency panic button:
1. The helmet sounds an audible buzzer immediately to alert the worker.
2. The surface dashboard sounds an alarm, flashes warning indicators, and pinpoints the miner’s exact underground tunnel location.
3. The system calculates the safest evacuation route out of the mine—guiding workers around gas pockets and congested corridors toward the nearest refuge chamber.
4. An automated emergency system calls and texts the mine safety manager’s phone via automated voice and SMS.
5. A built-in AI safety assistant analyzes current sensor trends, cross-references mining safety regulations, and advises operators on emergency procedures.

---

## 2. Technical Engineering Explanation

### System Definition
**SafeMine** is a distributed, event-driven, industrial cyber-physical monitoring and decision-support system. It combines edge sensing, wireless telemetry bridging, sensor fusion algorithms, and a centralized web-based operational command interface.

```
+---------------------------------------------------------------------------------------------------+
|                                 PHYSICAL TO DIGITAL TELEMETRY PATH                                |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|  [Underground Stope / Drift]                                                                      |
|  +------------------------+        HTTP POST / MQTT JSON       +-------------------------------+  |
|  | ESP32 Smart Helmet     | ---------------------------------> | Node.js Telemetry Ingestion   |  |
|  | (12-bit ADC, I2C Bus)  |   (Interval: 3000ms, WiFi/LoRa)   | Rate Limiter (300 req/min)    |  |
|  +------------------------+                                    +-------------------------------+  |
|                                                                                |                  |
|  [Surface Control Room]                                                        v                  |
|  +------------------------+            WebSocket Push          +-------------------------------+  |
|  | Web Tactical HUD       | <--------------------------------- | Multi-Variable Risk Scoring   |  |
|  | (Canvas 2D + Leaflet)  |        (JSON Broadcast Event)      | & Threshold Decision Engine   |  |
|  +------------------------+                                    +-------------------------------+  |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
```

---

## 3. The Real-World Problem It Solves

Underground mining remains one of the world's most perilous occupations. According to data from the **US Mine Safety and Health Administration (MSHA)** and the **International Council on Mining and Metals (ICMM)**, major underground hazards include:

1. **Atmospheric Toxicity and Asphyxiation**:
   - **Methane ($\text{CH}_4$)**: Lighter than air, accumulates in tunnel roof cavities. Highly explosive in concentrations between $5\%$ and $15\%$ by volume (Lower Explosive Limit, LEL to Upper Explosive Limit, UEL).
   - **Carbon Monoxide ($\text{CO}$)**: Byproduct of diesel engine exhaust, blasting, or low-temperature coal oxidation (spontaneous combustion). Odorless, toxic at $>50\text{ ppm}$, lethal within minutes at $>400\text{ ppm}$.
   - **Oxygen ($\text{O}_2$) Deficiency**: Heavy machinery consumption, timber rot, coal absorption, and worker respiration in dead-end headings without forced ventilation can drive $\text{O}_2$ below $19.5\%$ (hazardous) and below $16\%$ (loss of consciousness).
2. **RF Attenuation & Communication Blackouts**:
   - High-frequency RF signals (such as 4G/5G, Wi-Fi 2.4/5GHz, and GPS) experience severe attenuation when passing through high-permittivity rock strata and conductive ore bodies. Standard commercial mobile devices cannot maintain connectivity.
3. **Traumatic Physical Injury & Incapacitation**:
   - Rockfalls, pinch points from heavy machinery, slips, and heat stroke frequently incapacitate workers. If a solitary miner loses consciousness, they are unable to operate a radio or activate a manual distress call.
4. **Panic-Induced Disorientation During Evacuations**:
   - Power loss, smoke, and dust during underground mine emergencies impair visibility. Workers frequently attempt to escape through traditional pathways that may already be filled with toxic gas or obstructed by debris, leading to fatalities that could have been avoided.

---

## 4. Why This Problem Matters

- **Human Life & Safety Regulations**: Regulatory frameworks (e.g., the US **MINER Act of 2006**, EU Directive **92/104/EEC**, and UK **Coal Mines Regulations**) mandate continuous atmospheric monitoring, two-way wireless tracking, and certified emergency refuge chambers within designated travel distances.
- **Economic Liability**: Mine disasters result in massive operational shutdowns, regulatory fines, litigation costs, and equipment destruction reaching hundreds of millions of dollars.
- **Shift Efficiency**: Continuous monitoring of environmental air quality and worker fatigue levels prevents occupational illnesses and avoids unnecessary shift interruptions caused by false alarms.

---

## 5. Target Personas

| Persona | Operational Context | Primary Goals in SafeMine |
| :--- | :--- | :--- |
| **Underground Miner** | Working in drifts, stopes, and development headings. | Needs unobtrusive personal protection, instant local audio alerts (buzzer), fall detection, and an emergency manual SOS switch. |
| **Control Room Operator** | Managing surface command operations during 12-hour shifts. | Requires a unified single-pane-of-glass dashboard, live telemetry, early warning indicators, and audible threshold alarms. |
| **Mine Safety Manager** | Responsible for statutory compliance, safety audits, and emergency response. | Requires automated SMS/voice call escalations, incident audit logs, shift muster reports, and regulatory compliance data. |
| **Ventilation Engineer** | Overseeing main fans, auxiliary blowers, and ducting networks. | Requires real-time gas dispersion analytics, fan speed optimization metrics, and zone-specific airflow demand data. |
| **Maintenance Technician** | Managing sensor calibration, helmet charging, and hardware health. | Monitors sensor fault logs (drift, stuck readings, disconnected packets) and schedules equipment repairs. |

---

## 6. Core Features

```
                                  +-----------------------+
                                  |   SAFEMINE PLATFORM   |
                                  +-----------------------+
                                              |
      +-------------------+-------------------+-------------------+-------------------+
      |                   |                   |                   |                   |
      v                   v                   v                   v                   v
+------------+     +------------+      +------------+      +------------+      +------------+
| Telemetry  |     |   Safety   |      | Evacuation |      | Compliance |      | Artificial |
| Ingestion  |     |  Fusion &  |      | Path & Fan |      | & Muster   |      | Copilot &  |
| & Streaming|     | Escalation |      | Optimizer  |      | Reporting  |      | Telephony  |
+------------+     +------------+      +------------+      +------------+      +------------+
```

1. **Real-Time Telemetry Streaming**:
   - Sub-second distribution of worker vitals and atmospheric gases via WebSockets (`ws://`) and MQTT (`mqtt://`).
2. **On-Device & Server-Side Oxygen Depletion Engine**:
   - Replaces naive random walks with a respiration and gas displacement model:
     $$\text{O}_2 = 20.9\% - \min(N_{\text{workers}} \times 0.12, 2.5) - \min(\text{Gas}\% \times 1.2, 4.0) + \text{noise}$$
3. **Compound Sensor Fusion**:
   - Correlates multi-sensor telemetry to identify complex hazard patterns:
     - Gas Leak: $\text{CH}_4 \ge 0.5\% + \text{CO} \ge 25\text{ ppm}$
     - Fire Signature: $\text{Temp} \ge 38^\circ\text{C} + \text{CO} \ge 40\text{ ppm}$
     - Incapacitated Worker: $\text{O}_2 < 19.5\% + \text{Motion} = 0$
     - Explosive Distress: $\text{Panic Button} = 1 + \text{CH}_4 \ge 0.8\%$
4. **Three-Stage SOS Alert Escalation**:
   - Auto-escalates unacknowledged high-risk events:
     $$\text{Warning Stage (0–10s)} \longrightarrow \text{Critical Stage (10–30s)} \longrightarrow \text{Panic / Rescue Stage (30s+)}$$
5. **Dijkstra Dynamic Evacuation Pathfinding**:
   - Computes hazard-avoidance escape routes using a graph of mine junctions, dynamically adding a $+500$ penalty for hazard geofence crossings and $+15$ per congested worker.
6. **Multi-Objective Ventilation & Headcount Optimizer**:
   - Greedy zone ventilation prioritizing fan speed ($20\%\text{--}100\%$) based on aggregate toxic/flammable gas load.
   - Headcount balancer redistributing personnel away from overcrowded stopes ($>5$ workers).
7. **Predictive Fatigue Modeling**:
   - Combines battery depletion (shift duration proxy: $8\text{h} \times (100 - \text{battery})/100$), elevated heart rate, immobility, and ambient risk into a $0\text{--}10$ fatigue index.
8. **Dual-Mapping HUD**:
   - Bespoke HTML5 Canvas 2D engine rendering underground tunnel vectors, depth gradients, and pulsing hazard halos.
   - Leaflet.js surface map rendering georeferenced GPS positions with custom SVG marker pins.
9. **Automated Emergency Telephony**:
   - Automated outbound voice calls with speech synthesis (TwiML) and SMS dispatch to the Mine Safety Manager via Twilio REST APIs.
10. **Generative AI Safety Copilot**:
    - Embedded conversational assistant powered by **Groq Llama 3.3 70B** injected with live underground context.

---

## 7. Functional Requirements

| Req ID | Functional Requirement | Implementation Mechanism |
| :--- | :--- | :--- |
| **FR-01** | The system must ingest sensor packets from physical helmets at intervals of $\le 3000\text{ms}$. | HTTP POST `/api/telemetry` and MQTT topic `mineguard/helmet/:id`. |
| **FR-02** | The system must distinguish between physical hardware and internal simulation. | `realHardwareWorkers` Map tracks active hardware IDs; suppresses dummy engine mutation for 10 seconds. |
| **FR-03** | The system must detect falls using accelerometer vector magnitude and tilt. | MPU-6050 vector magnitude $\sqrt{x^2+y^2+z^2} > 3.0\text{g}$, followed by tilt $z < 0.3\text{g}$ and zero motion. |
| **FR-04** | The system must calculate dynamic evacuation paths avoiding active danger zones. | `dijkstraEvacRoute()` in `optimizer.js` traversing `TUNNEL_GRAPH`. |
| **FR-05** | The system must support degraded tracking mode if helmet telemetry ceases. | Wristband fallback ingestion via `/api/wristband`, setting status to `degraded_mode`. |
| **FR-06** | The system must trigger audible and visual alerts when thresholds are crossed. | Browser Web Audio API oscillator synthesis, Notification API, and WebSocket `NEW_ALERT` broadcasts. |
| **FR-07** | The system must maintain an audit log of all safety alerts, sensor faults, and incidents. | Stored in SQLite database tables `alerts`, `sensor_faults`, and `incidents`. |

---

## 8. Non-Functional Requirements (NFRs)

- **Latency**: End-to-end telemetry latency from packet ingestion to browser canvas render must be under $200\text{ms}$.
- **Cross-Platform Portability**: The backend must run on standard enterprise workstations without requiring platform-native C++ build tools (achieved via WebAssembly `sql.js`).
- **Resilience / Degraded Operation**: The loss of external cloud connectivity (e.g., Groq AI API or Twilio API) must not impair local monitoring, sensor fusion, or audio alarm generation.
- **Usability & Ergonomics**: The frontend interface must adhere to industrial HUD design standards: high-contrast dark theme ($<10\%$ luminance), legible monospace typography for numbers, and color-blind safe status indicators.

---

## 9. Constraints and Assumptions

### Constraints
1. **Zero Native Toolchain Constraint**: The system was built using `sql.js` (WebAssembly) rather than `better-sqlite3` to avoid requiring node-gyp, Python, and Visual Studio C++ compilers on host machines.
2. **Single-Process Monolith**: Ingestion, MQTT brokering, WebSocket distribution, REST APIs, and background optimization execute inside a single Node.js runtime process.
3. **Simulation Fallback**: If external LoRa gateways or ESP32 hardware nodes are absent, the system defaults to an internal drift engine simulating 12 workers and 10 anchors.

### Assumptions
1. The underground mine has wireless connectivity (WiFi APs or LoRaWAN gateways) deployed at major tunnel junctions.
2. In-tunnel location coordinates ($x, y, \text{depth}$) are derived from fixed anchor nodes using RSSI fingerprinting or UWB time-of-flight before reaching the backend.
3. The server system clock is synchronized via NTP (Network Time Protocol) to ensure consistent telemetry timestamps.

---

## 10. Scope Boundaries: Implemented vs. Mocked vs. Future

| Feature Domain | Production-Implemented in Code | Mocked / Simulated | Future Roadmap Requirement |
| :--- | :--- | :--- | :--- |
| **Telemetry Ingestion** | Full Express REST & MQTT handlers | Internal random walk engine | Industrial OPC UA / Modbus TCP connector |
| **Hardware Firmware** | C++ Arduino sketch for ESP32 | `esp32_simulator.js` node script | Over-The-Air (OTA) firmware update support |
| **Pathfinding** | Full Dijkstra graph implementation | Static 9-node tunnel layout | Dynamic 3D CAD / DXF mine wireframe import |
| **Telephony** | Full Twilio REST client & TwiML | Simulated log array fallback | Satellite radio / VHF emergency paging |
| **Drone Monitoring** | Database schema, REST CRUD | No physical drone autopilot API | PX4 / ROS 2 autonomous drone bridge |
| **Authentication** | Schema, bcrypt, JWT, and Supabase gate | Authentication gate **unmounted** in `server.js` | OAuth2 / Active Directory SSO integration |

---

## 11. What Makes SafeMine Different from Simpler Alternatives?

```
+---------------------------+-----------------------------------+-----------------------------------+
| Capability                | Basic Mining Telemetry System     | SafeMine Platform                 |
+---------------------------+-----------------------------------+-----------------------------------+
| Gas Level Tracking        | Static scalar threshold checking  | Respiration & gas displacement    |
|                           | (e.g. alert if CH4 > 1%)          | physics calculation model         |
+---------------------------+-----------------------------------+-----------------------------------+
| Alert Processing          | Independent isolated alerts       | Compound multi-sensor fusion      |
|                           | (causes notification fatigue)     | + 3-stage temporal escalation     |
+---------------------------+-----------------------------------+-----------------------------------+
| Evacuation Management     | Static paper maps or exit signs   | Real-time dynamic Dijkstra        |
|                           |                                   | routing avoiding active hazards   |
+---------------------------+-----------------------------------+-----------------------------------+
| Device Redundancy         | Single device (helmet only)       | Dual-tier: Smart Helmet +         |
|                           |                                   | fallback Wearable Wristband       |
+---------------------------+-----------------------------------+-----------------------------------+
| Decision Support          | Raw numeric dashboard data        | Context-aware Groq Llama 3 AI     |
|                           |                                   | assistant + Twilio voice alerts   |
+---------------------------+-----------------------------------+-----------------------------------+
```

---

## 12. Limitations of Current Implementation

1. **Synchronous File I/O on Database Writes**:
   - `db.js` uses `sql.js` (WebAssembly). Because WebAssembly memory is volatile, every database insert or update triggers `fs.writeFileSync(DB_PATH, Buffer.from(_sqlDb.export()))`. At scale, disk write contention will block the Node.js event loop.
2. **Monolithic Code Organization**:
   - `backend/server.js` contains 1,982 lines, mixing route definitions, WebSocket broadcasts, sensor threshold logic, MQTT subscriptions, and optimization interval loops.
3. **Unmounted Authentication Layer**:
   - While `auth.js` contains a robust JWT generation, password hashing, and Supabase validation implementation, it is never mounted onto the Express application in `server.js`.
4. **2D Representation of 3D Tunnels**:
   - The underground mine schematic renders on a 2D Canvas layout ($900 \times 520$). Although a numeric `depth` attribute exists in data records, the visual and pathfinding representations do not calculate true vertical incline/decline shaft traversal costs.
