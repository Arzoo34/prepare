# SafeMine: Ground-Up Rebuild & Implementation Roadmap

---

## 1. Overview & Learning Strategy

This roadmap provides a phased, step-by-step engineering curriculum enabling you to rebuild the SafeMine platform independently from scratch without copy-pasting code.

```
+---------------------------------------------------------------------------------------------------+
|                                 8-STAGE IMPLEMENTATION ROADMAP                                    |
+---------------------------------------------------------------------------------------------------+
| Stage 1: Hardware Prototyping & Sensor Acquisition (ESP32, I2C, ADC, Kinematics)                  |
|       |                                                                                           |
| Stage 2: Telemetry Ingestion & Real-Time Transport (Node.js, Express, Aedes MQTT, WebSockets)     |
|       |                                                                                           |
| Stage 3: Relational Persistence & State Machine (Database Schema, Normalization, WASM SQLite)    |
|       |                                                                                           |
| Stage 4: Safety Math & Physics Modeling (Oxygen Depletion, Dalton Displacement, Sensor Fusion)    |
|       |                                                                                           |
| Stage 5: Graph Theory & Optimization (Dynamic Dijkstra Pathfinding, Ventilation, Fatigue Index)   |
|       |                                                                                           |
| Stage 6: Command Interface & Dual-Mapping (HTML5 Canvas 2D Engine, Leaflet GPS, Web Audio API)    |
|       |                                                                                           |
| Stage 7: Automated Telephony & Cloud AI (Twilio TwiML Voice/SMS, Groq Llama 3 Prompt Injection)  |
|       |                                                                                           |
| Stage 8: Hardening, Security & Automated Testing (JWT Gate, Rate Limiting, Jest Test Suite)       |
+---------------------------------------------------------------------------------------------------+
```

---

## Stage 1: Hardware Prototyping & Sensor Acquisition

### 1. What You Need to Learn:
- ESP32 hardware architecture: Dual-core Xtensa 32-bit LX6, pinouts, ADC1 vs ADC2 restrictions when Wi-Fi is active.
- I2C bus arbitration: SDA, SCL lines, pull-up resistors, device addresses (`0x68` for MPU-6050, `0x77` for BMP180).
- Analog sampling: Volts-to-count mapping ($0\text{--}3.3\text{V} \to 0\text{--}4095$ for 12-bit ADC).
- Kinematic physics: Converting raw accelerometer LSB counts to gravitational units ($g$).

### 2. What to Implement:
- Write an Arduino/C++ sketch initializing GPIO pins, I2C bus, and serial communication at 115200 baud.
- Sample MQ-2 on analog GPIO 34, DHT11 on GPIO 4, BMP180 and MPU-6050 over I2C.
- Implement fall detection state machine: shock threshold ($>3.0\text{g}$) followed by prone tilt ($z < 0.3\text{g}$) and immobility.
- Implement on-device buzzer alarm on GPIO 25.
- Package readings into a clean JSON string and send via `HTTPClient.POST()` every 3000ms.

### 3. Files to Create:
- `firmware/smart_helmet.ino`

### 4. Common Mistakes to Avoid:
- **ADC2 Wi-Fi Conflict**: Trying to use ADC2 pins (GPIO 0, 2, 4, 12-15, 25-27) for analog input while Wi-Fi is active will fail because the Wi-Fi driver claims ADC2. Always use ADC1 pins (GPIO 32–39) for analog gas sensors.
- **Blocking Reconnect Loops**: Using `while (WiFi.status() != WL_CONNECTED)` with `delay()` freezes the main thread, disabling local fall detection during network dropouts.

### 5. How to Test It:
- Open Arduino Serial Monitor at 115200 baud. Shake the board to verify shock detection, tilt it horizontally to verify prone detection, and ensure JSON strings output every 3 seconds.

---

## Stage 2: Telemetry Ingestion & Real-Time Transport

### 1. What You Need to Learn:
- Node.js event-driven architecture, non-blocking I/O, and HTTP module.
- Express routing, body parsers, and rate limiting.
- The MQTT publish-subscribe pattern, broker mechanics, topics, and QoS levels.
- WebSocket RFC 6455 framing and connection lifecycles.

### 2. What to Implement:
- Initialize Node project with `express`, `ws`, `aedes`, `net`, and `cors`.
- Bind Express and `ws` to a shared HTTP server instance on port 3001.
- Instantiate an embedded Aedes MQTT broker on port 1883.
- Expose `POST /api/telemetry` for HTTP hardware nodes and subscribe to `mineguard/helmet/#` for MQTT nodes.
- Build an in-memory `broadcast(type, payload)` helper iterating active WebSocket clients.

