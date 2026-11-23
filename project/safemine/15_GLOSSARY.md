# SafeMine: Technical & Industrial Mining Glossary

---

## 1. Mining, Regulatory & Industrial Safety Terminology

### ATEX (Appareils destinés à être utilisés en ATmosphères EXplosibles)
European Union directives (2014/34/EU and 1999/92/EC) regulating equipment and protective systems intended for use in potentially explosive atmospheres (e.g. underground coal mines with flammable methane gas).

### Lower Explosive Limit (LEL)
The minimum concentration of combustible gas or vapor in air below which the gas cannot be ignited. For methane ($\text{CH}_4$), the LEL is **$5.0\%$ by volume**. SafeMine triggers warning alerts when methane reaches $0.5\%$ ($10\%$ of LEL) and critical evacuation alerts at $1.0\%$ ($20\%$ of LEL).

### Upper Explosive Limit (UEL)
The maximum concentration of combustible gas in air above which there is insufficient oxygen to sustain combustion. For methane, the UEL is **$15.0\%$ by volume**. Between $5\%$ and $15\%$, methane air mixtures are violently explosive.

### MSHA (Mine Safety and Health Administration)
An agency of the United States Department of Labor that enforces safety regulations across all mining operations in the US under the Federal Mine Safety and Health Act of 1977.

### MINER Act of 2006 (Mine Improvement and New Emergency Response Act)
US federal legislation enacted following the Sago and Alma mine disasters mandating that underground mines implement post-accident two-way wireless communications, continuous electronic worker tracking, hardened underground refuge chambers, and local emergency response teams.

### Stope
An underground excavation or working face from which ore is actively extracted in a series of steps or benches. Stopes are often poorly ventilated dead-end headings prone to gas accumulation.

### Drift
A horizontal subterranean passageway driven along or parallel to the course of a vein or rock formation, used for access, haulage, and ventilation.

### Heading
The advancing end or blind face of an excavation where miners and automated drill jumbos are actively working.

### Refuge Chamber (Life-Support Sanctuary)
A reinforced, air-sealed underground sanctuary equipped with independent oxygen tanks, carbon dioxide scrubbing systems, food, water, and first aid supplies designed to shelter miners for up to 96 hours during a fire or toxic gas release when exit shafts are impassable. SafeMine's Dijkstra engine dynamically routes workers to refuge chambers (`REFUGE-1`, `REFUGE-2`) if surface portals are blocked.

### Muster Point / Station
A designated safe assembly area, either on the surface or underground, where personnel gather during an evacuation to account for all workers on shift. SafeMine generates automated digital Muster Reports comparing checked-in workers against the total roster.

### Dalton’s Law of Partial Pressures
Physical law stating that in a mixture of non-reacting gases, total pressure equals the sum of the partial pressures of each individual gas. In SafeMine's oxygen depletion model (`o2Calculation.js`), when methane leaks into a confined tunnel, its volume displaces air, lowering the partial pressure of oxygen proportionally.

### Leaky Feeder System
A subterranean radio communications architecture utilizing a specialized coaxial cable with slots cut in its outer shield, allowing RF signals to "leak" into and out of the cable along the entire length of a mine tunnel. Acts as an extended antenna for handheld VHF/UHF radios and LoRa gateways.

---

## 2. Hardware, Embedded Systems & Sensor Physics

### ADC (Analog-to-Digital Converter)
An electronic circuit that converts a continuous analog voltage into a discrete digital number. The ESP32 contains two 12-bit SAR ADCs capable of quantizing voltages from $0\text{ to }3.3\text{V}$ into integer counts from $0\text{ to }4095$ ($2^{12}$ states). SafeMine samples the MQ-2 gas sensor on ADC1 channel 6 (GPIO 34).

### Full-Scale Range (FSR)
The maximum operational input span that a sensor or ADC is configured to measure. In `ESP32.ino`, the MPU-6050 accelerometer is configured for an FSR of $\pm 2\text{g}$, mapping $4\text{g}$ across 65,536 counts ($16,384\text{ LSB/g}$).

### Least Significant Bit per g (LSB/g)
The sensitivity scale factor of a digital accelerometer. Represents how many raw ADC units correspond to one standard gravitational acceleration ($1\text{g} \approx 9.81\text{ m/s}^2$).

### Proof-Mass (MEMS)
A microscopic seismic mass suspended on silicon springs inside a Micro-Electro-Mechanical Systems (MEMS) accelerometer. When accelerated, inertia causes the proof-mass to deflect, altering the capacitance between interdigitated finger plates.

