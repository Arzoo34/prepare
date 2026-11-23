# SafeMine: Technology Stack Deep Dive

---

## 1. Technology Matrix Overview

SafeMine spans physical hardware, edge embedded firmware, network transport protocols, application servers, persistence engines, analytical optimization suites, and client visualization engines.

```
+---------------------------------------------------------------------------------------------------+
|                                     TECHNOLOGY MATRIX BY TIER                                     |
+---------------------+-------------------------------+---------------------------------------------+
| Tier                | Primary Technology / Library  | Architectural Function                      |
+---------------------+-------------------------------+---------------------------------------------+
| Edge Hardware       | ESP32 Dev Module (C++/Arduino)| Sensor acquisition, buzzer alarm actuation  |
| Edge Sensors        | MQ-2, MPU-6050, DHT11, BMP180 | Gas, motion/shock, temp/humidity, baro      |
| Telemetry Transport | HTTP REST / MQTT (Aedes)      | Bidirectional telemetry & downlink control  |
| Server Runtime      | Node.js (v18+) & Express v4   | Event-driven application monolith           |
| Live Streaming      | WebSockets (ws v8)            | Low-latency client push broadcasting        |
| Persistence         | sql.js (WASM SQLite v1.11)    | In-memory database with disk sync           |
| Intelligence        | Groq API (Llama 3.3 70B)      | Context-aware AI safety assistant           |
| Telephony           | Twilio REST API & TwiML       | Automated voice phone calls & SMS dispatch  |
| Frontend Client     | Vanilla JavaScript (ES6+)     | Reactive DOM reconciliation, audio synth    |
| Visualization       | HTML5 Canvas 2D & Leaflet.js  | Subterranean tunnel schematic & surface GPS |
+---------------------+-------------------------------+---------------------------------------------+
```

---

## 2. Server Runtime & Ingestion Tier

### Node.js (v18+ LTS)
- **What It Is**: An asynchronous, event-driven JavaScript runtime built on Chrome's V8 engine and libuv.
- **Why Used in SafeMine**: SafeMine is an I/O-intensive application. It continuously receives telemetry from microcontrollers, writes records to a database, broadcasts updates over WebSockets, and handles outbound HTTP requests to Twilio and Groq. Node's non-blocking event loop handles high-concurrency I/O with minimal memory footprint compared to multi-threaded thread-per-request architectures (e.g. Apache/Tomcat).
- **Internal Mechanics**:
  - **Libuv Event Loop**: Manages phases: Timers (`setInterval` for optimizer/dummy engine), I/O Poll (incoming HTTP/MQTT/WebSocket sockets), Check (`setImmediate`), and Close callbacks.
  - **Single Threaded Execution**: All JavaScript execution in `server.js` runs on a single main thread. Long-running CPU computations (like complex Dijkstra iterations over thousands of nodes) can block the thread, delaying telemetry ingestion.
- **Trade-offs**: Single-point-of-failure; an uncaught synchronous exception (`throw new Error`) crashes the entire server process unless handled by process supervisors (e.g., PM2, systemd).

---

### Express.js (v4.19.2)
- **What It Is**: A minimalist, unopinionated web framework for Node.js providing routing, middleware chaining, and HTTP request/response abstractions.
- **How Used in Code**:
  - Request routing: Exposes endpoints for workers (`/api/workers`), telemetry (`/api/telemetry`), alerts (`/api/alerts`), optimization reports (`/api/optimization/*`), and AI chat (`/api/chat`).
  - Middleware: `cors({ origin: '*' })` enables cross-origin requests; `express.json()` parses inbound JSON payloads; `express-rate-limit` enforces $300\text{ req/minute}$ on `/api/`.
- **Architectural Nuance**: In `server.js`, Express shares the underlying HTTP server with the WebSocket engine:
  ```javascript
  const app = express();
  const server = http.createServer(app);
  const wss = new WebSocket.Server({ server });
  ```

---

