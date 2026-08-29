# Networking Essentials (System Design) — Elaborate Edition

**🔗 Source article:** https://www.hellointerview.com/learn/system-design/core-concepts/networking-essentials
_Hello Interview — System Design in a Hurry → Core Concepts → Networking Essentials_

> Style: deep, teaching-level notes — the *why* behind everything, analogies, diagrams, worked examples, and interview guidance. This is a **complete** capture of the article (every section, example, table, and code snippet), not a summary.

### Full section map (nothing skipped)
1. Networking 101 — why layers (OSI); L3/L4/L7; User Space vs Kernel Space
2. Walkthrough of a simple web request (DNS → TCP 3-way handshake → HTTP → 4-way teardown)
3. Network Layer (IP) — addressing, DHCP, public IPs, RIR
4. Transport Layer — UDP, TCP, when to choose, QUIC, comparison table
5. Application Layer — HTTP/HTTPS · REST · GraphQL · gRPC · SSE · WebSockets · WebRTC (STUN/TURN)
6. Load Balancing — vertical vs horizontal; client-side (Redis Cluster, DNS); dedicated L4 vs L7; health checks; algorithms; real-world implementations
7. Deep dives — Regionalization & latency (CDN, regional partitioning); Failures (timeouts, retries, backoff+jitter, idempotency, circuit breakers)
8. Wrapping up — 4 pillars + hands-on practice (Wireshark, Network Link Conditioner)
9. Numbers to burn in
10. Self-Test Q&A (14 tricky questions)

---

## How to read this file
Each major topic is structured as: **Intuition → How it actually works → Why it matters / tradeoffs → When to use in interviews.**
The bottom has a **Numbers** table and a **Self-Test Q&A** bank (tricky, understanding-level) for spaced-repetition quizzing.

Memory spine (memorize this first): **"Little Rabbits Take Apples, Leaving Roots Fresh"**
1. **L**ayers · 2. **R**equest walkthrough · 3. **T**ransport (TCP/UDP) · 4. **A**pplication protocols · 5. **L**oad balancing · 6. **R**egionalization · 7. **F**ailures

---

## 1. Networking 101 — Why We Have Layers

### Intuition
Networking is fundamentally about **connecting independent devices so they can communicate**. The field is enormous, so the industry tamed it with a **layered architecture** (the OSI model). Each layer is an **abstraction** that hides the messy details of the layer below it.

**Analogy:** When you call `open("file.txt")` in code, you don't think about magnetic domains on a disk platter or voltage levels — the OS gives you a clean abstraction. Networking layers do the same: when you request a webpage you don't reason about which voltages represent a `1` or a `0` on the wire. You just use the next layer down.

Each layer only needs to understand the interface of the layer directly beneath it. This is what makes the internet tractable for application developers.

### The three layers that matter for interviews
Real networking has 7 OSI layers, but interviews revolve around **three**:

```
┌─────────────────────────────────────────────┐
│  L7  Application   HTTP, DNS, WebSockets,     │  ← where devs live (User Space)
│                    WebRTC, gRPC …             │
├─────────────────────────────────────────────┤
│  L4  Transport     TCP, UDP, QUIC             │  ← reliability, ordering, flow control
├─────────────────────────────────────────────┤
│  L3  Network       IP  (+ InfiniBand etc.)    │  ← routing & addressing (packets)
└─────────────────────────────────────────────┘
```

**L3 — Network Layer (IP).** Responsible for **routing and addressing**. It chops data into **packets**, forwards them hop-by-hop between networks, and offers **best-effort delivery** to any destination IP. "Best-effort" means: it will *try*, but makes **no promise** about arrival, order, or duplicates. (Other L3 protocols exist — e.g. InfiniBand for massive ML training clusters — but IP dominates interviews.)

**L4 — Transport Layer (TCP / UDP / QUIC).** Provides **end-to-end** services *between applications*. Think of it as the layer that adds the guarantees IP lacks: **reliability, ordering, flow control**. This is where you choose your reliability/latency tradeoff.

**L7 — Application Layer (HTTP, DNS, WebSockets, WebRTC…).** The protocols developers actually design against. They build on top of TCP (or UDP for WebRTC) to give structured, purpose-built abstractions for web data.

### Why the split matters: User Space vs Kernel Space
- **L7 is processed in "User Space"** — flexible, easy to modify, where your app code runs.
- **L4/L3 are processed in the OS kernel ("Kernel Space")** — hard to change, but can be *extremely* efficient.

**Takeaway:** the higher you go, the more flexible and feature-rich but slower; the lower you go, the faster but more rigid. This tradeoff echoes throughout system design.

---

## 2. Walkthrough: What Actually Happens in a Simple Web Request

This is the classic "what happens when you type a URL and hit Enter?" question. Even though the low-level details rarely get tested directly, understanding them builds the mental model everything else rests on.

```
CLIENT                                          SERVER
  │                                                │
  │ 1. DNS: hellointerview.com  ──▶  32.42.52.62   │   (resolve name → IP)
  │                                                │
  │ 2. TCP 3-way handshake                         │
  │      ── SYN ───────────────────────────────▶  │   "can we talk?"
  │      ◀────────────────────────── SYN-ACK ───  │   "yes, can you hear me?"
  │      ── ACK ───────────────────────────────▶  │   "yes — connected"
  │                                                │
  │ 3. HTTP GET /  ─────────────────────────────▶ │
  │ 4.                                (processing) │   ← usually the only latency
  │ 5. ◀───────────────────────── HTTP 200 + HTML  │      SWEs think about
  │                                                │
  │ 6. TCP 4-way teardown                          │
  │      ── FIN ───────────────────────────────▶  │
  │      ◀────────────────────────────── ACK ───  │
  │      ◀────────────────────────────── FIN ───  │
  │      ── ACK ───────────────────────────────▶  │
```

