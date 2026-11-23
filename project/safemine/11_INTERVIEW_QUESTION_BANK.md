# SafeMine: Comprehensive 13-Level Technical Interview Question Bank

---

## Level 1: Project Fundamentals

1. **What is SafeMine, and what core real-world problem does it solve?**
2. **Who are the primary end-users of the platform, and what are their specific operational needs?**
3. **What makes underground mine safety monitoring fundamentally different from surface environmental monitoring?**
4. **Why can't standard mobile devices (smartphones with 4G/5G and GPS) be used in underground mines?**
5. **What are the primary atmospheric gas hazards tracked in SafeMine, and what are their danger thresholds?**
6. **What is the dual-wearable redundancy concept implemented in SafeMine?**
7. **What happens if a subterranean network connection is severed between the underground sensors and the surface server?**

---

## Level 2: Code-Level Understanding

8. **In `ESP32.ino`, explain the tri-axial accelerometer math used to assert a fall. Why is `ax` divided by 16384.0?**
9. **In `ESP32.ino`, why is there an `IMPACT_WINDOW` of 5000ms? What specific false positives does this prevent?**
10. **Explain how `realHardwareWorkers` in `backend/server.js` functions. What problem does it solve?**
11. **Walk through the logic of `handleHelmetUpdate()` in `server.js`. What happens from line 243 to line 540?**
12. **In `db.js`, how does `_toPositional()` convert named SQLite parameters (`@worker_id`) to positional placeholders (`?`)?**
13. **In `server.js`, why does `checkSafetyThresholds()` maintain `_lastAlert` with an `ALERT_TTL_MS` check?**
14. **Explain how `parseShiftRange()` and `isInRange()` in `server.js` (lines 1220–1241) handle overnight shifts crossing midnight (e.g. 18:00 to 06:00).**
15. **What is the role of `frontend/data/mockData.js`, and what anomaly exists in this file?**

---

## Level 3: Technology Stack Deep Dive

16. **Why was Node.js chosen as the runtime engine for this industrial IoT platform? Where does Node struggle in this context?**
17. **Explain the internal difference between `sql.js` and `better-sqlite3`. Why was `sql.js` chosen despite its I/O performance penalty?**
18. **Explain the MQTT protocol architecture. What are the differences between QoS 0, QoS 1, and QoS 2, and where are they used in SafeMine?**
19. **How does the embedded Aedes MQTT broker communicate with the Express application in `server.js`?**
20. **Why did you use native WebSockets (`ws`) instead of Socket.io or Server-Sent Events (SSE)?**
21. **How does the MQ-2 sensor operate at a physical and chemical level? How is its analog voltage mapped to methane percentage?**
22. **Explain the working principle of the BMP180 barometric pressure sensor. How does barometric pressure relate to mine depth?**
23. **What is Groq's Language Processing Unit (LPU) architecture, and why was it chosen over standard OpenAI API endpoints?**

---

## Level 4: System Architecture & Data Flow

24. **Is SafeMine a monolith, microservices, or event-driven architecture? Justify your architectural classification.**
25. **Trace the exact lifecycle of a sensor packet from physical GPIO pin sampling to a pixel update on the browser Canvas.**
26. **Explain the communication boundaries between the ESP32 hardware, the Node.js backend, and the browser client.**
27. **Why is the subterranean tunnel layout rendered on an HTML5 Canvas 2D engine while the surface map uses Leaflet.js?**
28. **Explain how the downlinks feature functions. How does an operator in the control room actuate a buzzer on a physical miner's helmet?**
29. **How does SafeMine implement sensor fusion? Give an example of a multi-sensor pattern that triggers an emergency.**

---

## Level 5: Technical Decision-Making & Trade-offs

30. **Why did you implement a physics-inspired mathematical model for oxygen depletion (`o2Calculation.js`) instead of relying solely on an O2 sensor reading?**
31. **What are the trade-offs of embedding an MQTT broker (`aedes`) in the main Node.js process versus running an external Mosquitto daemon?**
32. **Why was Dijkstra's algorithm chosen over $A^*$ or Floyd-Warshall for evacuation routing?**
33. **Why did you choose vanilla ES6 JavaScript for the frontend instead of React, Vue, or Angular?**
34. **Why use Twilio's cloud API for emergency phone calls instead of a physical GSM modem connected via USB?**
35. **Would you use the same database architecture (`sql.js`) if building this system for production deployment across 5,000 miners? Defend your answer.**

---

## Level 6: Algorithms, Mathematics & Complexity

36. **What is the time and space complexity of the Dijkstra implementation in `optimizer.js`? Where is the primary algorithmic bottleneck?**
37. **In `dijkstraEvacRoute()`, explain why hazard geofences inject a $+500$ penalty rather than pruning the edge from the graph entirely.**
38. **Explain the mathematical formulation of `calcOptimizedRisk()`. Why is a natural logarithm ($\ln(1+x)$) used for flammable gas and an exponential function ($e^x$) used for oxygen deficiency?**
39. **Explain the Dalton gas displacement formula in `o2Calculation.js`. How does 1% methane displace ambient oxygen?**
40. **How does `predictFatigue()` estimate a miner's hours underground without a dedicated check-in timestamp? What are the limitations of this proxy?**
41. **Explain the greedy ventilation prioritization algorithm in `optimizeVentilation()`. How are dynamic fan speeds calculated?**

