1. Now that our application has used **DNS** to find the target IP address and has formatted an **HTTP** request, it hands that data down to the **Transport Layer**.

#### THE TRANSPORT LAYER 
1. It is responsible for process to process communication - making sure data gets from a specific application on your computer(like chrome) to a specific application on the server (like the web server software).
2. It does it using the help of port numbers (like http port no 80 and HTTPS port number is 443).
3. The internet gives two primary protocols : TCP or UDP.

### TCP (TRANSMISSION CONTROL PROTOCOL) - THE RELIABLE MANAGER
1. Connection oriented protocol.
2. Before sending a single byte of application data, it forces the client and server to establish a formal connection (a handshake).
3. It prioritizes accuracy and reliability over absolute speed.
4. **Reliable Delivery:** If a packet is lost or corrupted in transit, TCP detects it and automatically retransmits it.
5. **Ordered Data:** Packets can arrive out of order because they take different paths across the internet. TCP uses **Sequence Numbers** to rearrange them back into the exact original order before handing them to the application.
6. **Flow & Congestion Control:** TCP monitors the network and the receiver's status to slow down or speed up data transmission, ensuring nothing gets overwhelmed.
7. **Heavyweight:** The TCP header is relatively large (**20 bytes**) to hold all this tracking metadata.
8. **Real-world Uses**Web browsing (HTTP/HTTPS), Email (SMTP), File Transfers (FTP), SSH


### UDP (USER DATAGRAM PROTOCOL) - THE FAST COURIER
1. It is a connectionless protocol.
2. It does not establish a connection, it does not handshake, and does not care if the receiver is active or ready.
3. Its imply wraps the data and fires it into the network.
4. It prioritizes speed and low latency over absolute reliability.
5. **Unreliable (Best-effort):** If a packet is lost on the way, UDP will _not_ retransmit it. The data is simply gone.
6. **Unordered:** Packets are processed in whatever random order they happen to show up.
7. **No Controls:** UDP has no flow or congestion control. It shoots data as fast as your application generates it.
8. **Lightweight:** Because it does so little, its header is tiny—just **8 bytes**.
9. Use case: Live Video Streaming, Online Gaming, Voice calls (VoIP), DNS queries


## TOPIC B : THE TCP 3-Way Handshake (Connection Establishment)

1. Before a browser can send an HTTP request to a server via TCP, they perform a three-step dance to synchronize **Sequence Numbers**.
2. These numbers ensure data is reassembled in the correct order and no packets are missed.
3. The process uses two primary flags in the TCP header:

- **SYN (Synchronize):** Used to initiate a connection and establish initial sequence numbers.
    
- **ACK (Acknowledge):** Used to confirm that a packet was successfully received.

#### The Steps:

1. **Step 1: SYN (Client $\rightarrow$ Server)**
    
    - The client picks a random initial sequence number ($x$) and sends a packet with the **SYN** flag set to `1`.
        
    - _Client State:_ `SYN_SENT` | _Server State:_ `LISTEN`
        
2. **Step 2: SYN-ACK (Server $\rightarrow$ Client)**
    
    - The server receives the request, allocates resources for the connection, picks its own random initial sequence number ($y$), and replies.
        
    - It sets both **SYN** and **ACK** flags to `1`. It acknowledges the client's packet by setting the acknowledgment number to $x + 1$.
        
    - _Server State:_ `SYN_RCVD`
        
3. **Step 3: ACK (Client $\rightarrow$ Server)**
    
    - The client receives the server's response and sends a final acknowledgment packet with the **ACK** flag set to `1`. It sets the acknowledgment number to $y + 1$.
        
    - _Client State:_ `ESTABLISHED` | _Server State:_ `ESTABLISHED`
        

Once this third packet is processed, the connection is live, and the client can immediately begin streaming actual HTTP application data.

# "You said TCP is connection-oriented. How exactly does it establish that connection?"

## THE TCP 4-WAY TEARDOWN (CONNECTION TERMINATION)

