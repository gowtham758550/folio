---
title: "LinkForge — Building a URL Shortener with .NET 10, Angular, and Redis"
description: "How I built a full-stack URL shortener covering short code generation, cache-aside redirects, Google OAuth, click analytics, and Docker Compose."
date: 2026-04-26
tags: post
layout: post.njk
---

URLs are ugly. Long redirect chains full of tracking parameters are worse. LinkForge turns any URL into a clean 7-character short link — with click analytics, custom aliases, and expiration built in.

This post walks through how each layer of the system works and the reasoning behind the key design choices.

---

## Stack

| Layer | Technology |
|---|---|
| API | ASP.NET Core (.NET 10) |
| Frontend | Angular 20 |
| Database | MongoDB |
| Cache | Redis |
| Auth | Google OAuth 2.0 → JWT |
| Containers | Docker + Compose |

---

## Architecture

The system splits into three clear layers. The API owns business logic. Infrastructure owns persistence. The client owns presentation. No logic leaks across boundaries.

<pre class="mermaid">
graph TD
    A[Angular SPA] -->|Google OAuth → JWT| B[ASP.NET Core API]
    B --> E[Infrastructure Layer]
    E -->|Read / Write| C[(MongoDB)]
    E -->|Cache| D[(Redis)]
    D -->|Miss| C
    C -->|Populate| D
</pre>

Redis sits in front of MongoDB on the hot path. For a redirect-heavy system, this is the difference between microseconds and milliseconds. MongoDB is the source of truth — Redis is a fast read layer that can be dropped and rebuilt at any time.

---

## How Short Codes Are Generated

Short codes are hash-derived — the same URL always produces the same code. No random IDs, no auto-increment sequences. Collisions are resolved by slightly modifying the input and rehashing, rather than picking a random fallback.

<pre class="mermaid">
flowchart TD
    A[Long URL] --> B[MD5 Hash → 16 bytes]
    B --> C[BigInteger → Base62]
    C --> D[Take first 7 chars]
    D --> E{Collision in DB?}
    E -->|No| F[Save & Return short code]
    E -->|Yes, attempt < 5| G[Append counter to input, retry]
    G --> B
    E -->|5 failures| H[Error]
</pre>

Base62 uses the alphabet `a-zA-Z0-9`, giving 62⁷ ≈ 3.5 trillion combinations from just 7 characters. MD5 isn't used here for cryptographic strength — it's used because it's fast and produces uniform distribution across the output space, which is all that matters for hashing into a key space.

Collisions are rare but handled: if a short code already exists in the database, the input is modified by appending a counter and rehashed. Up to 5 retries before giving up. In practice, this never triggers.

Custom aliases skip the hashing step entirely — the user's chosen string is validated and stored directly.

---

## Redirect Flow: Redis Cache-Aside

Redirects are the highest-traffic operation in the system. Every request checks Redis before touching MongoDB.

<pre class="mermaid">
sequenceDiagram
    participant User
    participant API
    participant Redis
    participant MongoDB

    User->>API: GET /{shortCode}
    API->>Redis: Lookup url:{shortCode}
    alt Cache Hit
        Redis-->>API: {trackFlag}|{longUrl}
        API->>API: Fire-and-forget increment clickCount + write ClickEvent
        API-->>User: 301 or 302 based on trackFlag
    else Cache Miss
        Redis-->>API: null
        API->>MongoDB: Find by shortCode
        alt Found and valid
            MongoDB-->>API: URL document
            API->>Redis: Cache with TTL
            API-->>User: 301 or 302 based on trackFlag
        else Expired or not found
            API-->>User: Redirect → /not-found (Angular page)
        end
    end
</pre>

The cache value is a small encoded string: `{trackFlag}|{longUrl}`. A single Redis read gives the API everything it needs — the destination URL and whether tracking is on.

Every API hit — cache or DB — fires a background task that always increments `clickCount` and writes a `ClickEvent` document, unconditionally.

The flag controls only the redirect type:

- **Fast mode** — **301 permanent redirect**. Browsers cache it forever. Subsequent clicks go straight to the destination — the API is never hit again. Zero server load after the first visit, but repeat clicks from the same browser won't be counted.
- **Accurate mode** — **302 with `no-cache` headers**. Browsers never cache it. Every click reaches the API, so every click is counted and recorded. Adds a small server round trip on each visit.

No second DB round trip needed — the flag is encoded in the cache value so the redirect type can be decided without an extra lookup.

For URLs with an expiry date, the cache TTL is set to `expiresAt - now`, so the cache entry and the URL expire together. Permanent URLs have no TTL set — the entry persists in Redis until the cache is cleared or the entry is manually invalidated.

When a short code is expired or doesn't exist, the API redirects the browser to the Angular `/not-found` page rather than returning a raw 404. The page explains the link is expired or removed and offers a path back to the dashboard.

Analytics recording is fire-and-forget — the redirect is never blocked waiting for a database write. A slow disk flush on the analytics side doesn't add latency to every user's click.

---

## Authentication: Google OAuth → JWT

The API is stateless — no server-side sessions, no cookies in production. Authentication goes through Google and immediately hands off to a signed JWT.

<pre class="mermaid">
sequenceDiagram
    participant Browser
    participant API
    participant Google
    participant MongoDB

    Browser->>API: GET /auth/google
    API->>Google: Redirect (OAuth challenge)
    Google->>Browser: Consent screen
    Browser->>Google: User approves
    Google->>API: Callback with auth code
    API->>Google: Exchange for user info
    API->>MongoDB: Upsert user (by googleId)
    API->>API: Sign JWT (HMAC-SHA256, 24h)
    API->>Browser: Redirect → ?token={jwt}
    Browser->>Browser: Store in localStorage
    Browser->>API: All future requests: Authorization: Bearer {token}