**Step by step:**
1. **DNS Resolution** — the human-readable domain (`hellointerview.com`) is converted to an IP address (`32.42.52.62`). DNS is essentially the internet's phone book.
2. **TCP Handshake (3-way)** — before *any* data, the two sides establish a connection:
   - **SYN** (synchronize): client asks to open a connection.
   - **SYN-ACK**: server acknowledges and asks back.
   - **ACK**: client confirms → connection established.
   This is a **full round trip of pure setup** before a single byte of real data moves.
3. **HTTP Request** — now the client sends `GET /`.
4. **Server Processing** — server builds the response. *This is the piece most engineers actually control and optimize.*
5. **HTTP Response** — server returns status + content.
6. **TCP Teardown (4-way)** — **FIN → ACK → FIN → ACK** closes both directions cleanly.

### Three lessons to carry forward
1. **Abstraction is a gift.** As an app developer you get to assume reliable, ordered delivery (TCP handles retransmits/ordering) and never worry about physically locating a server (DNS + IP routing handle that).
2. **One "request" = many packets.** A single conceptual request/response involves many packets and mini-exchanges, each adding latency. **The higher up the stack, the more latency and processing.** (This directly motivates the load-balancer discussion later.)
3. **A connection is *state* both sides must hold.** Unless you use **HTTP keep-alive** or **HTTP/2 multiplexing**, you repeat this whole setup **for every request** — a real, measurable overhead. This is the seed of everything about **persistent connections** (WebSockets, SSE, real-time systems).

> **Interview note:** the deep TCP-handshake trivia is *less* common in modern BigTech interviews, but the *implications* (setup cost, statefulness) show up constantly.

---

## 3. Network Layer (IP) in a Bit More Depth

- IP's job: **routing and addressing**. Every node needs an IP, usually assigned at boot by a **DHCP server**.
- IP addresses are **arbitrary labels** — they only mean something because we advertise them. On a private network you can assign whatever you like.
- For traffic to find you **on the public internet**, you need a **routable public IP** allocated by an **RIR** (Regional Internet Registry). The internet's routing backbone is *optimized* to know where public IP blocks live.
  - Fun fact: any address starting with `17.x.x.x` belongs to **Apple** — the backbone knows to route those packets to Apple's routers.

That's all we need from L3 — the interesting tradeoffs live one layer up.

---

## 4. Transport Layer — TCP vs UDP (the core L4 decision)

Three protocols exist (TCP, UDP, QUIC), but the **real interview decision is TCP vs UDP**.
- **QUIC** ≈ "a modernized, faster TCP" (built on UDP under the hood, powers HTTP/3). Treat it as *a better TCP with less universal adoption*. Mentioning QUIC/HTTP/3 can impress performance-minded interviewers, but usually spend your energy elsewhere.

### UDP — "the machine gun" (spray and pray)

**Intuition:** UDP adds almost nothing on top of IP. It's **connectionless** and **fast**. You fire packets ("datagrams") into the void and hope they arrive. If you receive one, you can see its source IP+port and destination IP+port — everything else is an opaque binary blob.

**Key characteristics:**
1. **Connectionless** — no handshake, no setup cost.
2. **No delivery guarantee** — packets can vanish with zero notification.
3. **No ordering** — packets can arrive out of order.
4. **Lower latency** — minimal overhead (tiny 8-byte header).

**Why would anyone accept "packets may just disappear"?** Because for some workloads **speed beats perfection**:
- Live video streaming, online gaming, VoIP, DNS lookups, high-volume telemetry/logs.
- **VoIP example:** if one packet drops, the client just skips it — a tiny audio hiccup — and the conversation stays intelligible. That's *far* better than pausing the whole call to retransmit a now-useless fragment and flooding the network with ACKs.

**Browser caveat:** browsers don't broadly support UDP *except* via WebRTC. So if your design leans on UDP (e.g. spamming hearts/reactions in Facebook Live), plan a fallback: app users might get a real-time UDP stream while **browser users get a slower, batched HTTP stream** you spread out in the UI.

### TCP — "the workhorse"

**Intuition:** TCP is the reliable, grown-up protocol that powers most of the internet. It sets up a connection (the 3-way handshake), then guarantees your data arrives **completely, correctly, and in order**.

The connection is called a **"stream"** and is a **stateful connection** between client and server. Because it's a single ordered stream, **two messages sent on the same connection arrive in the same order**. TCP makes the receiver **ACK** every message; if an ACK doesn't come back, TCP **retransmits** until it does.

**Key characteristics:**
1. **Connection-oriented** — dedicated connection before data flows.
2. **Reliable delivery** — in order, error-checked, nothing silently lost.
3. **Flow control** — prevents a fast sender from overwhelming a slow receiver.
4. **Congestion control** — backs off when the network is congested, preventing network collapse.

**Use TCP for:** basically **everything where UDP isn't a clear win** — data integrity matters.

### The deep reason UDP wins for *live* media (common interview trap)
It's not just "less overhead." TCP's reliability is **actively harmful** for real-time media:
- **Retransmitted data is stale.** If a packet drops at second 5, TCP re-requests it — but it arrives at second 6, by which point that audio/video frame is useless. You'd rather skip it.
- **Head-of-line (HOL) blocking.** TCP guarantees ordering, so a single lost packet forces TCP to **hold back every packet behind it** until the missing one is recovered → the whole stream **freezes/buffers**. With UDP each packet is independent, so a loss is a momentary blip, not a stall.