Because a TCP connection is **full-duplex** (both client and server can send and receive data at the same time independently), closing a connection requires a graceful shutdown from both sides. This requires four steps and uses the **FIN (Finish)** flag.

#### The Steps:

1. **Step 1: FIN (Client $\rightarrow$ Server)**
    
    - The client has no more data to send, so it sends a packet with the **FIN** flag set to `1`.
        
    - _Client State:_ `FIN_WAIT_1`
        
2. **Step 2: ACK (Server $\rightarrow$ Client)**
    
    - The server receives the FIN and sends an **ACK** back to confirm it. However, the server might still have background data it needs to finish sending to the client, so the connection remains half-open.
        
    - _Server State:_ `CLOSE_WAIT` | _Client State:_ `FIN_WAIT_2`
        
3. **Step 3: FIN (Server $\rightarrow$ Client)**
    
    - Once the server finishes sending all its remaining data, it sends its own **FIN** packet to the client to close its side of the connection.
        
    - _Server State:_ `LAST_ACK`
        
4. **Step 4: ACK (Client $\rightarrow$ Server)**
    
    - The client receives the server's FIN and responds with a final **ACK**.
        
    - _Server State:_ `CLOSED`


### TIME_WAIT

The **`TIME_WAIT`** state is one of the most subtle yet critical parts of the TCP state machine.

## What is Maximum Segment Lifetime (MSL)?

The **Maximum Segment Lifetime (MSL)** is the maximum amount of time that any network packet (segment) can physically exist within the network infrastructure before being discarded.

- **Why do packets live and die?** When routers forward packets, they use a **TTL (Time to Live)** field in the IP header, which acts as a hop counter. If a packet gets caught in a routing loop, the TTL eventually hits 0, and a router drops it.
    
- **The Time Value:** MSL is an engineering estimate of how long a packet might wander around a loopy or congested path before that drop happens. While the official RFC 793 specification suggests setting MSL to **2 minutes**, modern operating systems often optimize this and hardcode it to **30 seconds** or **60 seconds**.

## Why does `TIME_WAIT` exist?

During a 4-way teardown, the device that initiates the active close (let's assume it's the **Client**) sends the final `ACK` to the server's `FIN`. Instead of immediately transitioning to the `CLOSED` state and releasing its port resource, the client enters the **`TIME_WAIT`** state.

The client stays in `TIME_WAIT` for exactly **$2 \times \text{MSL}$** (often called 2MSL). If MSL is 30 seconds, the client sits silently for 60 seconds.


There are **two crucial reasons** why this delay is mandatory:

### Reason 1: To Guarantee the Final ACK is Delivered (Reliability)

TCP is obsessed with reliability. If the Client sends its final `ACK` and instantly closes, it washes its hands of the connection.

But what happens if that final `ACK` is dropped by a congested router?

1. The Server never receives the `ACK`.
    
2. The Server's timer expires, and it assumes its own `FIN` packet was lost.
    
3. The Server retransmits the `FIN` packet.

If the Client had instantly closed, its port would be shut down. When the retransmitted `FIN` arrives, the Client’s operating system would look at it, find no active connection matching that port, and respond with an **`RST` (Reset/Error) packet**. The Server, instead of closing gracefully, gets an error, leaving it in an unclosed or broken state.

By waiting $2 \times \text{MSL}$, the client ensures that if the server retransmits the `FIN`, the client is still alive to receive it and re-send the `ACK`.

#### Why exactly 2MSL?

- **1 MSL** ensures that any stray packets sent by the client toward the server completely expire and vanish from the network.
    
- **Another 1 MSL** ensures that any stray packets sent by the server toward the client (including retransmitted `FIN`s) also completely die off.
    
- After $2 \times \text{MSL}$, the network pipeline is perfectly clean. It is mathematically guaranteed that no ghost packets from the old connection remain alive.


# But what happens if your computer is an absolute powerhouse capable of sending data at 1 Gbps, but the server you are talking to is an older machine running on a slow connection that can only process data at 100 Mbps? Or what happens if the routers in the middle of the internet get overwhelmed with traffic?


