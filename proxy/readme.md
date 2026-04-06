# OrchProxy — Next-Gen Reverse Proxy & API Gateway
## ORCHATHON Hackathon — Full Production Model

```
 Client → [IP Blacklist] → [WAF] → [Rate Limiter] → [Circuit Breaker] → Backend
                ↓              ↓           ↓
              403           400          429
```

---

## Quick Start

**Step 1 — Start the backend** (one terminal)
```bash
node backend.js
# Backend API on http://localhost:8080
```

**Step 2 — Start the proxy** (second terminal)
```bash
node proxy.js
# Proxy on http://localhost:9090
# Dashboard on http://localhost:9091
```

**Step 3 — Open the live dashboard**
```
http://localhost:9091
```

**Step 4 — Run attack demos** (third terminal)
```bash
node attack_demo.js all          # Run everything
node attack_demo.js legit        # Normal traffic
node attack_demo.js ddos         # DDoS flood
node attack_demo.js bruteforce   # Login brute force
node attack_demo.js sqlinject    # SQL injection
node attack_demo.js xss          # XSS payloads
node attack_demo.js cmdinject    # Command injection
node attack_demo.js pathtrav     # Path traversal
node attack_demo.js blacklist    # Blacklisted IP
```

---

## Architecture

### Request Pipeline
```
Incoming Request
      │
      ▼
 ┌──────────────────────────────────────────────────────────────┐
 │                      OrchProxy (port 9090)                   │
 │                                                              │
 │  ① IP Blacklist ──→ 403 Forbidden (instant drop)            │
 │         │                                                    │
 │  ② Circuit Breaker ──→ 503 (if backend is down)             │
 │         │                                                    │
 │  ③ Rate Limiter ──→ 429 Too Many Requests                   │
 │    (Sliding Window Log, per IP per endpoint)                 │
 │         │                                                    │
 │  ④ WAF — URL scan ──→ 400 Bad Request                       │
 │    SQL inject / XSS / path traversal / cmd inject           │
 │         │                                                    │
 │  ⑤ WAF — Body scan ──→ 400 Bad Request                      │
 │         │                                                    │
 │  ⑥ Forward to Backend (with request ID tracing)             │
 │         │                                                    │
 └─────────┼────────────────────────────────────────────────────┘
           │
           ▼
    Backend (port 8080)
           │
           ▼
    Response → Client
```

### Dashboard (port 9091)
- Server-Sent Events (SSE) for real-time updates
- No page refresh needed
- Shows: RPS, blocked count, avg latency, top endpoints, top attackers
- Live security event log with attack type tags
- Circuit breaker status indicator
- Structured log stream

---

## Features

| Feature | Implementation |
|---|---|
| **Dynamic Routing** | config.json → backend_url, hot-reloaded via fs.watch() |
| **Rate Limiting** | Sliding Window Log algorithm, per IP per endpoint |
| **WAF: SQL Injection** | 7 patterns covering UNION, DROP, OR 1=1, xp_cmdshell, etc. |
| **WAF: XSS** | 5 patterns: script tags, javascript:, event handlers, iframe, eval |
| **WAF: Path Traversal** | Detects ../ and URL-encoded variants |
| **WAF: Cmd Injection** | Pipe, semicolon, subshell, backtick detection |
| **IP Blacklisting** | O(n) list lookup, instant 403 before any other processing |
| **Circuit Breaker** | CLOSED → OPEN → HALF_OPEN auto-recovery pattern |
| **Health Checks** | Periodic backend health polling, status in dashboard |
| **Request Tracing** | X-Request-Id header on every request for debugging |
| **Live Dashboard** | SSE-powered real-time stats, security feed, log stream |
| **Structured Logging** | JSON log lines with ts, level, component, message |
| **Body Size Limit** | Configurable max body size (default 1MB) |
| **Timeout Protection** | Configurable request timeout (default 10s) |
| **Dynamic Reload** | fs.watch() on config.json — zero downtime config updates |
| **Graceful Shutdown** | SIGINT handler closes servers cleanly |

---

## Config Reference (config.json)

```json
{
  "server": {
    "listen_port": 9090,          // proxy port
    "backend_url": "http://localhost:8080",
    "dashboard_port": 9091,       // dashboard port
    "request_timeout_ms": 10000,  // backend timeout
    "max_body_size_bytes": 1048576
  },
  "rate_limits": [
    {
      "path": "/login",           // prefix match
      "method": "POST",           // or "*" for any
      "limit": 5,                 // max requests
      "window_seconds": 60        // sliding window size
    }
  ],
  "security": {
    "block_sql_injection": true,
    "block_xss": true,
    "block_path_traversal": true,
    "block_command_injection": true,
    "blacklisted_ips": ["203.0.113.42"]
  },
  "circuit_breaker": {
    "enabled": true,
    "failure_threshold": 5,       // failures before OPEN
    "recovery_timeout_ms": 30000  // time before HALF_OPEN
  },
  "health_check": {
    "enabled": true,
    "path": "/health",
    "interval_ms": 5000
  }
}
```

**Dynamic Reload:** Edit config.json while running — changes apply within ~100ms via `fs.watch()`.

---

## Judging Criteria

### Performance
- Pure Node.js `http` module — no framework overhead
- Rate limiter uses in-memory Map (O(1) lookup)
- WAF rules compiled once at startup
- Can handle thousands of concurrent connections

### Correctness
- Sliding Window Log = accurate per-IP per-endpoint counting
- Each endpoint can have independent limits
- `*` method wildcard for catch-all rules

### Security
- 16 WAF rules covering 4 attack categories
- IP blacklist checked first (lowest cost, highest priority)
- Request ID tracing for forensic analysis
- Body size limits prevent OOM attacks

### Dynamic Reload
- `fs.watch()` triggers on file change — no polling overhead
- Zero downtime — existing connections unaffected
- New config applied to next incoming request

### Observability
- Real-time SSE dashboard on separate port (no impact on proxy perf)
- Structured JSON logs for parsing by log aggregators
- Circuit breaker state visible in real time
- Top attackers + top endpoints bar charts

---

## Backend API (port 8080 — protected by proxy)

```
GET  /health           → { status: "ok", uptime, ts }
GET  /getAllUsers       → { users: [...], total: 5 }
GET  /user/:id         → { user: {...} }
POST /login            → { token, user, expiresIn }  (admin / secret123)
POST /logout           → { message: "Logged out" }
GET  /products         → requires Bearer token
POST /products         → requires admin token
GET  /stats            → { users, products, activeSessions }
```

---

## Dependencies

**Zero external dependencies.** Uses only Node.js built-in modules:
- `http` — proxy and dashboard server
- `fs` — config file read + watch
- `url` — URL parsing
- `crypto` — request ID generation, token generation
- `path` — config file path resolution

No `npm install` needed.