**Crisp framing:** *"For live media, a retransmitted packet arrives too late to matter, and TCP's in-order guarantee causes head-of-line blocking that freezes the stream. UDP's drop-and-continue trades a minor glitch for low, consistent latency."*

### When to choose which (interview guidance)
- **Default to TCP** — so standard that you often don't even need to say it out loud.
- **Earn points by justifying UDP** (and not fumbling the details). Reach for UDP when:
  - Low latency is critical (real-time, gaming)
  - Some data loss is acceptable (media streaming)
  - High-volume telemetry/logs where occasional loss is fine
  - You don't need browsers (or you have a fallback)
- **Hybrid is common:** a video-conferencing app uses **TCP/HTTP for signaling + auth** but **UDP/WebRTC for the audio/video** streams.

### TCP vs UDP — reference table

| Feature | UDP | TCP |
|---|---|---|
| Connection | Connectionless | Connection-oriented |
| Reliability | Best-effort (may drop) | Guaranteed delivery |
| Ordering | None | Maintains order |
| Flow control | No | Yes |
| Congestion control | No | Yes |
| Header size | 8 bytes | 20–60 bytes |
| Speed | Faster | Slower (overhead) |
| Use cases | Streaming, gaming, VoIP, DNS | Everything else |

---

## 5. Application Layer Protocols (the heart of most designs)

This is where most developers spend their time. Mnemonic for the big ones:
**"Really Good Gophers Send Waves Regularly"** → **R**EST · **G**raphQL · **g**RPC · **S**SE · **W**ebSockets · Web**R**TC (all riding on HTTP/transport).

The single most useful mental model — the **real-time push spectrum** (capability increases left → right):

```
Request/Response   →   SSE            →   WebSockets        →   WebRTC
 (REST, classic HTTP)   (server push,      (persistent,          (peer-to-peer,
                         one-way)           bidirectional)         UDP-based)
 client must ask        server streams     both sides push        clients talk
 for everything         to client          anytime                directly
```

---

### 5.1 HTTP / HTTPS — the foundation

**Intuition:** HTTP is a **request-response** protocol — client asks, server answers. Crucially it is **stateless**: each request is independent; the server keeps **no memory** of previous requests.

**Why stateless is good:** it minimizes the surface area that must hold state. In system design, **statelessness = easy horizontal scaling** (any server can handle any request). Most simple HTTP servers behave like a **pure function of the request parameters**.

**Anatomy of HTTP:**
1. **Request methods** — GET, POST, PUT, PATCH, DELETE
2. **Status codes** — 200, 404, 500, …
3. **Headers** — flexible key/value metadata
4. **Body** — the actual payload

**Request methods (with idempotency notes):**
- **GET** — read data. **Idempotent**, and carries no body.
- **POST** — send/create data. *Not* idempotent by default.
- **PUT** — update/replace a resource.
- **PATCH** — partial update.
- **DELETE** — remove data. **Idempotent** (deleting twice = still deleted).