## TOPIC C: TCP FLOW CONTROL AND CONGETSION CONTROL

Once a connection is established, data packets begin flowing. However, network speeds cannot remain static because network conditions change.

- **Flow Control** protects the **Receiver** from being overwhelmed.
    
- **Congestion Control** protects the **Network Fabric** (routers and links) from being overwhelmed.


### 1. Flow Control (Protecting the Receiver)

Imagine a powerful server sending data at 1 Gbps to a small mobile device that can only process data at 50 Mbps. If the server blasts data at full speed, the mobile device's buffer will fill up instantly, causing it to drop packets.

To solve this, TCP uses a mechanism called the **Sliding Window Protocol**.

- **The Receive Buffer:** The receiver sets aside a portion of memory (a buffer) to hold incoming data until the application can read it.
    
- **The Window Size:** In every single TCP packet sent _back_ to the sender, the receiver includes a field called **`Window Size` (or Receive Window - `rwnd`)**. This tells the sender exactly how much free space is left in its buffer.
    
- **The Action:** The sender ensures that the total amount of unacknowledged data it sends never exceeds the advertised `rwnd`. If the receiver's application gets bogged down and the buffer fills up completely, it advertises a `Window Size = 0`. The sender stops transmitting immediately until it receives a packet showing the window has opened back up.


### 2. Congestion Control (Protecting the Network)

Even if the receiver has an infinite buffer, the routers sitting between the sender and receiver might be heavily congested with global traffic. If too many packets hit a router at once, its internal queues overflow, and it drops the packets.

To prevent this "global gridlock," the sender dynamically calculates a second variable called the **Congestion Window (`cwnd`)**.

The sender operates on a golden rule:

$$\text{Max Data Allowed in Flight} = \min(\text{rwnd}, \text{cwnd})$$

Unlike `rwnd` (which is explicitly told to the sender by the receiver), `cwnd` must be guessed by the sender through trial and error. It does this using three core phases:

#### Phase A: Slow Start (Exponential Growth)

When a connection starts, the sender has no idea what the network can handle.

- It sets `cwnd = 1` (or a small initial value like 10).
    
- Every time it receives an `ACK`, it doubles the `cwnd` value ($1 \rightarrow 2 \rightarrow 4 \rightarrow 8 \rightarrow 16$).
    
- Even though it's called "Slow Start," the growth is actually **exponential** and very aggressive.
    
- This continues until `cwnd` hits a threshold called **`ssthresh` (Slow Start Threshold)**.
    

#### Phase B: Congestion Avoidance (Linear Growth)

Once `cwnd` crosses `ssthresh`, the sender shifts into a cautious mode to avoid triggering a sudden crash.

- Instead of doubling, it increases `cwnd` by exactly **1 segment per round-trip** ($16 \rightarrow 17 \rightarrow 18 \rightarrow 19$).
    
- This **linear growth** continues safely until a packet drop occurs.
    

#### Phase C: Handling Packet Loss (Reaction)

How the sender reacts to a drop depends entirely on _how_ it discovers the drop:

1. **Scenario 1: Timeout (Severe Congestion)**
    
    - The sender transmits data, but a timer expires completely without receiving an `ACK`. This implies severe network blockage.
        
    - **The Reaction:** It reacts aggressively. It slashes `ssthresh` to half of the current `cwnd`, plunges `cwnd` back down to `1`, and enters **Slow Start** all over again.
        
2. **Scenario 2: Three Duplicate ACKs (Mild Congestion - Fast Retransmit)**
    
    - If the sender receives **3 identical, duplicate ACKs** for an old packet, it implies that packets _after_ the missing one are still getting through to the receiver, but the missing one left a gap. The network isn't dead; it's just dropping a line.
        
    - **The Reaction:** Instead of starting over from scratch, it triggers **Fast Retransmit**. It immediately re-sends the missing packet without waiting for a timeout, drops `cwnd` by only half, and continues with **Congestion Avoidance (Linear growth)**. This is a much smoother optimization.