### Piezoresistive Effect
The change in electrical resistivity of a semiconductor material when subjected to mechanical strain. The BMP180 sensor uses piezoresistors integrated into a silicon diaphragm to measure atmospheric air pressure changes.

### NTC Thermistor (Negative Temperature Coefficient)
A thermally sensitive resistor whose electrical resistance decreases non-linearly as temperature increases. Used in the DHT11 sensor to monitor ambient thermal heat stress.

### SnO2 Semiconductor Layer
Tin dioxide sensing material used in MQ-2 electrochemical gas sensors. Adsorbs oxygen ions in clean air, creating an electrical potential barrier that reduces conductivity. Contact with combustible hydrocarbons ($\text{CH}_4$) releases electrons, lowering electrical resistance.

### FreeRTOS
An open-source, real-time operating system kernel for microcontrollers providing preemptive task scheduling, semaphores, queues, and software timers. Used under the hood by the ESP32 Arduino framework.

---

## 3. Networking, Wireless Telemetry & Cloud Protocols

### LoRa / LoRaWAN (Long Range Wide Area Network)
A proprietary spread-spectrum radio modulation technology derived from Chirp Spread Spectrum (CSS) operating in sub-GHz unlicensed ISM bands ($868\text{ MHz}$ Europe, $915\text{ MHz}$ US). Capable of transmitting small sensor packets over long distances with high penetration through solid rock and underground tunnels.

### UWB (Ultra-Wideband)
A short-range RF technology operating across wide bandwidths ($>500\text{ MHz}$) using nanosecond pulses. Enables centimeter-level indoor positioning accuracy via Time-of-Flight (ToF) distance measurement.

### RSSI (Received Signal Strength Indication)
A measurement of the power present in a received radio signal, expressed in negative decibel-milliwatts ($\text{dBm}$). Values range from $-30\text{ dBm}$ (strong, adjacent to anchor) to $-110\text{ dBm}$ (weak, fringe connectivity). SafeMine uses RSSI trilateration across anchors for node positioning and penalizes high risk if $\text{RSSI} < -90\text{ dBm}$.

### MQTT (Message Queuing Telemetry Transport)
A lightweight, binary publish-subscribe messaging protocol (ISO/IEC 20922) designed for constrained devices and high-latency, unreliable networks. Features small packet headers (as low as 2 bytes) and persistent client sessions.

### Quality of Service (QoS in MQTT)
- **QoS 0 (At most once)**: Fire-and-forget; no delivery guarantee or acknowledgment. Used by SafeMine for regular telemetry updates.
- **QoS 1 (At least once)**: Packet is delivered and acknowledged with a `PUBACK`; retransmitted if unacknowledged. Used by SafeMine for emergency evacuation downlinks.
- **QoS 2 (Exactly once)**: Four-step handshake ensuring zero duplicates. High network overhead.

### WebSockets (RFC 6455)
A transport protocol providing full-duplex, persistent communication channels over a single TCP connection. Initiated via an HTTP upgrade request, enabling zero-latency server push broadcasting to browsers.

### TwiML (Twilio Markup Language)
An XML-based instruction language used by Twilio CPaaS to execute programmatic voice calling, text-to-speech synthesis (`<Say>`), call pausing (`<Pause>`), and telephony call trees.

---

## 4. Software Architecture, Database & Graph Theory

### WebAssembly (WASM)
A portable binary-code format and virtual machine executing inside modern web browsers and Node.js runtimes at near-native speeds. `sql.js` compiles the official SQLite C codebase into WASM, allowing an in-memory SQL relational database to run without native OS compiler toolchains.

### Virtual File System (VFS)
An abstraction layer inside SQLite that sits between the SQL query engine and the operating system's filesystem. In `sql.js`, the VFS is mapped to an in-memory byte array rather than disk file descriptors.

### Dijkstra’s Algorithm
A greedy graph search algorithm that calculates the shortest path from a single source vertex to all other vertices in a directed or undirected graph with non-negative edge weights.

### Adjacency List
A graph data structure representing a finite graph as an associative array or dictionary where each key represents a vertex and maps to an array of its neighboring vertices and edge weights.

### Libuv
A multi-platform C library providing asynchronous I/O based on an event loop and thread pool. Powers the asynchronous non-blocking event engine of Node.js.

### Cross-Origin Resource Sharing (CORS)
A browser security mechanism using HTTP headers to permit or restrict resources from being requested by web applications running on a different origin domain, protocol, or port.

### JSON Web Token (JWT, RFC 7519)
A compact, URL-safe means of representing claims between two parties. Composed of a Header, Payload, and cryptographic Signature (e.g. HMAC-SHA256). Used in SafeMine’s `auth.js` to authenticate operators.
