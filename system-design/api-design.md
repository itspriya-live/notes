# API Design (System Design) — Elaborate Edition

**🔗 Source article:** https://www.hellointerview.com/learn/system-design/core-concepts/api-design
_Hello Interview — System Design in a Hurry → Core Concepts → API Design_

> Style: deep, teaching-level notes — the *why* behind everything, analogies, diagrams, worked examples, and interview guidance. This is a **complete** capture of the article (every section, example, and code snippet), not a summary.

### Full section map (nothing skipped)
1. API Types — REST vs GraphQL vs RPC (how to choose)
2. REST — resource modeling, HTTP methods & idempotency, passing data (path/query/body), returning data (status codes)
3. GraphQL — the problem it solves, how it works, when to use, schema design, the N+1 problem, field-level authz
4. RPC — action-oriented model, gRPC + Protocol Buffers, type safety, when to use
5. The 9 API Design Principles
6. Common API Patterns — pagination (offset vs cursor), filtering/sorting, idempotency keys, error envelopes, versioning
7. Security — authN vs authZ, API keys vs JWT, RBAC, rate limiting
8. Conclusion — interview time-budget

> **Interviewer reality check (stated up front in the article):** most interviewers do NOT care about perfect API design. They want a *reasonable* API, then move on to the harder parts. **Spend ≤5 minutes on APIs.** API design matters more for **frontend/product roles** and **junior roles** (less distributed-systems expectation, so more time on APIs).

---

## 1. API Types — Choosing the Protocol

In an interview you'll typically pick between **three** protocols:

```
                    ┌─────────────────────────────────────────────┐
   Standard CRUD?   │  REST   → default. HTTP verbs on resources.  │
                    ├─────────────────────────────────────────────┤
   Diverse clients, │  GraphQL→ single endpoint, client picks the  │
   flexible fetch?  │           exact fields it needs.             │
                    ├─────────────────────────────────────────────┤
   Internal, perf-  │  RPC    → action-oriented, binary (gRPC).    │
   critical svc→svc?│           Think functions, not resources.    │
                    └─────────────────────────────────────────────┘
   Real-time push?  → NOT a traditional API → WebSockets / SSE (persistent connections)
```

1. **REST (Representational State Transfer)** — standard HTTP methods (GET/POST/PUT/DELETE) on resources identified by URLs. Maps naturally to CRUD + HTTP semantics. **This is your default.**
2. **GraphQL** — single endpoint + query language; clients specify exactly what data they need. Trigger phrases: *"mobile needs different data than web," "avoid over-fetching/under-fetching," "flexible data fetching," "frontend iterates fast."*
3. **RPC (Remote Procedure Call)** — e.g. gRPC, binary serialization over HTTP/2. Action-oriented (`checkPermission(userId, resource)`). Trigger phrases: *"microservices," "internal APIs," "high performance," "polyglot."*

> **Default to REST** unless you have a specific reason not to. "I'll use REST APIs" and move on is a perfectly good interview line.
> **Real-time features** (chat, notifications, live updates) need **WebSockets or SSE** — persistent connections, not traditional request/response APIs (see Networking Essentials / Real-time Updates).

---

## 2. REST — the default, so spend most time here

### 2.1 Resource Modeling
**Foundation of good REST design = identify resources correctly.** If you followed the delivery framework, **resources = your core entities.**

Ticketmaster example — core entities (events, venues, tickets, bookings) map to resources:
```
GET  /events                 # all events
GET  /events/{id}            # a specific event
GET  /venues/{id}            # a specific venue
GET  /events/{id}/tickets    # available tickets for an event
POST /events/{id}/bookings   # create a booking for an event
GET  /bookings/{id}          # a specific booking
```

**Two rules that matter:**
1. **Resources are *things*, not *actions*.** Model what *exists* (events, bookings), not what users *do* (book, purchase). If you write `/getUserBookings`, `/book`, `/processPayment` — stop, find the noun, and let the HTTP method be the verb (`GET /users/{id}/bookings`, `POST /events/{id}/bookings`, `POST /payments`).
2. **Resources are plural nouns** — `bookings`, `events`, `tickets`. Minor, but easy points.

