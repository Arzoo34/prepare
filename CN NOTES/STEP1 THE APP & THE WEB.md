HTTP -> Hypertext Transfer Protocol
-> Application layer protocol that runs on top of transport layer to let clients (browsers) and servers exchange data.
-> HTTP provides a universal language.
-> HTTP is used to transfer data (HTML web pages, images, videos) across the web. It operates on REQUEST-RESPONSE MODEL:
1. Your browser formats an HTTP request (e.g. GET/ index.html)
2. The server processes it and sends back an HTTP Response containing the requested file and status code (Like 200 OK)
#### HTTP/1.1 THE RELIABLE BUT SLOW VETERAN
-> Introduced persistent connections
-> means a single TCP connection could be reused for multiple requests (instead of opening a new connection for every single image or CSS).
##### THE PROBLEM :
It suffered from ==**Head-of-Line (HOL)** BLOCKING AT THE APPLICATION LAYER.==
-> If browser requests 4 files they have to be in order. if first file too slow the rest 3 are completely blocked behind it
#### HTTP/2 - THE MULTIPLEXING REVOLUTION (2015)
-> changes how data was framed without changing the core HTTP semantics
##### THE FIX:
-> It introduced ==**MULTIPLEXING via a Binary Framing Layer.**==
-> Instead of text, data was broken into tiny binary frames.
-> Multiple requests and responses could interleaved and sent simultaneously over a single TCP connection.
###### ADDITIONAL FEATURE
-> ==**Header Compression** (HPACK) to reduce bandwidth, and Server Push (the server sending assets to the browser before the browser even asks for them)==
##### HIDDEN FLAW
It fixed HoL blocking at application layer, but introduced it at the transport layer.
-> because it relies on a single TCP connection, if **one single packet is dropped at transmit  TCP halts everything to retransmit that missing packet.

#### HTTP/3 - TRUE SPEED via QUIC (2020+)
-> It dropped TCP completely and moved to a new transport protocol called **QUIC** which runs on top of UDP.
-> **QUIC - QUICK UDP INTERNET CONNECTIONS**

Traditional protocol stack:
==HTTP -> TLS (Transport layer security) -> TCP -> IP==

QUIC PROTOCOL STACK:
==HTTP/3 -> QUIC (includes TLS) -> UDP -> IP==

-> QUIC uses UDP instead of TCP because UDP is simpler and allows QUIC to implement reliability, congestion control, and encryption in user space without OS- level TCP limitations.
-> Integrates **TLS 1.3** directly into the protocol.
-> All QUIC connections are encrypted by default.
-> **SUPPORTS:**
1. 1-**RTT** (ROUND TRIP TIME) connection establishment.
2. **0-RTT** allows a client to send data immediately when reconnecting to a previously known server.
-> **Connection Migration** Connection remains active even if IP address changes. Useful when switching from Wi-Fi to Mobile data.
-> implemented mostly in user space.
-> Easier to update than kernel-based TCP implementation.
-> QUIC identify connections by using Connection IDs instead of relying solely on IP addresses and port numbers.

#### IDEMPOTENCY -> same result even after applying function/ operation multiple times

==An HTTP method is idempotent if making multiple identical requests has the exact same effect on the server state as making a single request.==


| **HTTP METHOD** | **IDEMPOTENT?** | WHAT HAPPENS IF REPEATED ?                                                                                                                                             |
| --------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET             | YES             | just fetching the data. Fetching user profile 10 times does not change anything in database.                                                                           |
| PUT             | YES             | Used to update/replace a resource entirely. If u send Set balance = 100 five times, the balance remains 100 only.                                                      |
| DELETE          | YES             | Used to delete resource. The first call deletes it. The next 4 calls return 404 Not Found, but the state of the server has not changed further; the item is still gone |
| POST            | NO              | Used to create resource or submit data. If you hit a payment or submit form 3 times, you might accidently create 3 separate orders or get charged 3 times.             |
| PATCH           | NO              | Used for partial updates. It can be non-idempotent depending on implementation.                                                                                        |


## TOPIC B : DNS (DOMAIN NAME SYSTEM)

   ![[Pasted image 20260608160337.png]]

DNS is not a single giant server; it is a distributed hierarchical tree structure.
-> When your machine needs to find an IP address, it queries upto 4 types of servers.

#### CORE DNS RECORD TYPES
When you buy a domain name, you configure a DNS zone file filled with different record entries. 
1. **A RECORD (Address)** Maps a hostname directly to an IPv4 address 
     example: google.com -> 142.250.190.46
2. **AAAA RECORD (Quad-A):** Maps a hostname to an IPv6 address.
      example: google.com -> 2607:f8b0:805::200e
3. **CNAME (CANONICAL NAME):** Maps an alias name to another true domain name. Used when you want multiple subdomains to point to the same location.
4. **MX RECORD(MAIL EXCHANGER):** Specifies the mail servers responsible for receiving email on behalf of domain name.

## DNS CACHING & TTL
To ensure DNS does not crawl to a halt from billions of global lookups, DNS relies heavily on ==caching==.

Before a query even leaves your computer, your browser checks its own local cache, followed by your Operating system local cache.
-> If missing, it hits DNS Recursor, which also maintains a massive cache of recently resolved domains.
#### TTL (TIME TO LIVE)
-> Every dns has a TTL value (in seconds). 
-> It dictates exactly how long a recursive resolver or your OS is allowed to save that record in its cache before it expires and must be fetched freshly from authoritative server.

### HOW HTTP AND DNS WORKS TOGETHER
1. You use DNS to look up the IP Address of lets say shopping.com
2. Once your browser connects to that IP address, you use HTTP to ask for the website's homepage, look at images, and send your payment information securely.