### 3. Files to Create:
- `backend/server.js` (core HTTP/WS server)
- `backend/broker.js` (isolated MQTT broker module)

### 4. Common Mistakes to Avoid:
- **Port Conflicts (`EADDRINUSE`)**: Forgetting to attach error event handlers to the HTTP server instance causes opaque unhandled exceptions if the port is already open.
- **Missing WebSocket Clean-up**: Failing to verify `client.readyState === WebSocket.OPEN` before calling `send()` will throw runtime exceptions when clients disconnect abruptly.

### 5. How to Test It:
- Use `curl -X POST http://localhost:3001/api/telemetry -H "Content-Type: application/json" -d '{"worker_id":"TEST-1","ch4":0.5}'` and verify a connected WebSocket test script logs the received packet.

---

## Stage 3: Relational Persistence & State Management

### 1. What You Need to Learn:
- Relational schema design: State tables (`workers`) vs Time-Series logs (`telemetry`).
- SQLite parameter binding and SQL syntax (`ON CONFLICT DO UPDATE`).
- WebAssembly execution mechanics and memory buffers.

### 2. What to Implement:
- Integrate `sql.js` (or native `better-sqlite3`).
- Define relational DDL schemas: `workers`, `telemetry`, `alerts`, `devices`, `anchors`, `geofences`.
- Build a query wrapper supporting parameterized execution (`all`, `get`, `run`).
- Implement an automated seeder script populating 12 demo miners, 10 tunnel anchors, and baseline geofences.

### 3. Files to Create:
- `backend/db.js`
- `backend/schema.sql`
- `backend/seed.js`

### 4. Common Mistakes to Avoid:
- **Synchronous Write Blocking**: Calling synchronous file export (`fs.writeFileSync`) on every incoming telemetry packet saturates disk I/O. Implement debounced asynchronous flushes.
- **SQL Injection**: Never concatenate raw strings into SQL queries. Always use positional parameters (`?`).

### 5. How to Test It:
- Run `node backend/seed.js`. Inspect the generated SQLite file using SQLite CLI or DB Browser for SQLite to verify tables and data records.

---

## Stage 4: Safety Math & Physics Modeling

### 1. What You Need to Learn:
- Dalton's Law of Partial Pressures and atmospheric gas displacement.
- Human respiration rates in confined subterranean environments.
- Non-linear scoring functions (logarithmic scaling, exponential penalties).
- Multi-sensor fusion heuristics.

### 2. What to Implement:
- Build `o2Calculation.js`: calculate dynamic oxygen based on worker headcount and methane displacement.
- Build multi-criteria risk scoring: evaluate flammable gas, O2 deficiency, heart rate, motion, and compound multipliers.
- Build sensor fault detection: identify stuck sensors (consecutive zero/max readings) and sudden unrealistic jumps.
- Build multi-sensor fusion: assert fire signature ($\text{Temp} \ge 38^\circ\text{C} + \text{CO} \ge 40\text{ ppm}$) and incapacitated worker ($\text{O}_2 < 19.5\% + \text{motion}=0$).

### 3. Files to Create:
- `backend/o2Calculation.js`
- `backend/sensorFusion.js`

### 4. Common Mistakes to Avoid:
- **Floating Point Edge Cases**: Failing to clamp inputs (`NaN`, negative counts) causes arithmetic contamination across scoring engines.

### 5. How to Test It:
- Create unit test scripts validating that extreme gas inputs ($10\%\text{ CH}_4$) clamp properly to safe upper and lower bounds.

---

## Stage 5: Graph Theory & Optimization

### 1. What You Need to Learn:
- Graph theory representation: Adjacency lists vs Adjacency matrices.
- Dijkstra's algorithm for single-source shortest paths on weighted graphs.
- Dynamic edge weighting: calculating geometric intersections between line segments and circular geofences.
- Greedy resource allocation algorithms.

### 2. What to Implement:
- Model the underground mine tunnel topology as an undirected graph (`TUNNEL_GRAPH` and `TUNNEL_EDGES`).
- Implement `dijkstraEvacRoute(x, y)`:
  - Connect worker coordinates to nearest junction.
  - Calculate dynamic penalties: $+500$ for edges overlapping hazard geofences, $+15$ per congested worker.
  - Traverse to the lowest-cost exit portal or refuge chamber.
  - Reconstruct the optimal waypoint path.
