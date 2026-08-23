# Scaling Benchmarks

Benchmarking Node.js (Express, Fastify, cpeak) and C++ HTTP servers under realistic load — Postgres, Redis, PM2 cluster mode, and raw throughput comparisons.

**Test config:** `autocannon -c 20 -d 20 -w 6`
**Mode:** Single process (1 core) unless stated otherwise
**Machine:** Intel i7-1255U, 10 cores / 12 logical processors, 15.7GB RAM

---

### Autocannon Parameters

[Autocannon](https://github.com/mcollina/autocannon) is a Node.js HTTP load testing tool. Here's what each flag means:

| Flag | Value | Meaning |
|------|-------|---------|
| `-c` | `20` | **Connections** — 20 concurrent HTTP connections kept open at all times. Each connection is like a user continuously sending requests. |
| `-d` | `20` | **Duration** — run the test for 20 seconds. |
| `-w` | `6` | **Workers** — spawn 6 worker threads to generate load. More workers = more CPU available to generate requests, reducing the chance that autocannon itself becomes the bottleneck. |
| `-p` | `2` | **Pipelining** — send 2 requests back-to-back on the same connection without waiting for the first to respond (HTTP/1.1 pipelining). Multiplies effective concurrency. Only used on endpoints that support it. |
| `-m` | `PATCH` | **Method** — HTTP method to use (GET by default). |
| `-H` | `Content-Type: ...` | **Header** — custom HTTP header to attach to every request. |
| `-i` | `body.json` | **Input file** — read the request body from a file (used for POST/PATCH with a JSON body). |

---

### What Actually Limits a Server?

When benchmarking a server there are three resources that can become the bottleneck — and only one of them is in play at a time:

**CPU** is the bottleneck when the server has to do real computation on every request — parsing JSON, building large response objects, running validation logic, doing math. The `/update-something` endpoint is a good example: it builds a 100-item history array and formats strings on every request. More cores (via PM2) directly helps here because each core can handle one request independently.

**Database / external I/O** is the bottleneck when the server spends most of its time waiting for a response from Postgres or Redis. The CPU is mostly idle — it sends a query, then just sits there waiting. This is why Node.js's async event loop is so powerful for these workloads: instead of blocking while waiting, it moves on and handles other requests. `/code-v1`, `/code-v2`, `/code-v3` are all database-bound. Adding more CPU cores does not help here — the bottleneck is how fast Postgres can answer.

**Network (throughput)** is the bottleneck when the response payload is large and the server is limited by how much data it can push through the network per second. The `Bytes/Sec` column in autocannon measures this. For example `/update-something` returns ~32KB per response — at 3,000 Req/Sec that's ~96 MB/s of outgoing data. On a real production server with multiple clients across the internet, network bandwidth becomes a hard ceiling regardless of how fast the CPU or database is. In these local benchmarks the client and server are on the same machine (loopback), so network is not a real constraint — but in production it absolutely is. A server doing 100k Req/Sec returning even 1KB each is pushing 100 MB/s — that's a dedicated gigabit link saturated.

---

## /simple

**What it does:** Returns a static `{"message":"hi"}` — no database, no Redis, no computation. Pure framework overhead. This measures the raw request/response throughput of each framework.

| Framework | Req/Sec (avg) | Req/Sec (50%) | Total Requests | Latency (avg) |
|-----------|--------------|--------------|----------------|---------------|
| Express   | 7,578        | 7,951        | 152k / 20s     | 4.25 ms       |
| Fastify   | 19,160       | 19,023       | 383k / 20s     | 1.39 ms       |
| cpeak     | 18,862       | 19,663       | 377k / 20s     | 1.41 ms       |

---

## /update-something (PATCH)

**What it does:** Accepts a JSON body with 10 fields, validates URL params, builds a 100-item history array with large metadata strings, and returns ~32KB of JSON per response. No database — pure CPU and memory work. Tests how the framework handles real computation per request.

| Framework | Req/Sec (avg) | Req/Sec (50%) | Total Requests | Latency (avg) |
|-----------|--------------|--------------|----------------|---------------|
| Express   | 2,020        | 2,167        | 40k / 20s      | 17.34 ms      |
| Fastify   | 3,284        | 3,083        | 66k / 20s      | 10.48 ms      |
| cpeak     | 3,261        | 3,337        | 65k / 20s      | 10.55 ms      |

---

## Database Setup

**Table:** `codes`

| Column | Type | Details |
|--------|------|---------|
| `id` | `SERIAL PRIMARY KEY` | Auto-incrementing integer, indexed |
| `code` | `CHAR(500) UNIQUE NOT NULL` | A randomly generated 500-character alphanumeric string |
| `created_at` | `TIMESTAMP` | Defaults to `now()` |

**Seeding:** The table was seeded with **3 million records** using the command:
```
node database/seed.js -r 3000000
```
Each row's `code` field is a unique random 500-character string generated in Node.js and batch-inserted into Postgres. This simulates a real-world table with a large dataset, which is exactly what makes the query strategy in the following benchmarks matter so much.

---

## /code-v1 (Postgres - full table scan)

**What it does:** `SELECT * FROM codes ORDER BY RANDOM() LIMIT 1` — scans all 3 million rows, assigns a random number to each, sorts them, returns one. O(n) — gets worse as the table grows.

| Framework | Req/Sec (avg) | Total Requests | Latency (avg) |
|-----------|--------------|----------------|---------------|
| Express   | ~0 (timeouts) | 54 / 20s (36 timeouts) | >20s (all timed out) |
| Fastify   | ~0 (timeouts) | 54 / 20s (36 timeouts) | >20s (all timed out) |

---

## /code-v2 (Postgres - index lookup)

**What it does:** Runs `SELECT COUNT(*)` to get total rows, picks a random ID, then fetches by primary key index. Two DB queries per request but no full table scan.

| Framework | Req/Sec (avg) | Total Requests | Latency (avg) |
|-----------|--------------|----------------|---------------|
| Express   | 1.95         | 75 / 20s (18 timeouts) | 4273.42 ms    |
| Fastify   | 11.55        | 249 / 20s      | 1498.1 ms     |

---

## /code-v3 (Postgres - optimised index lookup)

**What it does:** Gets max ID via `ORDER BY id DESC LIMIT 1`, picks a random ID up to that, fetches by index. Slightly faster than v2 — still 2 DB queries but avoids the expensive COUNT.

| Framework | Req/Sec (avg) | Total Requests | Latency (avg) |
|-----------|--------------|----------------|---------------|
| Express   | 2,223.7      | 44k / 20s      | 7.59 ms       |
| Fastify   | 3,440        | 69k / 20s      | 4.73 ms       |

---

## /code-fast (Redis only)

**What it does:** Gets max ID from Redis (`codes:seq`), picks a random ID, fetches the record from Redis hash (`code:{id}`). No Postgres — everything served from memory. O(1) lookup.

| Framework | Req/Sec (avg) | Total Requests | Latency (avg) |
|-----------|--------------|----------------|---------------|
| Express   | 2,717.6      | 54k / 20s      | 6.12 ms       |
| Fastify   | 4,718.11     | 94k / 20s      | 3.31 ms       |

---

## PM2 Cluster Mode (12 instances)

**What it does:** Node.js is single-threaded by default — one process, one core. PM2 cluster mode spawns one Node.js process per logical CPU core (12 on this machine) and puts a load balancer in front of them. Incoming requests are distributed round-robin across all 12 processes, so all cores are working in parallel instead of one. This is how you scale a Node.js app to use your full machine.

**Note on Redis + PM2:** PM2 helped `/simple` significantly (Fastify jumped to 21k Req/Sec) because that endpoint is pure CPU — more cores = more parallel work. But `/code-fast` barely improved because Redis is single-threaded. All 12 Node.js processes funnel into the same Redis instance, so Redis becomes the bottleneck, not Node.js. To scale Redis itself you'd need Redis Cluster.

| Endpoint   | Framework | Req/Sec (avg) | Total Requests | Latency (avg) |
|------------|-----------|--------------|----------------|---------------|
| /simple    | Express   | 7,173.35     | 143k / 20s     | 1.99 ms       |
| /simple    | Fastify   | 21,239       | 425k / 20s     | 0.33 ms       |
| /code-fast | Express   | 2,972.5      | 59k / 20s      | 5.54 ms       |
| /code-fast | Fastify   | 4,735.86     | 95k / 20s      | 3.29 ms       |