**Relationships — nested path vs query param (the decision rule):**
- **Nested path / path parameter** when the relationship is **REQUIRED** to make the query meaningful: `/events/{id}/tickets` — you always must say whose tickets.
- **Query parameter** when the filter is **OPTIONAL** (one of many): `/tickets?event_id=123&section=VIP` — you might want all tickets, or filtered by event, or by event+section.
> Rule: **required to identify → path; optional filter → query.**

### 2.2 HTTP Methods & Idempotency
| Method | Purpose | Idempotent? |
|---|---|---|
| **GET** | retrieve, no changes | ✅ yes (safe too) |
| **POST** | create new resource (server assigns id) | ❌ no — repeated calls create duplicates |
| **PUT** | replace entire resource (or create) | ✅ yes — same data → same final state |
| **PATCH** | partial update | ⚠️ **depends on implementation** |
| **DELETE** | remove resource | ✅ yes — stays deleted |

**Idempotency = repeated identical requests leave the server in the SAME final state.** (It is NOT about the response code being identical.)

**PATCH nuance:** idempotent only if the operation is an **absolute set** (`{"email": "x@a.com"}` → "set email to X"). **Not** idempotent if it **accumulates** (`append to list`, `increment balance`) — each call changes state further.

**DELETE gotcha:** first call returns `204`, second returns `404`. **Still idempotent** — the *state* (resource deleted) is unchanged; only the status code differs to describe reality.

**Why idempotency matters:** networks fail → clients retry. GET/PUT/DELETE are safe to retry blindly; **POST and accumulating PATCH need protection** (→ idempotency keys, §6.3).

### 2.3 Passing Data to APIs — three places, three roles
- **Path parameters** = *structural* — identify **which** resource. Required. `/events/123`.
- **Query parameters** = *modifiers* — filter/sort/paginate, change **how** the endpoint behaves. Optional. `/events?city=NYC&date=2024-01-01`, `/events?page=2&limit=20`. (First param after `?`, rest joined by `&`.)
- **Request body** = *payload* — the actual data to create/update; complex/large/sensitive data. 

Worked example (book VIP tickets for event 123, with optional notify):
```
POST /events/123/bookings?notify=true
{
  "tickets": [
    {"section": "VIP", "quantity": 2},
    {"section": "General", "quantity": 1}
  ],
  "payment_method": "credit_card"
}
```
- Event `123` → **path** (required to identify the resource)
- `notify=true` → **query** (optional behavior)
- tickets + payment → **body** (the core data)

Mnemonic: **path = which endpoint · query = how it behaves · body = the data itself.**

### 2.4 Returning Data
A response = **status code** + **body** (usually JSON).
Common status codes: **200** success · **201** created · **400** bad request · **401** auth required · **404** not found · **500** server error.
> Don't overthink codes in interviews — they mainly want you to know **4xx = client error** vs **5xx = server error**. Even writing `2XX`/`4XX` is usually fine.

---

## 3. GraphQL