**Status codes worth memorizing:**
- **2xx Success** — `200 OK`, `201 Created`
- **3xx Moved** — `301 Moved Permanently`, `302 Found (temporary)`
- **4xx Client Error** — `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `429 Too Many Requests`
- **5xx Server Error** — `500 Server Error`, `502 Bad Gateway`

**Headers as a design lesson — content negotiation:** headers are deliberately flexible key/values, a great example of designing an interface that survives unknown future needs. Example: the client sends `Accept-Encoding: gzip, br` to declare what compressions it supports; the server replies with `Content-Encoding: br` and picks the best one it has. This gives **backward compatibility + graceful degradation** for free.

**HTTPS = HTTP + TLS/SSL.** Encrypts traffic, defeating eavesdropping and man-in-the-middle (MITM) tampering. Any public website should use it, always.

> **Critical security nuance (very common interview + real-world bug):**
> HTTPS guarantees the request wasn't read or altered **by a third party in transit**. It does **NOT** guarantee the request contents are *honest*, because **the client itself generates the body** and could be the attacker.
> Classic bug: server reads `userId` from the request body and queries the DB with it. An attacker just sends `userId=<someone_else>` and reads arbitrary users' data — this is **IDOR (Insecure Direct Object Reference)**.
> **Rule:** derive identity from the **authenticated token/session**, never from a value the caller can set. Validate everything server-side. Encryption ≠ trust.

---

### 5.2 REST — the sensible default for APIs

**Intuition:** REST models your system as **resources** (nouns — like DB tables or files) that you manipulate with HTTP **verbs**. It's simple, universally understood, and easy to reason about.

If you've done requirements/entity modeling first, your **Core Entities usually map directly onto REST resources** — a nice shortcut.

**Examples:**
```
GET  /users/{id}          -> User                         (read one)
PUT  /users/{id}          -> User   { "username": ..., "email": ... }   (update)
POST /users               -> User   { "username": ..., "email": ... }   (create; server assigns id)
GET  /users/{id}/posts    -> [Post]                       (nested relationship)
```

**The key mental shift:** think in **resources + operations**, not function names. Engineers instinctively write `updateUser()` or `startGame()` — those are *operations*, not RESTful. Reframe:
- `updateUser` → `PUT /users/{id}`
- `startGame` → `PATCH /games/{id}` with `{ "status": "started" }`

**Where to use:** extremely broad. (ElasticSearch exposes a rich REST API for docs/indexes.) JSON is a bit inefficient to (de)serialize and REST isn't the *most* performant for very high throughput — but **most systems aren't bottlenecked on serialization**.

> **Interview guidance:** **default to REST**, just like you default to TCP. Only reach for GraphQL / gRPC / SSE / WebSockets when you have a *specific* need REST can't meet.

---

### 5.3 GraphQL — flexible data fetching

**The problem it solves.** Teams often split into frontend + backend. When the frontend needs a new page with REST, it faces three bad options:
- **(a) Under-fetching** — call many endpoints and stitch results (e.g. 1 call for a user list + 10 calls for each user's details). Many round trips → latency.
- **(b) Over-fetching** — build fat aggregation endpoints that return way more than needed "just in case." Slow, bloated, hard to maintain.
- **(c) A brand-new custom API per page** — a maintenance nightmare.

**How GraphQL fixes it.** The client sends a **query describing exactly the fields and nested objects it wants**, and the backend returns data shaped precisely that way — in one request.

```graphql
query GetUsersWithProfilesAndGroups($limit: Int = 10, $offset: Int = 0) {
  users(limit: $limit, offset: $offset) {
    id
    username
    profile { id fullName avatar }
    groups {
      id name description
      category { id name icon }
    }
    status { isActive lastActiveAt }
  }
  _metadata { totalCount hasNextPage }
}
```
Instead of many bespoke endpoints, the frontend writes one query and gets a tailor-made response.

**Where it shines:** frontends that **iterate fast**, and organizations where **multiple teams make wide queries over overlapping data**.

> **The misconception to avoid:** "GraphQL reduces over/under-fetching, so it's *always* more efficient." Misleading — it optimizes the **client/network** side but **pushes cost onto the backend**: resolving flexible, deeply-nested queries can require complex resolvers (sometimes the very bespoke code you were trying to avoid) and introduce latency and N+1 query risks. It's a **tradeoff**, not a free win.
>
> **Interview guidance:** GraphQL's benefits are *murky in interviews* because (1) interview requirements are **fixed**, not the moving-target frontends where GraphQL pays off, and (2) interviewers usually want to see you **optimize specific query patterns**, which GraphQL's generic resolver layer obscures. Bring it up **only** when the problem is explicitly about **flexibility / rapidly-changing / deliberately-uncertain requirements**.

---

### 5.4 gRPC — efficient internal service communication

**Intuition:** gRPC is Google's high-performance **RPC** (Remote Procedure Call) framework. You call a remote function almost like a local one. It rides on **HTTP/2** and serializes data with **Protocol Buffers** (protobuf) — think "JSON with a strict schema and a compact binary encoding."

**Why protobuf is efficient — a concrete size comparison:**
```protobuf
message User { string id = 1; string name = 2; }
```
- JSON (embeds field names every time): `{ "id": "123", "name": "John Doe" }` ≈ **40 bytes**
- Protobuf binary (skinny numeric tags + varint-encoded strings) ≈ **15 bytes**:
  `0A 03 31 32 33 12 08 6A 6F 68 6E 20 64 6F 65`

Less space on the wire **and** less CPU to parse.

**Service definitions** compile into client + server **stubs** in many languages:
```protobuf
message GetUserRequest  { string id = 1; }
message GetUserResponse { User user = 1; }
service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
}
```
gRPC also brings streaming, deadlines, and client-side load balancing. Bottom line: **binary, strongly-typed, and much faster than JSON-over-HTTP** — some benchmarks show ~**10x throughput**. Strong typing also catches errors at **compile time** rather than runtime.

**Where to use:** **internal service-to-service** communication, especially when performance is critical or latency is dominated by the network rather than server work.

> **Why NOT expose gRPC to public browsers:** browsers don't natively speak gRPC (you'd need a gRPC-Web proxy), and the binary tooling is less mature than plain JSON-over-HTTP for clients you don't control.
> **The hybrid pattern — "REST out, gRPC in":** REST/JSON for public/external APIs; gRPC internally between your own microservices. In many interviews REST for both is a fine starting point.
> **Warning:** don't hyper-optimize your RPC protocol before you've handled the *real* bottlenecks. Premature optimization is the root of all evil.

---

### 5.5 Server-Sent Events (SSE) — real-time push, server → client

**Intuition:** normally an HTTP response is **one blob you can only use after it fully arrives** — useless for pushing live events. SSE is a clever **hack on top of HTTP**: the server holds one response open and **streams many messages over time** through it.

**Normal HTTP (can't push):**
```json
{ "events": [ {"id":1,...}, {"id":2,...}, ... {"id":100,...} ] }   // usable only when complete
```

**SSE (each line is delivered and processed as it arrives):**
```
data: {"id": 1, "timestamp": "...", "description": "Event 1"}
data: {"id": 2, "timestamp": "...", "description": "Event 2"}
...
data: {"id": 100, "timestamp": "...", "description": "Event 100"}
```
It's still **one HTTP response over one TCP connection**, but delivered as many small chunks; the client reacts to each line immediately.

**The "hack tax" (real limitations):**
- **Connections can't stay open forever** — a server, load balancer, or middlebox proxy may close them. The SSE spec's **`EventSource`** object **auto-reconnects** and sends the **ID of the last message it received**; the server is expected to **resend messages missed** during the gap.
- **Misbehaving networks may batch** all SSE chunks into one response, defeating the streaming behavior.
- Many interviewers don't know these gotchas — but engineers who've *actually* implemented SSE carry the scars, and mentioning these signals real experience.

**Where to use:** clients need to receive **events/notifications as they happen**, one-directionally. Great example: keep bidders updated with the **current price in an auction**. **SSE is unidirectional (server → client only).**

---

### 5.6 WebSockets — real-time, bidirectional, persistent

**Intuition:** SSE only pushes server→client. Many apps need **both** directions in real time. WebSockets give you a **persistent, TCP-style, bidirectional** channel with **broad support (including browsers)**. Either side can **push at any time** — no waiting for a request.

**How it works:**
```
1. Client initiates a WebSocket handshake over HTTP (backed by a TCP connection)
2. HTTP "Upgrade" → the protocol switches; WebSocket takes over that TCP connection
3. Both client and server send binary messages freely, in both directions
4. The connection stays open until explicitly closed
```
- The **HTTP "upgrade"** mechanism is elegant: an existing TCP+HTTP connection changes its L7 protocol to WebSocket, letting you **reuse existing HTTP session info** (cookies, headers, auth) during setup.
- **You define the message format yourself** — WebSockets give you a raw binary channel but **don't dictate an application protocol**. In practice, **serialized JSON messages** are a great, simple choice, and defining them *is* effectively defining your service's API.

> **Infra caveat (bites people in practice):** every hop between client and server — firewalls, proxies, **load balancers** — must support WebSocket connections, or the upgrade breaks. This is why WebSockets pair with **L4 load balancers** (see §6).

**Where to use:** **high-frequency, persistent, bi-directional** communication — real-time apps, chat, multiplayer games, collaborative editors.

> **Interview guidance (important):** don't reach for WebSockets unless you can **justify the need**. If simple request/response works, or SSE's server-push is enough, WebSockets are **overkill** — and jumping to them unprompted is a classic "thumbs down." Stateful, long-lived connections are **expensive at scale** and force real accommodations in your design (connection management, sticky routing, memory per connection).

---

### 5.7 WebRTC — peer-to-peer (the odd one out)

**Intuition:** WebRTC enables **direct browser-to-browser** communication with **no intermediary server in the data path**. It's the **only** application-layer protocol here that uses **UDP** — perfect for **video/audio calling and conferencing**.

**Why P2P is hard:** most clients **refuse inbound connections** (security) and sit behind a **NAT** (Network Address Translation) device, so they aren't directly reachable. Left alone, most peers simply couldn't connect to each other. WebRTC solves this with helper servers:

- **Signaling server** — a central coordinator that tracks which peers are available and helps them **exchange connection info** (who wants to talk to whom, plus candidate addresses).
- **STUN** ("Session Traversal Utilities for NAT") — lets a client discover **its own public IP address and port** as seen from outside its NAT ("hole punching"), so it can then share that reachable address with a peer.
- **TURN** ("Traversal Using Relays around NAT") — a **relay fallback**: when a direct P2P connection can't be established, data is **bounced through a central server** to the peer.

**The connection dance (happy path):**
```
1. Peers connect to the SIGNALING server to discover each other + exchange info
2. Each peer hits a STUN server to learn its own public IP:port
3. Peers share that info via the signaling server
4. Peers open a DIRECT peer-to-peer connection and stream data
   (if this fails → fall back to a TURN relay)