---

## Level 7: Database Architecture & API Contracts

42. **Walk through the relational schema of `mineguard.db`. What are the primary tables and their relationships?**
43. **Why does `workers` store real-time operational state while `telemetry` functions as an append-only time-series log?**
44. **What indexes currently exist in the database, and what secondary indexes should be created to optimize production queries?**
45. **What happens during a call to `db.prepare(sql).run()`? Explain the exact steps leading to `fs.writeFileSync()`.**
46. **What is the difference between `POST /api/telemetry` and `POST /api/wristband` in terms of data schema, state updates, and worker status?**
47. **How does the WebSocket event contract distinguish between an incremental worker telemetry update and a critical safety alert?**

---

## Level 8: Security, Vulnerabilities & Compliance

48. **What is the most severe security vulnerability currently present in `backend/server.js`?**
49. **Explain how `app.use(express.static(path.join(__dirname, '..')))` creates a directory traversal risk. What sensitive files are exposed?**
50. **How does the codebase prevent SQL injection vulnerabilities? Provide code examples from `server.js` and `db.js`.**
51. **Why is `cors({ origin: '*' })` problematic in an industrial safety control system? How should it be restricted?**
52. **How does `auth.js` handle password hashing? What salt rounds are used, and why is bcrypt chosen over SHA-256?**
53. **How does the system prevent financial denial-of-service (telecom billing exhaustion) via Twilio endpoints?**

---

## Level 9: Performance, Concurrency & Scalability

54. **What happens to the Node.js event loop when 500 ESP32 helmets submit telemetry packets every 3 seconds under the current architecture?**
55. **Identify the single greatest I/O bottleneck in `backend/db.js` and explain how to refactor it for 10x throughput.**
56. **How would you redesign the architecture to scale from 12 miners to 100,000 miners distributed across 50 international mine sites?**
57. **How does client-side rendering scale when 1,000 miners move simultaneously? Why would DOM rendering freeze, and how does Canvas 2D mitigate this?**
58. **Why does SafeMine batch client WebSocket broadcasts during high load? What client-side state optimizations are required?**

---

## Level 10: Practical Debugging Scenarios

59. **Scenario: A physical helmet is transmitting packets, but the terminal logs `[HTTP Telemetry] 400 Invalid JSON body`. How do you isolate the issue?**
60. **Scenario: An operator clicks "Trigger Zone Evacuation", but connected browsers do not show the alert banner. Walk through your debugging checklist.**
61. **Scenario: The server crashes on startup with `Error: listen EADDRINUSE: address already in use :::3001`. How do you identify and terminate the conflicting process on Windows?**
62. **Scenario: A worker's oxygen level is reported as `15.0%` constantly, even though gas readings are zero. What code block is causing this, and how do you fix it?**
63. **Scenario: Twilio outbound voice calls fail during a panic alert with HTTP 401 Unauthorized. How do you verify and debug the issue without committing secrets?**

---

## Level 11: System Modification & Design Scenarios

64. **What exact code modifications are required to replace `sql.js` with PostgreSQL and `pg-promise`?**
65. **How would you add Over-the-Air (OTA) firmware update capabilities to the ESP32 smart helmets using the existing MQTT broker?**
66. **What changes would be necessary to support 3D underground mine wireframe models (DXF/OBJ files) instead of the 2D Canvas schematic?**
67. **How would you modify the system to operate in a completely air-gapped facility with zero external Internet connectivity?**
68. **How would you integrate an industrial Modbus TCP or OPC UA PLC controller to automatically start physical ventilation fans based on `optimizeVentilation()` recommendations?**

---

## Level 12: Behavioral, HR & Ownership Questions

69. **Which specific part of this codebase was the most technically challenging to design and implement, and why?**
70. **Tell me about a significant architectural mistake you made in this project and how you would rectify it today.**
71. **How did you balance the need for rapid prototype delivery against strict industrial safety-critical reliability?**
72. **What was your decision-making process when choosing between an embedded MQTT broker and an external message queue?**
73. **If you had an additional two weeks to work on SafeMine, what top three improvements would you prioritize?**

---

## Level 13: Difficult & Stress-Test Interviewer Questions

74. **"Your `auth.js` implements complete Supabase and JWT gate logic, yet you forgot to mount it in `server.js`. Did you actually build and test this authentication system, or did you copy-paste it?"**
75. **"You claim SafeMine is safety-critical, yet your database synchronously writes 18MB to disk on every single 3-second sensor packet. In an emergency with 50 miners, your event loop freezes and drops packets. Isn't this system dangerously unfit for life-safety?"**
76. **"Your Dijkstra implementation uses `queue.sort()` inside a while loop instead of a priority queue, yielding $O(V^2 \log V)$. Why are you claiming Staff-level engineering with an introductory CS data structure flaw?"**
77. **"Your fall detection algorithm in `ESP32.ino` relies on a fixed 5000ms delay and hardcoded thresholds. How do you know a miner jumping off a haul truck won't trigger a false mine-wide evacuation?"**
78. **"Why did you put 1,500 lines of CSS inside `frontend/data/mockData.js`? What does this say about your code organization and quality control?"**