### Aedes MQTT Broker (v0.51.2) & MQTT Protocol
- **What It Is**: A bare-metal, high-performance MQTT broker implemented in pure Node.js, combined with the MQTT pub/sub protocol.
- **Why Used in SafeMine**: MQTT (Message Queuing Telemetry Transport) is the industry standard for IoT and IIoT communication. Its binary packet header (as small as 2 bytes) minimizes network overhead over constrained bandwidth links (e.g., subterranean LoRaWAN or leaky feeder repeaters) where HTTP header overhead ($>500\text{ bytes}$) is wasteful.
- **How Implemented**:
  - In `server.js` (lines 30–40), `aedes()` is bound to a Node `net.createServer` listening on port 1883.
  - The server creates an internal client (`mqtt.connect`) that subscribes to wildcards:
    - `mineguard/helmet/#`: Ingests miner telemetry.
    - `mineguard/wristband/#`: Ingests vital sign fallback packets.
    - `mineguard/anchor/#`: Ingests repeater health beacons.
  - Downlink commands are published to `mineguard/helmet/:id/downlink` with `qos: 1` to guarantee delivery of critical evacuation alarms to hardware nodes.
- **Quality of Service (QoS) Mechanics**:
  - `QoS 0` (At most once): Used for recurring telemetry (3s interval); dropping an occasional packet is acceptable because a fresh packet arrives 3 seconds later.
  - `QoS 1` (At least once): Used for emergency downlink commands (`sendNodeDownlink`); ensures the hardware node acknowledges receipt of the evacuation command.

---

### WebSockets (`ws` v8.17.0)
- **What It Is**: A standardized protocol (RFC 6455) providing persistent, full-duplex, bidirectional communication channels over a single TCP connection.
- **Why Used in SafeMine**: Traditional HTTP polling introduces latency ($1\text{--}5\text{ seconds}$) and massive HTTP header overhead. WebSockets allow the SafeMine server to push live telemetry updates, emergency alerts, and countdown timers to connected browser dashboards with sub-millisecond latency the instant a packet arrives from the mine.
- **Internal Protocol Handshake**:
  1. Client sends standard HTTP GET request with headers:
     `Upgrade: websocket` and `Connection: Upgrade`.
  2. Server responds with HTTP status code `101 Switching Protocols`.
  3. The TCP connection remains open, transmitting lightweight framed binary or UTF-8 text packets without HTTP headers.
- **Code Implementation**:
  - In `server.js`, `broadcast(type, payload)` iterates through the `wss.clients` Set:
    ```javascript
    function broadcast(type, payload) {
      const msg = JSON.stringify({ type, payload, ts: Date.now() });
      wss.clients.forEach(c => {
        if (c.readyState === WebSocket.OPEN) c.send(msg);
      });
    }
    ```

---

## 3. Data & Storage Tier

### `sql.js` (v1.11.0) — WebAssembly SQLite
- **What It Is**: The official SQLite C codebase compiled to WebAssembly (WASM) using Emscripten, enabling a full relational SQL database engine to execute inside the JavaScript virtual machine.
- **Why Used in SafeMine**:
  - Traditional native SQLite bindings for Node.js (such as `better-sqlite3` or `sqlite3`) require platform-specific native C++ compilation during `npm install`. This frequently fails on Windows enterprise laptops that lack Visual Studio C++ Build Tools or Python.
  - `sql.js` runs anywhere Node.js or a modern browser runs, providing cross-platform reliability.
- **How It Works Internally**:
  1. `initSqlJs()` loads the WebAssembly binary into memory.
  2. `_sqlDb = new SQL.Database(buf)` allocates a virtual memory space representing the SQLite VFS (Virtual File System).
  3. SQL queries execute entirely in WebAssembly heap memory at near-native speeds.
  4. Disk Persistence: Because WebAssembly heap memory is volatile, `db.js` manually implements persistence by calling `_sqlDb.export()` (which serializes the entire database into a `Uint8Array`) and writing it to disk with `fs.writeFileSync(DB_PATH)`.