```

> **"No server" is an oversimplification.** Servers are needed to **bootstrap** the connection (signaling discovery + STUN NAT traversal). Once connected, media flows **directly peer-to-peer** — *except* when **TURN** kicks in, in which case a server literally relays the data. So it's "no server in the **data path**, in the happy case."
>
> **Interview guidance:** keep WebRTC scoped to **audio/video calling & conferencing**. It's genuinely hard to get right and even great implementations drop connections; it's a **niche** tool. Most "collaborative" problems (e.g. Google Docs) actually use **WebSockets** — you usually need a central server to store/coordinate the document anyway. (An advanced alternative is WebRTC + **CRDTs** for direct peer updates, but that's rarely the right interview answer.) Candidates lose more time going down the WebRTC rabbit hole than they gain.

---

## 6. Load Balancing — How We Scale

### The scaling choice
- **Vertical scaling** — bigger servers. Modern hardware is *very* powerful, so prefer this when you can (fewer moving parts). See "Numbers to Know" for how much a single box can do.
- **Horizontal scaling** — more servers. **This is what interviews assume most of the time.**

But adding servers to a diagram is useless unless clients know **which** server to talk to. That routing problem is **load balancing**.

### 6.1 Client-Side Load Balancing
**Intuition:** the **client itself** decides which server to hit. It consults a **service registry/directory** for the list of available servers, then calls one directly. It periodically re-syncs (poll or push) as servers come and go.

**Upside:** very fast — **no extra network hop per request**. You only pay the (occasional) cost of syncing the server list.

**Example — Redis Cluster.** Cluster nodes run a **gossip protocol**, so **every node knows about every other node** (membership, status, which shard holds what). To use it: the client asks any node for the cluster layout + shard map, then **hashes the key** to find the right shard and talks to that node directly. Hit the wrong node? Redis replies **`MOVED`** to redirect you.

**Example — DNS (client-side LB in disguise).** A DNS resolver returns a **rotated list of IPs** for a domain; each client gets a different ordering (or set), so different clients hit different servers. This is also how you **avoid a single point of failure at the load balancer**: run **two LBs in different data centers/regions** and rotate between their IPs via DNS — if one dies, clients try the other.

**Where client-side LB works well:**
- **(1)** a small number of clients you **control** (easy to push updates) — e.g. the Redis Cluster client, or **gRPC's built-in client-side LB** for internal services; **or**
- **(2)** a large number of clients that can **tolerate slow updates** — e.g. DNS, where entries have a **TTL** and far-flung resolvers cache them, so **your updates can't propagate faster than the TTL**.
- Excellent for **internal microservices** (baked into gRPC). Many interviewers don't probe the lines between services, but if asked — mention client-side LB.

### 6.2 Dedicated Load Balancers
**Intuition:** a server/appliance sits **between** clients and backends and makes the routing decision. Clients don't need to know multiple servers exist. Cost: **one extra hop per request**. Benefit: **fast server-list updates** and **fine-grained routing control**.

They operate at different layers — and **which layer you pick has real consequences.**

```
        CLIENT
          │
          ▼
   ┌──────────────┐
   │ Load Balancer│
   └──────────────┘
     │    │     │
     ▼    ▼     ▼
   srv1  srv2  srv3