</pre>

ASP.NET Core's Google middleware handles the OAuth handshake and stores the result in a transient cookie. The callback handler reads the user claims from that cookie, upserts the user into MongoDB, and generates a JWT. The Google cookie is then immediately cleared — it exists only for the duration of the handshake. From that point on, the API-issued JWT is the only credential, valid until it expires.

The upsert is idempotent by `googleId` — returning users update their profile picture if it changed, new users are inserted. The same endpoint handles both cases.

---

## MongoDB Collections

Three collections, each shaped around its query pattern rather than just normalized storage.

<pre class="mermaid">
erDiagram
    URLS {
        ObjectId _id
        string shortCode
        string longUrl
        ObjectId userId
        datetime createdAt
        datetime expiresAt
        int clickCount
        bool isCustomAlias
        bool trackEveryClick
    }

    USERS {
        ObjectId _id
        string googleId
        string email
        string name
        string picture
        datetime createdAt
    }

    CLICK_EVENTS {
        ObjectId _id
        string shortCode
        ObjectId userId
        datetime timestamp
    }

    USERS ||--o{ URLS : creates
    URLS ||--o{ CLICK_EVENTS : generates
</pre>

`expiresAt` carries a sparse index for query performance. Expired URLs are intentionally retained in the database — they are never automatically deleted. This allows users to see their full link history in the dashboard, filtered by Active, All, or Expired tabs. The redirect layer enforces expiry at request time by checking `expiresAt` against the current timestamp and sending the browser to the custom not-found page if the link has lapsed.

`clickCount` is incremented with MongoDB's `$inc` operator, which is atomic at the document level. Multiple API instances can serve redirects simultaneously without racing on the same counter.

`googleId` and `email` on the users collection both carry unique indexes, making the upsert safe under concurrent logins.

---

## Click Analytics Pipeline

Every redirect records a `ClickEvent` document and increments `clickCount` — unconditionally, regardless of whether Fast or Accurate mode is set. The mode only controls whether the browser caches the redirect. The dashboard shows both the running total and a 30-day daily breakdown, filterable by All, Active, or Expired links.

<pre class="mermaid">
flowchart LR
    A[Redirect hits API] --> B[Increment clickCount atomically]
    A --> C[Fire-and-forget: write ClickEvent]

    D[User opens Analytics] --> E[MongoDB aggregation pipeline]
    E --> F["$match: shortCode + last 30 days"]
    F --> G["$group: by date, count per day"]
    G --> H[Fill zeros for empty days]
    H --> I[30-day chart in Angular]
</pre>

The running total comes from `clickCount` on the URL document — one field, one read, always fast. The daily breakdown comes from aggregating `click_events` — grouped by date string, sorted ascending, with the frontend filling in zeros for days with no activity so the chart always covers a full 30-day window.

The two sources serve different purposes. `clickCount` is optimized for display at a glance. `click_events` is optimized for time-series analysis. Both exist because neither alone is sufficient.

---

## Docker Compose

One command starts all five services. No manual setup, no local dependencies.

<pre class="mermaid">
graph TD
    Client["client<br/>(Angular → nginx)<br/>:4200"] --> API
    API["api<br/>(ASP.NET Core)<br/>:5001"] --> MongoDB
    API --> Redis
    MongoDB["mongodb:8<br/>:27017<br/>persistent volume"]
    Redis["redis:7-alpine<br/>:6379<br/>ephemeral"]
</pre>

MongoDB data persists via a named Docker volume. Redis is intentionally ephemeral — if the cache container is lost, the next request to each short code repopulates it from MongoDB. No data is at risk.

All secrets — JWT signing key, Google OAuth credentials, connection strings — live in a `.env` file and are passed as environment variables. Nothing sensitive is baked into the image.

---

## Design Decisions Worth Noting

**MongoDB over PostgreSQL** — URL metadata is naturally schema-flexible. Some URLs have an expiry date, some don't. Some have custom aliases, some don't. MongoDB's document model handles optional fields cleanly without nullable columns and partial indexes on a relational schema.

**Cache-aside over write-through** — Write-through pre-populates the cache on every write, regardless of whether the link ever gets clicked. Cache-aside only caches URLs that are actually accessed, keeping memory usage proportional to real traffic. If the cache is lost, it self-heals on the next request without any rebuild step. For a system where a URL is created once but potentially redirected thousands of times, lazy population is the more efficient fit.

**MD5 for hashing** — Cryptographic strength isn't needed here. Uniform distribution and speed are. MD5 is fast, deterministic, and produces outputs that spread evenly across the Base62 space. The retry loop handles the rare collision without adding complexity to the primary path.

**Fire-and-forget analytics** — Redirect latency is user-visible. Analytics write latency is not. Decoupling them means a slow MongoDB write never shows up as a slow redirect.

**Server-side filtering over client-side** — The dashboard filter tabs (All / Active / Expired) push the filtering into a MongoDB query rather than fetching all records and slicing in the browser. As link counts grow, the payload stays small regardless of how many expired links accumulate.

**Expiry validation at creation** — The API rejects any expiry date less than 1 hour from now. This prevents negative Redis TTLs and ensures the link is actually usable before it expires.

---

## Source

Full source code: [github.com/gowtham/linkforge](#)