- Implement `optimizeVentilation()`: compute zone gas load and assign proportional fan speeds ($20\%\text{--}100\%$).
- Implement `predictFatigue()`: calculate fatigue index from battery drain (shift duration proxy) and heart rate.

### 3. Files to Create:
- `backend/optimizer.js`

### 4. Common Mistakes to Avoid:
- **Graph Partitioning**: Removing hazardous edges completely can disconnect the graph and return "No route found". Always penalize with high weights rather than deleting edges.

### 5. How to Test It:
- Place a worker in Zone C with a hazard geofence active on the primary shaft. Assert that the algorithm selects the alternate route to `REFUGE-1`.

---

## Stage 6: Command Interface & Dual-Mapping

### 1. What You Need to Learn:
- HTML5 Canvas 2D Context API: path drawing, arc rendering, affine transformations, immediate-mode rendering.
- Leaflet.js library: GeoJSON layers, tile providers, custom SVG marker pins.
- Web Audio API: `AudioContext`, oscillators, gain nodes for programmatic sound generation.
- Client WebSocket management: connection handshakes and automatic backoff reconnection.

### 2. What to Implement:
- Create `index.html` with a left navigation sidebar, top status marquee, and multi-view page sections.
- Create `frontend/app.js`:
  - Establish persistent WebSocket client with auto-reconnection.
  - Build the HTML5 Canvas 2D subterranean renderer: draw tunnel paths, depth gradients, animated miner dots, and pulsing emergency rings.
  - Build the Leaflet surface GPS map: render real-world coordinates and geofences.
  - Implement programmatic siren sounds using Web Audio API oscillators.

### 3. Files to Create:
- `index.html`
- `frontend/app.js`
- `frontend/styles.css`
- `frontend/utils/api.js`

### 4. Common Mistakes to Avoid:
- **DOM Reflow Saturation**: Rebuilding DOM tables on every WebSocket packet causes UI stuttering. Update text nodes in place or batch renders using `requestAnimationFrame`.

### 5. How to Test It:
- Open `http://localhost:3001` in two separate browser windows. Trigger an event in one and verify the other updates instantly over WebSockets.

---

## Stage 7: Automated Telephony & Cloud AI

### 1. What You Need to Learn:
- CPaaS telecom architecture (Twilio REST API, TwiML voice XML).
- Prompt engineering for LLMs: system instructions, few-shot examples, dynamic context injection.
- Groq Cloud API: ultra-fast LPU inference endpoints.

### 2. What to Implement:
- Build `backend/emergency-contact.js`: format outbound SMS and TwiML voice call requests with fallback to simulated log arrays.
- Build `POST /api/chat`: query active database state (peak gas, active alerts, high-risk miners), inject into system prompt, and stream response from Groq Llama 3.3 70B.
- Implement floating chatbot widget in `frontend/bot/chatbot.js`.

### 3. Files to Create:
- `backend/emergency-contact.js`
- `frontend/bot/chatbot.js`

### 4. Common Mistakes to Avoid:
- **Prompt Injection & Bloat**: Injecting thousands of historical telemetry records exceeds LLM context windows and increases latency. Only inject high-level summaries (peak gas, active emergencies, danger geofences).

### 5. How to Test It:
- Send a test prompt to `/api/chat` asking: *"What is the status of Zone C?"* Verify that the LLM response accurately cites live database gas readings.

---

## Stage 8: Hardening, Security & Automated Testing

### 1. What You Need to Learn:
- Token-based authentication: JSON Web Tokens (JWT), claims, expiration, signing secrets.
- Password hashing: Cryptographic salts and bcrypt rounds.
- Automated testing: Unit testing with Jest, integration testing with Supertest.
- Path traversal protection in web servers.

### 2. What to Implement:
- Mount `backend/auth.js` in `server.js`: expose `/api/auth/register`, `/api/auth/login`, and wrap protected routes with `createAuthApiGate()`.
- Fix static directory exposure: serve only `frontend/`, never the parent project root.
- Implement Jest unit test suites for `o2Calculation.js` and `optimizer.js`.
- Implement Supertest integration tests for `/api/telemetry` and `/api/workers`.

### 3. Files to Create:
- `backend/auth.js` (refactored & mounted)
- `tests/o2Calculation.test.js`
- `tests/optimizer.test.js`
- `tests/api.test.js`

### 4. How to Test It:
- Execute `npm test`. Verify all test assertions pass and verify that accessing `/backend/mineguard.db` in a browser returns a 404/403.