```

#### Layer 4 (L4) Load Balancers — transport level
- Route on **IP + port**; they **don't inspect packet contents**.
- Effect: **as if you randomly picked one backend and the client had a direct TCP connection straight to it.**
- Characteristics:
  - Maintain a **persistent TCP connection** client↔server
  - Fast/efficient (minimal inspection)
  - **Cannot** route based on application data
  - Chosen when raw performance is the priority
- Because the same backend handles the **entire TCP session**, L4 is **ideal for persistent-connection protocols — WebSockets.** Conceptually it's *as if there's a direct TCP pipe from client to backend* that higher layers ride on.
- **Where to use:** WebSockets and other persistent connections; high-performance apps with little app-layer processing needed.

#### Layer 7 (L7) Load Balancers — application level
- Understand app protocols (HTTP). Can **inspect the actual request content** (URL, headers, cookies) and make **smart routing decisions**.
- Mechanically, an L7 LB **terminates the incoming client connection and opens a *new*, separate connection to the backend** — so there are **two independent connections**, and the backend TCP connection is "just a pipe" for forwarding L7 requests.
- Characteristics:
  - Terminate client connections; create new ones to backends
  - Route by URL / headers / cookies
  - **More CPU** (packet inspection)
  - More flexibility and features
  - Best for HTTP traffic
- Examples: send `/api/*` to one fleet and web pages to another (like an **API Gateway**); or keep a user's requests on the same server via a **cookie** (sticky sessions).
- **Where to use:** HTTP-based traffic — which is *everything here except WebSockets*. (Some L7 LBs *can* support WebSockets, but **L4 is generally better for WebSockets**; L7 is better for HTTP patterns like long polling.)

> **The L4-vs-L7 trap to remember:** "a single end-to-end TCP connection between client and backend" is **true of L4, false of L7** (L7 terminates and re-opens). That persistent pass-through is exactly *why L4 suits WebSockets*.

#### Health Checks & Fault Tolerance
Load balancers don't just distribute traffic — they **monitor backend health** and stop routing to a dead/crashed server until it recovers. This **automatic failover** is what makes them essential for **high availability** — failures get routed around with no user intervention.
- **TCP health check** — simple/efficient: can the server accept a new connection?
- **L7 health check** — makes an HTTP request and expects success (e.g. `200` vs a `500` or no response indicating a crash).

#### Load Balancing Algorithms
A dedicated LB also lets you choose **how** traffic is distributed:
- **Round Robin** — sequentially cycle through servers.
- **Random** — pick a server at random.
- **Least Connections** — send to the server with the **fewest active connections**.
- **Least Response Time** — send to the **fastest-responding** server.
- **IP Hash** — hash the client IP to a server (gives **session persistence**).

**Rule of thumb:** Round Robin / Random are fine for **stateless** apps (new servers naturally start receiving traffic).
> For **persistent connections (SSE / WebSockets)**, prefer **Least Connections**. Why: those connections are **long-lived**, and Round Robin only balances *arrivals*, not *what's still open* — so one server gradually **accumulates all the live connections** and gets overloaded. Least Connections tracks the **live count** and self-balances.

#### Real-World Implementations
- **Hardware** — F5 Networks BIG-IP (can handle **hundreds of millions of RPS**).
- **Software** — HAProxy, NGINX, Envoy (more limited than dedicated hardware).
- **Cloud** — AWS ELB/ALB/NLB, Google Cloud Load Balancing, Azure Load Balancer.
- Scaling the LB itself is rarely part of a SWE interview; if throughput is enormous, "use a hardware load balancer" is a clean out.

---

## 7. Common Deep Dives & Real-World Challenges

### 7.1 Regionalization and Latency

**The setup.** Global services distribute servers worldwide — typically **multiple data centers per region** (Amazon calls them **Availability Zones**, so one building's burst pipe doesn't sink the whole service), replicated across cities/continents.

**The problem — physics.** Distance = latency, and you can't beat the speed of light.
- Light in fiber travels at ≈ **⅔ of light speed ≈ 200,000 km/s**.
- New York ↔ London is ~5,600 km one way. A **round trip** (~11,200 km) has a **theoretical floor ≈ 56 ms** — *before* any processing, routing, or queuing.
- Same-region latency is **<1 ms**; cross-continent is **>80 ms** in practice.
- **No server optimization beats this** — so the fix is **move data closer to users**, not "make the central server faster."

**The guiding principle — data locality.** Keep the data needed to answer a query **(a) close together** and **(b) close to the user**. If users are in LA but web servers are in NY, every DB query eats tens of ms of network latency before any processing.

Two main strategies:

#### Content Delivery Networks (CDNs)
- Networks of servers ("**edge locations**") in hundreds/thousands of cities. If an edge server can answer, the user gets **lightning-fast** responses — "the data is just up the road."
- Powered by **caching**, so it's ideal for **static / cacheable content**: images, video, assets.
- In interviews: reach for a CDN when data is **cacheable and queried globally** (e.g. caching Facebook search results) — it **lowers latency AND offloads the backend**.

#### Regional Partitioning
- Partition data **by region** so each region's infrastructure holds only the data relevant to it.
- **Uber example:** a rider in Miami never needs a driver currently in New York. Globally there are millions of riders/drivers, but within one city only thousands. Mirror that in the architecture: each region gets its **own DB on distinct servers**, with request servers **co-located** next to the DB they query. Regional queries are fast; local DB access is very fast.

> **CDN vs regional partitioning — the decision rule:**
> - **Static + global + cacheable** (e.g. product images identical for everyone) → **CDN**. (A CDN caches the file at an edge near the user, killing the transatlantic hop.)
> - **Region-specific relational data + locality of queries/writes** (e.g. Uber rides) → **regional partitioning**.
> Using the wrong one is a classic misstep: regional DBs don't help serve identical static assets; a CDN doesn't help region-scoped writes.

### 7.2 Handling Failures and Fault Modes

**Mindset:** plan for **both** server failures (crashes, bit flips from cosmic rays, power loss) **and** network failures (cut cables, dead routers, dropped packets). The most dangerous assumption in distributed systems is **"the network is reliable."** It isn't. **Always assume calls will fail, be delayed, or return unexpected results.**

#### Timeouts and Retries
- **Timeout:** if a request exceeds the time you expect, **give up and try again** rather than hanging forever.
- **Retry:** great for **transient** failures (a briefly slow/overloaded server). But retries are only safe if the operation is **idempotent** (see below).

#### Backoff (and why jitter matters)
- Naive retries can make things **worse** — hammering a struggling service repeatedly.
- **Backoff:** wait before retrying; wait **progressively longer** on repeated failures (**exponential backoff**) to give the system room to recover.
- **Jitter (randomness):** don't let all clients retry at the **same instant**. Without jitter, failed requests **synchronize** and slam the server in waves "like a jackhammer." Jitter spreads retries randomly over time.
- **Interview magic phrase:** *"retry with exponential backoff"* — and for senior levels, add **jitter**.

#### Idempotency
- Retries are dangerous when the operation has **side effects**. Retry a "$10 charge" three times → you charge **$30**. Ouch.
- **Idempotent** = calling it multiple times yields the **same result** as calling it once.
- **GET** is naturally idempotent (reading doesn't change state). For **writes**, introduce an **idempotency key** — a unique ID per logical request (e.g. `userId + date`, or a UUID).
- Server behavior with the key:
  - **Already processed** → skip the side effect, **return the stored original result** (friendly) or an error saying it exists (less friendly).
  - **In progress** → wait and return the same result to all callers.
  - **New key** → process once, record the key + result.
- Net effect: a retry becomes a **safe no-op** that returns the original outcome — **no double charge**.

#### Circuit Breakers (preventing cascading failures)
**The problem:** when a dependency (say a DB) goes down, "just keep retrying" can *prevent it from ever recovering*. A DB rebooting **one instance at a time** gets **buried by the retry firehose** the instant it flickers to life ("**thundering herd**") and falls over again → you're stuck. Even backoff + jitter only **spread** retries; they don't **stop** them, and the aggregate volume against a fragile, half-started service is still fatal.

**The fix — the circuit breaker state machine** (named after electrical breakers):
```
        failures < threshold
      ┌───────────────────────┐
      ▼                       │
  ┌────────┐  failures exceed  ┌────────┐
  │ CLOSED │ ────threshold───▶ │  OPEN  │  ← requests FAIL FAST immediately
  └────────┘                   └────────┘     (no call attempted → dependency
      ▲                            │            gets total silence to recover)
      │ test request               │ after timeout
      │ succeeds                   ▼
      │                       ┌───────────┐
      └────────────────────── │ HALF-OPEN │  ← allow ONE probe request
         test fails → OPEN    └───────────┘
```
1. **Closed** (normal) — requests flow; the breaker counts failures.
2. Failures cross a threshold → **trip to Open**.
3. **Open** — requests **fail fast** without even attempting the call. *This silence is what backoff alone can't provide* and is what lets the dependency recover.
4. After a timeout → **Half-Open**.
5. **Half-Open** — let **one test request** through: success → back to **Closed**; failure → back to **Open** (wait again).

**Benefits:** fail fast (don't wait on timeouts), reduce load on struggling services, self-healing (auto-probe recovery), better UX (fast fallback instead of a hanging UI), and overall system stability (one failure doesn't cascade).
**Where to use:** external 3rd-party API calls, DB connections/queries, service-to-service calls in microservices, and any network call that could fail or become slow.

---

## 8. Wrapping Up — the 4 pillars

1. **Basics** — IP addressing, DNS, the TCP/IP model.
2. **Protocols** — TCP vs UDP; HTTP/HTTPS; WebSockets; gRPC.
3. **Load balancing** — client-side vs dedicated (L4/L7), algorithms, health checks.
4. **Practical realities** — regionalization (CDN, partitioning) and failure patterns (timeouts, backoff+jitter, idempotency, circuit breakers).

Networking touches **every** system property — latency, throughput, reliability, security. In interviews, **justify each choice by the specific requirements**; there's rarely one right answer, and interviewers want to see **tradeoff reasoning**, not memorized defaults.

### Hands-on practice (build real intuition)
- **Wireshark** — capture live traffic on your machine and watch the whole protocol stack in action.
- **Mac Network Link Conditioner** (via XCode) — simulate latency and packet loss; see how apps degrade under a nasty cell connection. You'll find surprises (and bugs).

---

## 9. Numbers to Burn In

| Anchor | Value |
|---|---|
| UDP / TCP header size | 8 B / 20–60 B |
| NY ↔ London latency | ~56 ms theoretical min · >80 ms real · <1 ms same-region |
| Speed of light in fiber | ~200,000 km/s (≈ ⅔ c) |
| gRPC vs JSON | ~10x throughput; ~15 B vs ~40 B for the same object |
| TCP handshake / teardown | 3-way (SYN / SYN-ACK / ACK) · 4-way (FIN / ACK / FIN / ACK) |
| Status codes | 2xx OK(200)/Created(201) · 3xx 301/302 · 4xx 401/403/404/429 · 5xx 500/502 |

---

## 10. Self-Test Q&A (tricky, understanding-level)

**Q1. Video conferencing uses TCP for signaling but UDP for media — why UDP for media?**
Speed > reliability; occasional loss is tolerable. Deeper: TCP **retransmits stale frames** (useless once late) and its **in-order guarantee causes head-of-line blocking** (one lost packet freezes the whole stream). UDP drops-and-continues → minor glitch, low consistent latency.

**Q2. Why does opening a new TCP connection per HTTP request hurt, and how to avoid it?**
Cost = the **TCP 3-way handshake** (plus a **TLS handshake** over HTTPS) — full round trips of pure setup **before any data**, on every request. Avoid via **HTTP keep-alive** (reuse the connection) or **HTTP/2 multiplexing** (many concurrent streams over one connection).

**Q3. API reads userId from the body over HTTPS — teammate says "encrypted, so it's safe." Right?**
No. HTTPS protects data **in transit** from third-party tampering/eavesdropping, but the **client controls the body**, so an attacker sends `userId=victim` (**IDOR**). Authorize against the **authenticated identity** (token/session); never trust caller-supplied IDs. Encryption ≠ trust.

**Q4. Is POST idempotent? How to make a $10 payment POST safe to retry?**
Not inherently — naive retries double-charge. Use an **idempotency key**: the client tags the request; the server records processed keys and **dedupes**, returning the original result on retry instead of charging again.

**Q5. "GraphQL reduces over/under-fetching, so it's always more efficient." Why misleading? Why do interviewers dislike it?**
It optimizes client/network but **shifts cost to the backend** (complex resolvers, latency, N+1 risk) — a tradeoff, not a free win. In interviews requirements are **fixed** (its flexibility payoff doesn't apply) and interviewers want **specific query-pattern optimization**, which GraphQL obscures. Use only when flexibility/uncertainty is the explicit ask.

**Q6. Why not expose gRPC to browsers? What's the hybrid pattern?**
Browsers don't natively support gRPC and its binary tooling is immature for public clients. **"REST out, gRPC in"** — REST/JSON externally, gRPC internally for service-to-service.

**Q7. "SSE is server push over HTTP, so I'll build chat with it." Problem?**
SSE is **unidirectional** (server → client only). Chat is bidirectional and high-frequency → use **WebSockets**.

**Q8. WebRTC is "peer-to-peer, no server" — why signaling + STUN + (sometimes) TURN?**
Signaling = peer discovery + exchange connection info. STUN = discover **your own** public IP/port (NAT traversal). TURN = **relay** fallback when direct P2P fails. "No server" means no server in the **data path** in the happy case — but servers bootstrap it, and TURN relays data when P2P fails.

**Q9. T/F: an L7 LB keeps a single end-to-end TCP connection client↔backend.**
**False.** L7 **terminates** the client connection and **opens a new one** to the backend (two connections) → enables content-based routing. **L4** passes through and pins one client↔backend TCP connection — which is why **L4 suits WebSockets**.

**Q10. WebSocket servers behind an L7 LB using round-robin — what's wrong?**
(a) L7 fights persistence → use **L4**. (b) Round-robin balances *arrivals*, ignoring that connections **persist**, so one server accumulates them → use **Least Connections** (routes by live connection count, self-balances).

**Q11. DNS LB: removed a dead server but clients still hit it for minutes. Why? How does this help LB failover?**
**TTL caching** — resolvers/clients cache the record until the TTL expires; updates can't propagate faster than the TTL. The same behavior enables failover: put **multiple LB IPs** in the record so clients hold a list and auto-fail-over to a healthy LB (no single point of failure).

**Q12. us-east-1 service, London user, slow static product images — CDN or regional partitioning?**
**CDN** — images are static, cacheable, and global; edge caching near London kills the transatlantic hop and offloads the origin. Regional partitioning is for region-specific relational data (locality of queries/writes) and doesn't help identical static assets.

**Q13. Why can't NY↔London beat ~56 ms?**
Light in fiber ≈ ⅔ c ≈ 200,000 km/s; ~5,600 km one way → ~56 ms round-trip floor, before any processing. This drives **geo-distribution** (CDN/partitioning) instead of "faster servers."

**Q14. DB crashed, retries + backoff in place — how can retries still prevent recovery? Fix?**
A cold-booting DB gets buried by the retry backlog ("**thundering herd**"); backoff/jitter only **spread** retries, they don't **stop** them. Fix = **circuit breaker**: **Closed → Open (fail fast, zero traffic) → Half-Open (one probe) →** Closed/Open. The enforced silence lets the DB recover.