- **Performance Trade-off**:
  - Reads are fast in-memory lookups.
  - Writes incur heavy I/O overhead: every insert or update re-exports and re-writes the complete database file to disk synchronously.

---

## 4. Hardware & Edge Sensing Tier

### ESP32 Microcontroller (Tensilica Xtensa Dual-Core 32-bit LX6)
- **Specifications**: 240 MHz clock speed, 520 KB SRAM, integrated 2.4 GHz 802.11 b/g/n Wi-Fi and Bluetooth v4.2 BR/EDR and BLE.
- **Role in SafeMine**: Serves as the central computing unit inside the Smart Helmet (`ESP32.ino`).
- **Internal Operation**:
  - Runs the Arduino/C++ framework on top of a FreeRTOS real-time kernel.
  - Samples analog and digital sensors within a non-blocking `loop()` scheduled at `SEND_INTERVAL = 3000ms`.
  - Executes local oxygen calculation algorithms directly on the microcontroller to ensure local alarm capability even if network connectivity is severed.

---

### Sensor Mechanics & Working Principles

#### 1. MQ-2 Flammable Gas Sensor
- **Target Gases**: Methane ($\text{CH}_4$), Butane, LPG, Smoke.
- **Physical Principle**:
  - Uses a Tin Dioxide ($\text{SnO}_2$) semiconductor sensitive layer. When heated to operating temperature ($\sim 200\text{--}300^\circ\text{C}$ by an internal coil), oxygen ions adsorb onto the $\text{SnO}_2$ crystal surface, creating a potential barrier that restricts electron flow (high resistance in clean air).
  - When combustible reducing gases (like $\text{CH}_4$) come into contact with the surface, they react with the adsorbed oxygen ions, releasing electrons back into the semiconductor conduction band. This lowers electrical resistance proportionally to gas concentration.
- **Hardware Circuit**:
  - Arranged as a voltage divider with a load resistor ($R_L$). As gas concentration rises, resistance ($R_s$) drops, increasing the output analog voltage.
- **ADC Sampling**:
  - Connected to ESP32 pin GPIO 34 (ADC1 channel 6). The 12-bit ADC converts $0\text{--}3.3\text{V}$ into digital counts from $0$ to $4095$.

---

#### 2. MPU-6050 6-Axis Motion Tracking Sensor
- **Capabilities**: 3-axis MEMS accelerometer ($\pm 2\text{g}$) + 3-axis MEMS gyroscope ($\pm 250^\circ/\text{s}$).
- **Physical Principle**:
  - **Accelerometer**: Operates using micro-machined capacitive differential sensors. Acceleration causes a suspended seismic proof-mass to deflect from its neutral position, altering the capacitance between fixed plates and plates attached to the mass. On-chip circuitry converts capacitance deltas into a proportional voltage.
  - **Gyroscope**: Utilizes the Coriolis Effect. When the sensor rotates along any axis, vibrating drive elements experience a Coriolis acceleration, producing an orthogonal vibration that capacitive sensors detect.
- **Bus Interface**: Connected via standard I2C bus (address `0x68`, SDA GPIO 21, SCL GPIO 22).

---

#### 3. DHT11 Temperature & Relative Humidity Sensor
- **Capabilities**: Temperature ($0\text{--}50^\circ\text{C} \pm 2^\circ\text{C}$), Humidity ($20\text{--}90\%\text{ RH} \pm 5\%$).
- **Physical Principle**:
  - **Humidity**: Uses a resistive moisture-sensing substrate sandwiched between two electrodes. As water vapor is absorbed by the porous polymer dielectric, conductivity increases.
  - **Temperature**: Uses an integrated negative temperature coefficient (NTC) thermistor whose resistance decreases non-linearly as temperature rises.
- **Communication Protocol**: Proprietary single-wire bidirectional serial protocol on GPIO 4 with tight microsecond timing handshakes.

---