### The problem it solves
Facebook, 2012: mobile app needed different data than web; REST forced either **endpoint proliferation** or **over-fetching**. With REST + diverse clients you get two bad options:
- **Many endpoints** per use case → maintenance headache.
- **Fat endpoints** returning everything → **over-fetching** (mobile downloads MBs it doesn't use).
- (And **under-fetching** → many calls + round trips to assemble a page.)

### How it works
**Single endpoint** that accepts a **query describing exactly the data you want**; server returns that exact shape.
```graphql
query {
  event(id: "123") {
    name
    date
    venue { name address }
    tickets { section price available }
  }
}
```
Mobile requests just `name`/`date`; web requests the full tree — same endpoint, no new backend endpoints.

### When to use it (interview)
- Diverse clients with different data needs ("mobile needs different data than web").
- "Avoid over-fetching/under-fetching" is the explicit signal.
- Frontend teams need to iterate **without backend changes** (request any field already in the schema).
> But it **adds complexity** (query parsing, schema validation, caching). Default to REST unless the problem calls for GraphQL's flexibility.

### Schema design
Model core entities as **types**, with relationships defined **in the schema**:
```graphql
type Event  { id: ID!  name: String!  date: DateTime!  venue: Venue!  tickets: [Ticket!]! }
type Venue  { id: ID!  name: String!  address: String! }
type Query  { event(id: ID!): Event   events(limit: Int, after: String): [Event!]! }
```

### ⚠️ The N+1 problem (GraphQL's signature gotcha)
Querying `events { venue }` can run **1 query for events + N queries (one per event's venue)** = **N+1**. 100 events → **101 queries** instead of 2.
- **Why:** resolvers run **per field, per object** — each `venue` is resolved independently.
- **Fix:** **batching / DataLoader** — defer + collect the venue IDs for one tick, issue a single `SELECT ... WHERE id IN (...)`, and hand each resolver its result (also caches per request). Collapses N+1 → 2.
- **Tradeoff:** real complexity you don't have with REST (one join). A reason not to default to GraphQL.

### Authorization differs
REST secures **whole endpoints**; GraphQL secures **individual fields** (in resolvers) — e.g. a user sees an event's `name`/`date` but not `venue`. **Harder** because one query touches many fields/types, each needing its own rule → more surface area, easier to miss.

> In interviews: mention GraphQL when you see clear over/under-fetching, but don't default to it. Solve the core architecture with simpler tools first.

---

## 4. RPC (Remote Procedure Call)

**Intuition:** call a procedure on a remote server as if it were a local function, without dealing with network details. gRPC uses **Protocol Buffers** + **HTTP/2** → faster than JSON-over-HTTP for service-to-service.

### Action-oriented (vs REST's resource-oriented)
```
// Instead of GET /events/123
getEvent(eventId: "123")
// Instead of POST /events/123/bookings
createBooking(eventId: "123", userId: "456", tickets: [...])
// Instead of GET /events/123/tickets
getAvailableTickets(eventId: "123", section: "VIP")
```
RPC "feels right" when the operation is fundamentally **a behavior** that doesn't map cleanly to CRUD on a noun — e.g. `checkPermission(userId, resource)`, `validateToken(...)`, `calculateShippingCost(...)`. Forcing those into a REST resource is awkward (what noun is a permission check?).

Most popular: **gRPC** (Protocol Buffers + HTTP/2). Also **Apache Thrift** (Facebook, multi-language).

### Protocol Buffers & type safety
```protobuf
service TicketService {
  rpc GetEvent(GetEventRequest) returns (Event);
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
  rpc GetAvailableTickets(GetTicketsRequest) returns (TicketList);
}
message GetEventRequest { string event_id = 1; }
message Event { string id = 1; string name = 2; int64 date = 3; Venue venue = 4; }
```
From one `.proto`, gRPC **generates client + server code in many languages** → e.g. a Go service and a Java service communicate with **compile-time type safety**, catching mismatches before deploy.

### When to use (interview)
Consider RPC when:
- **Performance is critical** (binary + HTTP/2 ≫ JSON REST)
- **Type safety matters** (generated stubs prevent runtime errors)
- **Service-to-service** internal APIs (no need for REST resource semantics)
- **Streaming** needed (gRPC supports bidirectional streaming)

> Pattern: **REST for public endpoints, gRPC for internal** service-to-service (booking ↔ payment ↔ inventory). And you usually **won't outline internal APIs** in the API step — just note "internal services talk over RPC" in the high-level design; focus the API step on **user-facing** endpoints.

---

## 5. The 9 API Design Principles

These are the standard your design is judged against. You won't recite them — you **apply them out loud** as justifications ("I'll use an idempotency key here so a retry can't double-book").

1. **Design around resources, not actions.** Expose things; let HTTP methods express what clients do. `/getUserBookings` → `GET /users/{id}/bookings`; `/processPayment` → `POST /payments`.
2. **Be consistent everywhere.** Same naming, params, response shapes across endpoints. If one filters `?status=paid` and another `?state=paid`, consumers can't predict your API. Consistency is boring — that's the point.
3. **Principle of least surprise.** Use HTTP as everyone expects: GET never mutates, DELETE deletes, `200` means success (not an error wrapped in `"success": false`). Every deviation taxes every consumer forever.
4. **Keep the API stateless.** Each request carries everything the server needs (auth token, params, payload). Then **any server handles any request** → horizontal scaling is a load-balancer problem, not an app problem. (Why **JWTs** are popular.)
5. **Make retries safe.** Networks fail mid-request; the client can't tell if the work happened. GET/PUT/DELETE are idempotent by nature; for POST creating things users care about (bookings, payments), use **idempotency keys**.
6. **Paginate anything that can grow.** Every collection endpoint gets pagination from day one, **plus a cap on page size** so a client can't `?limit=100000` and take down your DB.
7. **Secure every endpoint by default.** Auth is **opt-out, not opt-in**: every endpoint needs a valid token unless deliberately public. Authorization confirms the caller can act on **this specific** resource. Rate limiting backs it up.
8. **Evolve without breaking clients.** Once someone depends on your API you don't control their upgrades (old mobile apps live for years). Adding a field is safe; renaming/removing/retyping is breaking → needs a versioning story.
9. **Make errors actionable.** Right status class (4xx = you, 5xx = us) + machine-readable code + human-readable message.

> You get points for **applying** them, not listing them.

---

## 6. Common API Patterns

### 6.1 Pagination
Can't return millions of rows at once. Two approaches:

**Offset-based** — `?offset=20&limit=10` (skip 20, take 10). Simple, supports "jump to page N."
- **Failure mode on live data:** if a record is inserted/deleted near the front while paging, all positions shift → you get **duplicates** (insert) or **skipped rows** (delete). Offsets drift under you.

**Cursor-based** — a **pointer to a specific record** (encoded id/timestamp of the last item):
```
GET /events?limit=10
{ "events": [...], "next_cursor": "cmd9atj3p000007ky19w1dpy2" }
GET /events?cursor=cmd9atj3p000007ky19w1dpy2&limit=10
```
- **Why it's stable:** anchored to a **record identity**, not a position → inserts/deletes elsewhere don't cause dupes/skips.
- **Gives up:** random access — **no "jump to page 5"**; only sequential next/previous. Also harder to implement.

> Interview: **offset is usually fine** unless real-time/high-volume or the interviewer probes. They care most that you **remembered to paginate**.

### 6.2 Filtering and Sorting
Convention = **query parameters**: `/events?city=NYC&status=upcoming` (filter), `/events?sort=date` (sort), leading minus for descending `?sort=-date`.
The real work is **consistency** — if events filter by `status`, bookings should too (not `state`). Mismatched filter names across endpoints is a top consistency violation. In an interview, show one filtered endpoint and say the same conventions apply everywhere.

### 6.3 Idempotency Keys
Scenario: user books, request times out, client retries — did the first go through? If yes, the retry **double-books / double-charges**. POST isn't idempotent and "don't retry" isn't an option (networks fail).
```
POST /events/123/bookings
Idempotency-Key: 8e03978e-740c-4c2e-abd1-52f427d5c177
{ "tickets": [{"section": "VIP", "quantity": 2}] }
```
**Server behavior:** store the key alongside the first request's result. On retry with the same key:
- **completed** → return the **stored response** (no reprocessing),
- **in-progress** → wait and return the same result (or reject "already exists") — never double-process,
- **new** → process once, store key + result.
Same key → same result → no double charge. (Exactly how **Stripe** handles payment retries.) Raising this on payments/bookings/orders is a strong interview signal — shows you think about failure, not just the happy path.

### 6.4 Consistent Error Responses
Status code = category; body = what actually happened. Use a **consistent envelope** on every endpoint:
```json
{ "error": { "code": "SEAT_UNAVAILABLE", "message": "Section VIP has only 1 seat remaining" } }
```
- `code` = **stable string** client code branches on (e.g. show a seat picker on `SEAT_UNAVAILABLE`).
- `message` = **for humans**, can change freely without breaking anyone.
Keep the shape identical everywhere so consumers write error handling once.

### 6.5 Versioning Strategies
- **URL versioning** — `/v1/events`, `/v2/events`. Explicit, obvious, easy to route/test. **Preferred for interviews.**
- **Header versioning** — `Accept-Version: v2`. Cleaner URLs, follows HTTP better, but less obvious and harder to test in a browser.
- **Breaking vs non-breaking:** adding a field = **non-breaking** (clients ignore unknown fields); renaming/removing/retyping = **breaking** → needs a new version.
- **Why it matters for mobile:** you don't control when users update; **old app versions linger for years** and can't be forced to upgrade — so breaking changes must be versioned while the old version keeps serving legacy clients.
> Breakdowns often omit versioning — not because it's unimportant in practice (it is), but because most interviewers don't care.

---

## 7. Security Considerations

Showing security awareness sets you apart. You don't need a bulletproof design — signal you think about production.

### 7.1 Authentication vs Authorization
- **Authentication (authN)** = verify **identity** (are you who you claim?).
- **Authorization (authZ)** = verify **permissions** (are you allowed to do *this specific* action?).
- Ticketmaster example: authN verifies the request is from `john@example.com`; **authZ** checks John is cancelling **his own** booking, not someone else's. (authN can pass while authZ must still fail.)
> Interview default: call out **which endpoints require auth**, and say you'd use a **JWT** or a DB-backed session.

### 7.2 API Keys vs JWT
**API keys** — long random strings acting as passwords **for applications**, sent in the `Authorization` header; server looks up the key for identity/permissions/rate limits.
```
GET /events
Authorization: Bearer sk_live_abc123...
```
- Best for **server-to-server** (you control both sides) and **3rd-party developer** access.
- **Bad for user-facing apps** — users shouldn't manage crypto strings; keys don't expire or carry user context like sessions.

**JWT (JSON Web Token)** — encodes user info **in the token**, signed with a secret. On each request you verify the signature and read user context **without a DB lookup**.
```
// JWT payload
{ "user_id": "123", "email": "john@example.com", "role": "customer", "exp": 1640995200 }
```
- Great for **distributed systems**: any service with the verification key validates independently (API gateway verifies, forwards to booking service with confidence).
- **Stateless** (no session lookup) + carries user context → ideal for **user-facing** web/mobile.

> **Rule:** API keys for internal service + external developer access; **JWT for user sessions** in web/mobile.

### 7.3 Role-Based Access Control (RBAC)
Assign **roles** to users, **permissions** to roles:
```
customer      → book tickets, view own bookings
venue_manager → create events, view sales for their venues
admin         → everything
```
In API design, check **both**:
```
GET /bookings/{id}
1. authenticated?  (valid JWT)
2. authorized?     (owns this booking OR is admin)
```
> Interview: at most mention which endpoints which roles can hit; often not relevant.

### 7.4 Rate Limiting & Throttling
Restricts requests per client per period → protects from abuse & accidental overload.
- **Per-user**: 1000/hour per authenticated user
- **Per-IP**: 100/hour for unauthenticated
- **Endpoint-specific**: 10 booking attempts/min (anti-scalping)
Implemented at the **API gateway** or middleware; exceed → **429 Too Many Requests**.
> Interview: "we'll rate limit to prevent abuse" is usually enough; don't design the algorithm unless asked.

---

## 8. Conclusion — interview judgment

API design in interviews = **engineering judgment, not perfect specs.** Pick the right protocol (usually **REST**), model resources clearly, show basic authN/security awareness. When unsure, fall back on the principles: **consistent, predictable, stateless, safe to retry.**
> **Time budget:** candidates mess up more by **over-investing** in APIs than under-investing. **Spend ≤5 minutes** on the API step, then move to the bigger architectural challenges.

---

## 9. Quick Reference

| Topic | Default / Key point |
|---|---|
| Protocol default | **REST** ("I'll use REST APIs" and move on) |
| GraphQL when | diverse clients, over/under-fetching, flexible/fast-iterating frontend |
| RPC (gRPC) when | internal service-to-service, perf-critical, polyglot, streaming |
| Real-time | WebSockets / SSE (not traditional APIs) |
| Idempotent methods | GET, PUT, DELETE (POST & accumulating PATCH are not) |
| Input placement | path = which · query = how · body = payload |
| Pagination | offset (simple, drifts on live data) vs cursor (stable, no jump-to-page); **always cap page size** |
| Retries | idempotency key for POST bookings/payments |
| Errors | envelope: machine `code` + human `message`; 4xx you / 5xx us |
| Versioning | URL (`/v1`) preferred; adding field = safe, renaming = breaking |
| Auth | JWT for user sessions (stateless); API keys for internal/3rd-party |
| Time budget | ≤ 5 minutes on API design |

---

## 10. Self-Test Q&A (tricky, understanding-level)

**Q1. Interviewer: "mobile needs a few fields, web needs everything, and the frontend keeps asking for endpoint variations." Which protocol + which REST pains?**
**GraphQL.** REST pains: **endpoint proliferation** (maintenance cost) and **over-fetching** (fat endpoint returns unused data) — plus **under-fetching** (many calls/round trips). GraphQL's single endpoint lets each client request exactly the fields it needs.

**Q2. You want gRPC for your public mobile API for speed — why wrong, and the split?**
Browsers can't natively speak gRPC (HTTP/2 + binary protobuf; needs gRPC-Web proxy) and tooling is immature for clients you don't control. Split = **"REST out, gRPC in"**: REST/JSON public, gRPC internal service-to-service.

**Q3. What's wrong with `POST /events/{id}/book`, `POST /bookings/purchase`, `GET /getUserBookings`?**
All embed **action verbs in the path**. REST = resource nouns + HTTP method as the verb. Fix: `POST /events/{id}/bookings`, `POST /payments` (or `PATCH /bookings/{id}`), `GET /users/{id}/bookings`. (Plural nouns is a separate minor rule.)

**Q4. Nested path vs query param — the rule?**
**Required to identify the resource → path/nested** (`/events/{id}/tickets`). **Optional filter among many → query** (`/tickets?event_id=123&section=VIP`). If you might want all tickets regardless of event, event_id is optional → query param.

**Q5. Pass: which event, optional notify flag, ticket list. Where does each go?**
Event → **path** (structural, required). notify → **query** (optional modifier). tickets+payment → **body** (payload). `POST /events/123/bookings?notify=true { tickets, payment_method }`.

**Q6. Idempotency of GET/POST/PUT/PATCH/DELETE? Is PATCH idempotent?**
Idempotent: GET, PUT, DELETE. Not: POST, PATCH. PATCH depends on semantics — "set field = X" is idempotent; "append/increment" is not (delta keeps changing state).

**Q7. DELETE returns 204 then 404 — violates idempotency?**
No. Idempotency is about the final **server state** (resource stays deleted), not identical status codes. The 204→404 just describes state; it doesn't change it.

**Q8. GraphQL `events { venue }` performance problem?**
**N+1**: 1 query for events + N for venues (resolvers run per field/object). 100 events → 101 queries. Fix: **DataLoader batching** (collect ids → single `IN` query, + caching) → 2 queries. Tradeoff: complexity REST doesn't have.

**Q9. Authorization: REST vs GraphQL?**
REST secures **endpoints**; GraphQL secures **fields** (resolvers). Harder in GraphQL because one query touches many fields/types, each needing its own rule → more surface area, easier to miss.

**Q10. RPC action example + action-vs-resource?**
`checkPermission(userId, resource)` — a behavior, not a noun; awkward as a REST resource. REST = nouns + HTTP verbs (resource-oriented); RPC = call procedures (action-oriented). RPC fits internal behaviors/computation.

**Q11. Statelessness → what scaling benefit + which auth mechanism?**
**Horizontal scaling**: any server handles any request → load-balancer problem, not app problem. Embodied by **JWT** — signed token carries user context, so any server verifies + authorizes with no session lookup.

**Q12. Idempotency-key retry — what does the server do?**
Store key→result on first call. Retry with same key: completed → return stored response; in-progress → wait/return same (or reject); new → process once. No double charge (Stripe-style).

**Q13. Offset vs cursor pagination — offset's failure, cursor's fix + cost?**
Offset drifts on live data: insert/delete near front → duplicates/skips. Cursor anchors to a stable record id → insertion-safe. Cost: no "jump to page N," sequential only; harder to implement.

**Q14. `GET /events?limit=100000` works — which principle violated + fix?**
"Paginate anything that can grow" — specifically the missing **page-size cap**. Fix: clamp `limit` to a max (e.g. `min(requested, 100)`) so a client can't overwhelm the DB.

**Q15. Adding a field vs renaming a field — breaking? Why mobile matters?**
Adding = non-breaking (clients ignore unknown fields); renaming/removing/retyping = breaking. Mobile: old app versions linger for years and can't be force-updated → breaking changes need a new API version while the old one keeps serving legacy clients.