#### 4. BMP180 / BMP085 Digital Barometric Pressure Sensor
- **Capabilities**: Barometric pressure ($300\text{--}1100\text{ hPa}$), high precision elevation resolution ($0.25\text{m}$).
- **Physical Principle**: Utilizes piezoresistive silicon MEMS technology. Atmospheric air pressure exerts physical force against a flexible diaphragm with integrated piezoresistors. Mechanical deflection induces strain, modifying resistance via the piezoresistive effect. An on-chip 16-bit ADC samples the bridge voltage.
- **Mining Application**:
  - Tracks atmospheric pressure drops: If ventilation fans shut down or a subterranean blast wave propagates through a tunnel, pressure fluctuates sharply ($\Delta P > 20\text{ hPa}$).
  - Depth calculation: Barometric formula relates increasing hydrostatic atmospheric column pressure to subterranean drift depth below the surface entrance.

---

## 5. Intelligence, Telephony & Cloud Tier

### Groq Cloud AI & Llama 3.3 70B
- **What It Is**: Ultra-low-latency AI inference infrastructure powered by Groq’s custom Language Processing Units (LPUs), serving Meta's open-weights Llama 3.3 70B Versatile model.
- **Why Used**:
  - Standard GPU cloud providers (OpenAI, AWS Bedrock) exhibit token generation latencies of $1\text{--}3\text{ seconds}$.
  - In an underground mining emergency (e.g. stope collapse or toxic gas breakout), safety operators require immediate assistance. Groq generates text at speeds exceeding $250\text{--}300\text{ tokens/sec}$, returning structured safety recommendations in under $500\text{ms}$.
- **Context Injection Architecture (`server.js`, line 1302)**:
  - Queries active database state: worker vitals, peak methane, active geofence breaches, and unresolved alarms.
  - Injects live context into the system prompt:
    ```
    LIVE MINE STATUS (2026-11-23T...):
    - Miners: 12 total | 1 emergency | 1 critical
    - Peak CH4: 1.20% | Peak CO: 55ppm | Min O2: 19.4%
    - Active alerts: [CRITICAL] Methane Critical — Zone C
    ...
    ```
  - Dispatches query via HTTP POST to `https://api.groq.com/openai/v1/chat/completions`.

---

### Twilio Telecommunications API & TwiML
- **What It Is**: A global cloud Communications Platform as a Service (CPaaS) enabling programmatic voice calling and SMS messaging via REST APIs.
- **How Used in SafeMine**:
  - When an emergency or critical alert is verified by the sensor fusion engine, `emergency-contact.js` sends an authenticated HTTP POST request to Twilio’s endpoint.
  - **TwiML (Twilio Markup Language)**: The server dynamically generates XML voice instructions:
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <Response>
      <Say voice="alice" language="en-IN">
        MINE GUARD EMERGENCY ALERT.
        Automated call from Mine Guard Pro Safety System.
        Methane Critical Zone C.
        Please check the dashboard immediately.
      </Say>
      <Pause length="2"/>
    </Response>
    ```
  - Twilio dials the Mine Safety Manager’s mobile device (`ADMIN_PHONE`) and synthesizes speech over the cellular voice network.

---

## 6. Client Presentation & Visualization Tier

### HTML5 Canvas 2D Engine
- **Why Canvas over DOM Nodes**:
  - Rendering 12 miners with animated pulsing hazard circles, 10 anchor nodes, radio signal coverage radii, and dynamic multi-level tunnel networks using HTML DOM elements (`<div>`) causes high layout recalculation and repaint overhead.
  - Canvas 2D uses immediate-mode graphics: JavaScript draws directly to a bitmap buffer via hardware-accelerated 2D contexts, maintaining 60 FPS smoothly.

---

### Leaflet.js (v1.9.4)
- **What It Is**: An open-source, mobile-friendly JavaScript library for interactive web maps.
- **How Used**:
  - Powers the Surface Map view (`#page-map`), rendering OpenStreetMap tile layers centered at the mine's geographical coordinates (`mineLat: 6.2041, mineLng: -1.6747`, Obuasi Gold Mine).
  - Overlays custom SVG miner marker pins showing real-time GPS positions for surface and shallow portal operations.
