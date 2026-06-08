# QKDN Codebase Walkthrough
**Software-Defined Quantum Key Distribution — Complete Technical Reference**

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Module Map](#2-module-map)
3. [BB84 Engine — `qkd/bb84.py`](#3-bb84-engine)
4. [Key Management System — `kms/kms.py`](#4-key-management-system)
5. [SDN Controller — `sdn/sdn.py`](#5-sdn-controller)
6. [Chat Application — `app/chat.py`](#6-chat-application)
7. [Relay Server — `app/relay.py`](#7-relay-server)
8. [Dashboard — `dashboard/server.py`](#8-dashboard)
9. [Deployment Pipeline](#9-deployment-pipeline)
10. [Integration Test — `test_local.py`](#10-integration-test)
11. [Key Data Flows](#11-key-data-flows)
12. [Security Model](#12-security-model)
13. [Bug History & Fixes](#13-bug-history--fixes)

---

## 1. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Mac (developer)                                                │
│  test_local.py  ·  deploy.sh  ·  wg_gen.py                     │
└────────────────────────────────┬────────────────────────────────┘
                                 │ SSH + rsync
          ┌──────────────────────┼─────────────────────┐
          │                      │                     │
   ┌──────▼──────┐        ┌──────▼──────┐    ┌────────▼────────┐
   │  Pi 4       │        │ Cloud VPS   │    │  Radxa Cubie    │
   │  Alice      │        │             │    │  Bob            │
   │  KMS :7001  │        │ SDN  :7000  │    │  KMS :7002      │
   │  WG 10.8.0.2│◄──WG──►│ Relay:9001  │◄──►│  WG 10.8.0.3   │
   └──────┬──────┘        │ Dash :8888  │    └────────┬────────┘
          │  192.168.100.x│             │             │
          └───────────────┤ WireGuard   ├─────────────┘
           direct ethernet│ 10.8.0.0/24 │
          (simulated fiber)└────────────┘
```

**Three physical tiers:**

| Tier | Nodes | Role |
|------|-------|------|
| Edge nodes | Pi 4, Radxa | QKD key generation + KMS |
| Cloud relay | VPS | SDN control, message relay, dashboard |
| Developer | Mac | Deployment, monitoring, chat client |

**Transport principle:** Key *bytes* never cross the network. Only the UUID (`key_ID`) and AES ciphertext travel over the wire. Each side independently holds the raw key bytes in its local SQLite DB.

---

## 2. Module Map

```
qkd_mvp/
│
├── qkd/
│   └── bb84.py          # BB84/BBM92 simulation engine
│
├── kms/
│   └── kms.py           # ETSI QKD 014 Key Management Entity (FastAPI)
│
├── sdn/
│   └── sdn.py           # SDN controller — BFD + Dijkstra + LFA + ECMP
│
├── app/
│   ├── chat.py          # Terminal QKD chat client
│   └── relay.py         # TCP store-and-forward relay
│
├── dashboard/
│   └── server.py        # HTML status dashboard (HTTP, no JS framework)
│
├── setup/
│   ├── wg_gen.py        # WireGuard keypair + config generator
│   ├── setup_vps.sh     # VPS systemd service installer
│   └── setup_node.sh    # Pi/Radxa systemd service installer
│
├── deploy.sh            # One-command Mac → all-nodes deploy
├── test_local.py        # Full local integration test (no hardware)
└── requirements.txt
```

---

## 3. BB84 Engine

**File:** `qkd/bb84.py`  
**Purpose:** Simulate BB84 quantum key distribution over a fiber link. No hardware required — produces statistically realistic QBER, photon loss, and eavesdropper detection.

### Physics constants

```python
QBER_THRESHOLD = 0.11    # 11% — ETSI GS QKD 011 security limit
FIBER_LOSS_DB  = 0.20    # dB/km — standard SMF-28 single-mode fiber
DETECTOR_EFF   = 0.20    # 20% — InGaAs SPAD efficiency
DARK_COUNT_RATE= 1e-5    # per gate — typical room-temp detector noise
```

### Core function: `run_bb84(link_id, dist_km, n_raw=4096, eavesdrop=False)`

**Step-by-step:**

1. **Fiber transmittance** — `η = 10^(-0.20 × d / 10) × 0.20`  
   At 5 km: η ≈ 0.126 × 0.20 = 0.0252 → roughly 2.5% of photons detected.

2. **Alice prepares** `n_raw=4096` qubits with random bits + random bases (Z/X).

3. **Eve intercept-resend** *(if `eavesdrop=True`)* — Eve measures in a random basis and re-sends. This introduces ~25% additional errors on basis-mismatched bits.

4. **Photon loss** — Each photon independently survives with probability `η + dark_count_rate`. Typically ~100 photons detected from 4096 sent.

5. **Basis sifting** — Keep only indices where Alice and Bob chose the same basis. Expected ~50% retention → ~50 sifted bits.

6. **QBER estimation** — Sample 25% of sifted bits as "check bits":
   - Normal: QBER = Gaussian noise centered at 0.02
   - Eve: QBER forced ≥ 0.18 (always above threshold)

7. **Privacy amplification** — Non-check sifted bits → `SHA3-256(raw_bits ‖ link_id)` → 256-bit key. This hashes away any partial information Eve may have gained.

8. **Return `SessionResult`** with `success=True` if QBER < 0.11.

### Pool generation: `generate_pool(link_id, dist_km, n=20)`

Runs `run_bb84` in a loop, filtering failed sessions (QBER ≥ threshold). Returns up to `n` successful `SessionResult` objects. Each gets a UUID assigned by KMS when inserted.

### `SessionResult` dataclass

```python
@dataclass
class SessionResult:
    link_id:   str      # e.g. "alice↔bob"
    dist_km:   float    # fiber length
    n_raw:     int      # photons sent
    n_sifted:  int      # after basis sifting
    n_key:     int      # always 256 (after PA)
    qber:      float    # measured error rate
    key_hex:   str      # 64-char hex = 32 bytes = 256-bit AES key
    ts:        float    # unix timestamp
    success:   bool
    eavesdrop: bool
```

---

## 4. Key Management System

**File:** `kms/kms.py`  
**Framework:** FastAPI + uvicorn  
**Standard:** ETSI GS QKD 014 (REST interface for quantum key exchange)  
**Storage:** SQLite with WAL mode + threading lock

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `NODE` | `alice` | Node identity |
| `PORT` | `7001` | HTTP listen port |
| `DIST` | `5.0` | Fiber distance (km) for BB84 |
| `POOL_MAX` | `40` | Target pool depth |
| `REFILL_AT` | `12` | Trigger refill when pool drops below this |
| `SYNC_INT` | `2` | Seconds between refill checks |
| `FRESH_DB` | `1` | Wipe DB on startup (clean runs) |

### Database schema

```sql
CREATE TABLE keys (
    key_id   TEXT PRIMARY KEY,  -- UUID v4
    peer     TEXT NOT NULL,     -- "bob" or "alice" (who this key is for)
    key_hex  TEXT NOT NULL,     -- 64-char hex (256-bit AES key)
    qber     REAL DEFAULT 0,    -- measured during BB84 session
    ts       REAL DEFAULT 0,    -- unix timestamp of generation
    used     INTEGER DEFAULT 0, -- 0=available, 1=consumed
    synced   INTEGER DEFAULT 0  -- 0=pending sync, 1=confirmed on peer
);

CREATE TABLE audit (
    id      INTEGER PRIMARY KEY AUTOINCREMENT,
    event   TEXT,    -- enc_served | dec_served | synced | sync_recv | dec_miss_*
    key_id  TEXT,
    peer    TEXT,
    detail  TEXT,    -- e.g. "batch=40" or "already_used peer_col=alice"
    ts      REAL
);

CREATE TABLE metrics (
    ts    REAL,
    peer  TEXT,
    event TEXT,
    qber  REAL
);
```

### The `synced` column — critical design

**Why it exists:**  
`enc_keys` must only return keys that are *already in Bob's DB*. Without this, if a key's sync HTTP call is in-flight or failed, Alice can hand it to the app, Bob can't decrypt it, and you get a 404.

**Lifecycle:**
```
_fill() generates keys
    → INSERT with synced=0
    → _sync_to_peer() POSTs to Bob's /sync/alice
        → on HTTP 200: _mark_synced() sets synced=1 on Alice's copy
        → on failure: key goes to retry queue, synced stays 0
_pop() (enc_keys): WHERE used=0 AND synced=1  ← only confirmed keys
```

Bob's copy is inserted with `synced=1` immediately upon receipt (it's already "here").

### Key lifecycle state machine

```
              generate
                 │
                 ▼
         [used=0, synced=0]
                 │
         POST /sync/peer
                 │
         ┌───────┴───────┐
         │ 200 OK        │ failure
         ▼               ▼
  [used=0, synced=1]   retry queue
         │               │
  enc_keys eligible      │ async retry succeeds
         │               │
         ▼               ▼
  [used=1, synced=1]  [used=0, synced=1]
     (consumed)        (now eligible)
```

### API endpoints

#### ETSI QKD 014 standard endpoints

```
GET  /api/v1/keys/{peer}/status
     → stored_key_count (synced only), unsynced_key_count, avg_qber, link_state

GET  /api/v1/keys/{peer}/enc_keys?number=N
     → {"keys": [{"key_ID": "<uuid>", "key": "<hex>"}]}
     Only returns synced=1, used=0 keys. Marks them used=1 immediately.

POST /api/v1/keys/{peer}/dec_keys
     Body: {"key_IDs": [{"key_ID": "<uuid>"}]}
     → {"keys": [{"key_ID": "<uuid>", "key": "<hex>"}]}
     Finds key by ID, marks used=1. No peer filter (key_id is globally unique).

GET  /api/v1/audit?limit=N
     → list of audit events, most recent first
```

#### Internal / operational endpoints

```
POST /sync/{source_node}
     Body: [{"key_id", "key_hex", "qber", "ts"}, ...]
     Receives pre-shared keys from the other KMS. Inserted with synced=1.

GET  /health
     Node status, pool stats per peer

GET  /metrics
     Prometheus-format text: pool depth, avg QBER, total generated

POST /admin/refill/{peer}
     Force an immediate pool refill (background task)

GET  /debug/keys/{peer}?limit=N
     Full DB dump: every key with used/synced state — transparency endpoint

GET  /debug/audit?limit=N
     Extended audit log with detail field
```

### Background tasks (startup → forever)

**`_bg_fill(peer)`** — Runs at startup for each peer. Calls `_fill()` in a thread pool executor so it doesn't block the event loop.

**`_refill_loop()`** — Every `SYNC_INT` seconds: if `_count(peer, synced_only=False) < REFILL_AT`, call `_fill()`. Uses total (not just synced) to avoid over-generating while syncs are pending.

**`_sync_retry_loop()`** — Every 4 seconds: drains the retry queue. Uses `httpx.AsyncClient` (non-blocking). On success, calls `_mark_synced()`. On failure, re-queues.

### Thread safety

- Single `sqlite3.Connection` with `check_same_thread=False`
- `threading.Lock` (`_lock`) guards all DB read/write operations
- `threading.Lock` (`_SYNC_RETRY_LOCK`) guards the retry queue
- `_fill()` runs in `run_in_executor` (thread pool) — holds `_lock` only during INSERT, not during HTTP calls

---

## 5. SDN Controller

**File:** `sdn/sdn.py`  
**Framework:** FastAPI + uvicorn  
**Algorithms:** BFD, OSPF-inspired LSDB, QBER-weighted Dijkstra, LFA Fast Reroute, ECMP

### Link State Database (`LinkStateDB`)

Central data structure. Holds:
- `nodes: dict[name, info]` — live KMS node metadata
- `links: dict[tuple(a,b), link_info]` — per-link QBER/state/latency
- `bfd: dict[name, {misses, last_ok}]` — BFD probe counters

**Edge weight formula:**
```python
weight = qber * 1000 + (10 / key_rate) + latency_ms * 0.01
```
- QBER dominates: a link with QBER=0.05 costs 50 vs QBER=0.001 costs 1
- Key rate secondary: slower key generation → higher cost
- Latency minimal: only matters for equal-QBER links
- `LinkState.DOWN` → `weight = math.inf` → Dijkstra automatically avoids

**Link state transitions:**
```
QBER < 0.08  → UP
0.08 ≤ QBER < 0.11  → DEGRADED  
QBER ≥ 0.11  → DOWN
```

### BFD (Bidirectional Forwarding Detection)

`_bfd_loop()` probes every node via `GET /health` every `PROBE_INT=1.0` second.  
- 3 consecutive misses (`BFD_MULTIPLIER=3`) → node marked DOWN → all its links set to `LinkState.DOWN`
- Recovery: next successful probe → misses reset to 0 → links re-evaluated on next topology poll

### Topology polling

`_topology_loop()` runs every `POLL_INT=8.0` seconds:
1. For each KMS node: `GET /health` + `GET /api/v1/keys/{peer}/status`
2. Latency measured as round-trip time of the status call
3. `update_node()` writes to LSDB under async lock

### Dijkstra SPF

Standard single-source shortest path. Returns `{dst: (cost, next_hop, path)}`.
- Source node: cost=0
- Relaxation: only over non-DOWN links
- Result drives `/path/{src}/{dst}` routing decisions

### LFA Fast Reroute (RFC 5286)

For each source→dest pair, finds a loop-free alternate (LFA) neighbor `N` where:
```
dist(N, D) < dist(N, S) + dist(S, D)
```
This guarantees that if the primary path fails, traffic rerouted through N will reach D without looping back through S.

### ECMP (Equal-Cost Multi-Path)

BFS from source, collecting all paths with cost ≤ `best_cost × 1.05` (5% tolerance). Returns up to 4 paths for load balancing.

### WebSocket live push

`_push_loop()` broadcasts topology JSON to all connected WebSocket clients every 2 seconds. Clients that disconnect are pruned silently.

### API endpoints

```
GET /health              → service status
GET /topology            → full nodes + links + bfd state
GET /path/{src}/{dst}    → primary path + cost + LFA backup + ECMP paths
GET /spf/{src}           → full SPF tree from source
GET /links               → all links with computed weights
GET /bfd                 → BFD state per node
WS  /ws                  → live topology push every 2s
```

---

## 6. Chat Application

**File:** `app/chat.py`  
**Mode:** Terminal UI, asyncio, AES-256-GCM per message

### Send flow

```
User types message
    │
    ▼
GET /api/v1/keys/{peer}/enc_keys  ← Alice KMS
    │ returns: key_ID + 32-byte key
    ▼
nonce = os.urandom(12)            ← fresh 96-bit nonce every message
ct = AESGCM(key).encrypt(nonce, plaintext, None)
    │
    ▼
TCP → Relay:  {type:msg, from, to, key_id, nonce:b64, ciphertext:b64}
    │
    ▼
Key bytes discarded — never stored, never sent over network
```

### Receive flow

```
TCP ← Relay:  {from, key_id, nonce, ciphertext}
    │
    ▼
POST /api/v1/keys/{peer}/dec_keys  ← Bob KMS (local)
    │ returns: 32-byte key by UUID
    ▼
pt = AESGCM(key).decrypt(nonce, ct, None)
    │
    ▼
Rendered to terminal
```

### Commands

| Command | Action |
|---------|--------|
| `/status` | Poll KMS for pool depth, QBER, link state |
| `/pool` | Alias for `/status` |
| `/eavesdrop` | Run `run_bb84(..., eavesdrop=True)` and show QBER result |
| `/quit` | Graceful disconnect |

### Relay reconnection

`receiver()` background task checks `relay.connected` every 4 seconds. If disconnected, calls `relay.reconnect()` which waits 3s then re-registers with the relay server.

---

## 7. Relay Server

**File:** `app/relay.py`  
**Protocol:** Plain TCP, newline-delimited JSON  
**Supported nodes:** Exactly two — `alice` and `bob`

### Protocol

```
Client → Server:  {"type": "register", "node": "alice"}   (first message)
Client → Server:  {"type": "ping"}
Server → Client:  {"type": "pong", "ts": <unix>}
Client → Server:  {"type": "msg", "to": "bob", "key_id": ..., ...}
Server → "bob":   (same dict, with "relay_ts" added)
```

### Offline queuing

If the destination client is not connected when a message arrives, it's appended to `_queue[dst]` (a `deque` with `maxlen=200`). When the destination connects, the queue is drained first.

### Connection state

`_clients: dict[str, asyncio.StreamWriter]` — only one writer per node name. If a node reconnects, the old entry is overwritten. Disconnection removes the entry.

### Design constraints

- No TLS — relay is inside WireGuard VPN (encrypted at network layer)
- No authentication — nodes are identified by self-declared name
- No key bytes ever appear in relay messages — only UUIDs and ciphertexts

---

## 8. Dashboard

**File:** `dashboard/server.py`  
**Stack:** Python stdlib `http.server` + background `asyncio` thread for polling  
**Refresh:** `<meta http-equiv="refresh" content="5">` — pure server-side

### Architecture

```
HTTPServer (main thread, sync)
    ├── GET /          → _render() → HTML string with inline state
    ├── GET /api       → JSON dump of _state
    ├── GET /sdn       → proxies GET /topology from SDN controller
    └── GET /health    → {"status":"ok"}

asyncio event loop (daemon thread)
    └── _fetch_loop()  every 5s:
        ├── GET {SDN_URL}/topology → _state["nodes"], _state["links"], _state["bfd"]
        └── GET {KMS}/health for each node → enriches _state["nodes"]
```

### Rendered data

- Per-node cards: status (up/down/degraded), port, dist_km, peers, BFD misses
- Links table: state, QBER, keys available, latency
- BFD state derived from miss count (≥3 misses = down)
- All CSS inline, no external dependencies

---

## 9. Deployment Pipeline

### `setup/wg_gen.py`

Generates three WireGuard configs using the `wg` CLI:
- `wg_vps.conf` — VPS hub with two peer stanzas (Alice + Bob)
- `wg_alice.conf` — Pi, routes all 10.8.0.0/24 through VPS
- `wg_bob.conf` — Radxa, same

Also outputs `wg_keys.txt` (keep private) and `wg_configs/deploy.sh` (SCP + activate).

### `setup/setup_vps.sh`

Runs on VPS as root. Actions:
1. `apt install python3 python3-pip wireguard`
2. `pip install fastapi uvicorn httpx cryptography`
3. `cp -r /tmp/qkd_mvp /opt/qkd`
4. Write three systemd unit files: `qkd-sdn`, `qkd-relay`, `qkd-dashboard`
5. `systemctl enable --now` all three
6. Open UFW/iptables ports: 7000, 8888, 9001, 51820/udp

**Systemd unit pattern:**
```ini
[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/qkd/sdn/sdn.py
Environment=KMS_NODES=alice:10.8.0.2:7001,bob:10.8.0.3:7002
Restart=always
RestartSec=3
```

### `setup/setup_node.sh`

Runs on Pi or Radxa as root. Actions:
1. Install deps
2. Deploy code to `/opt/qkd`
3. Detect ethernet interface (`eth0`, `end0`, `enp3s0`, etc.)
4. Assign static IP `192.168.100.x/24` via `systemd-networkd`
5. Write + enable `qkd-kms.service`

KMS service invocation:
```bash
# Pi (alice)
python3 /opt/qkd/kms/kms.py alice 7001 bob:10.8.0.3:7002

# Radxa (bob)  
python3 /opt/qkd/kms/kms.py bob 7002 alice:10.8.0.2:7001
```

### `deploy.sh`

One-command Mac → all-nodes deploy:
```
Step 1: wg_gen.py  →  wg_configs/
Step 2: rsync code to VPS + Pi + Radxa
Step 3: SCP WG configs + wg-quick up on all three
Step 4: SSH → setup_vps.sh on VPS
         SSH → setup_node.sh alice on Pi
         SSH → setup_node.sh bob on Radxa
Step 5: curl smoke tests on all HTTP endpoints
```

---

## 10. Integration Test

**File:** `test_local.py`  
**Runs:** All services as subprocesses on `localhost`, no hardware needed

### Test sequence

```
1. BB84 Engine (pure Python, no network)
   ├── Normal session: assert QBER < 0.11, key_hex len=64
   ├── Eve simulation: assert QBER > 0.11 or success=False
   └── Pool generation: assert >0 keys returned

2. KMS E2E (network, two subprocesses)
   ├── Wipe /tmp/qkdn_*.sqlite3*
   ├── Start Bob (:7002) — wait 5s until ready
   ├── Start Alice (:7001) — wait 6s (fill + sync to ready Bob)
   ├── Poll Alice status: wait for stored_key_count > 0 (synced keys)
   ├── Poll Bob status: wait for stored_key_count > 0
   ├── enc_keys on Alice → key_ID + 32-byte key
   ├── AES-256-GCM encrypt "hello QKD world"
   ├── Poll Bob /debug/keys/alice: confirm UUID present
   ├── dec_keys on Bob → same 32-byte key
   ├── Assert Alice key == Bob key
   ├── AES-256-GCM decrypt → assert "hello QKD world"
   └── Dump audit logs + DB summaries

3. SDN Controller (network, one subprocess, KMS still running)
   ├── Start SDN (:7000) pointing at :7001 + :7002
   ├── Wait 5s for topology poll to complete
   ├── Assert /topology has 2 nodes, 1 link
   ├── Assert /bfd shows both nodes up
   └── Assert /path/alice/bob returns a path

4. Chat Relay (network, one subprocess)
   ├── Start relay (:9001)
   ├── Open two raw TCP connections (alice + bob)
   ├── Register both nodes
   ├── Send message alice→bob via relay
   ├── Assert bob receives it with correct key_id
   └── Ping/pong test
```

### Startup order matters

Bob must start before Alice so Alice's first `_fill()` → `_sync_to_peer()` finds Bob ready. If Alice starts first, all 4 sync retry attempts may exhaust before Bob is ready, and the async retry queue adds ~6+ seconds of latency.

### Process cleanup

`stop_all()` calls `p.terminate()` for all tracked subprocesses, waits 2s, then `p.kill()` for any survivors. Called after all tests regardless of pass/fail.

---

## 11. Key Data Flows

### Flow A: Initial key pool setup

```
Alice KMS startup
    │
    ├─ _bg_fill("bob")  [thread pool]
    │       │
    │       ├─ generate_pool("alice↔bob", 5.0, n=40)
    │       │       └─ 40× run_bb84() → SHA3-256 → key_hex
    │       │
    │       ├─ INSERT 40 rows (synced=0) into Alice's SQLite
    │       │
    │       └─ _sync_to_peer("bob", rows)
    │               │
    │               ├─ POST http://bob:7002/sync/alice
    │               │       └─ Bob inserts 40 rows (synced=1) into Bob's SQLite
    │               │
    │               └─ _mark_synced("bob", [40 UUIDs])
    │                       └─ Alice sets synced=1 for these 40 rows
    │
    └─ _refill_loop() [async, every 2s]
            └─ if count < 12: _fill("bob")
```

### Flow B: Sending a chat message

```
Alice app: user types "hello"
    │
    ├─ GET /api/v1/keys/bob/enc_keys?number=1  (Alice KMS :7001)
    │       └─ _pop("bob") → SELECT synced=1, used=0 LIMIT 1
    │                         UPDATE used=1
    │                         → returns {key_ID: uuid, key: hex}
    │
    ├─ nonce = os.urandom(12)
    ├─ ct = AESGCM(key).encrypt(nonce, b"hello", None)
    │
    └─ TCP → Relay :9001
            {type:msg, from:alice, to:bob, key_id:uuid, nonce:b64(nonce), ciphertext:b64(ct)}
```

### Flow C: Receiving a chat message

```
Bob app: relay delivers message
    │
    ├─ POST /api/v1/keys/alice/dec_keys  (Bob KMS :7002)
    │       Body: {"key_IDs": [{"key_ID": uuid}]}
    │       └─ _consume(uuid, "alice")
    │               → SELECT key_hex WHERE key_id=uuid AND used=0
    │               → UPDATE used=1
    │               → returns key_hex
    │
    ├─ key = bytes.fromhex(key_hex)
    ├─ pt = AESGCM(key).decrypt(b64decode(nonce), b64decode(ct), None)
    │
    └─ render "hello" to terminal
```

### What never crosses the network

| Data | Stays local | Why |
|------|-------------|-----|
| Raw key bytes | In SQLite on each KMS node | Never serialized into relay messages |
| BB84 raw bits | Never stored | Only the SHA3-256 output is kept |
| Alice's key hex | Alice's DB only | Bob has the *same* bytes from /sync, not forwarded from Alice's enc_keys response |
| Plaintext | Never leaves the app | Encrypted before TCP send |

---

## 12. Security Model

### Threat model

| Threat | Mitigation |
|--------|------------|
| Eavesdropper on quantum channel | QBER > 11% → session aborted |
| MITM on key relay | Key bytes never on relay; only UUID + ciphertext |
| Replay attack | Each message uses a fresh QKD key (used=1 after single use) |
| Key reuse | `used` column enforced at DB level with SQLite lock |
| Stale keys after restart | `FRESH_DB=1` wipes DB; each deployment is clean |
| VPS compromise | VPS holds no key bytes; only routes opaque ciphertexts |
| WireGuard channel | Keys exchanged only over WG VPN; VPS→node traffic encrypted |

### Cryptographic primitives

| Primitive | Usage | Parameters |
|-----------|-------|-----------|
| BB84 + SHA3-256 | Key generation + privacy amplification | 256-bit output |
| AES-256-GCM | Message encryption | 96-bit random nonce, no AAD |
| WireGuard | Transport encryption | ChaCha20-Poly1305 |

### ETSI QKD 014 compliance

The KMS implements the standard REST interface:
- `enc_keys`: returns fresh key material for the encryptor
- `dec_keys`: retrieves key by ID for the decryptor  
- `status`: reports pool depth and link quality
- Keys are 256-bit, identified by UUID v4, single-use

---

## 13. Bug History & Fixes

### Bug 1 — FastAPI 0.136 multipart regression

**Symptom:** KMS crashed at import time on macOS with FastAPI ≥ 0.115:
```
RuntimeError: Form data requires "python-multipart" to be installed.
```
**Root cause:** FastAPI 0.115+ treats any route parameter typed as `list` or `dict` as form data. Both `/sync/{source_node}` (`body: list`) and `/dec_keys` (`body: dict`) triggered this.  
**Impact:** `/sync/` endpoint never registered → every sync attempt returned 422 → all 40 keys stayed `synced=0` → `enc_keys` had nothing to serve.  
**Fix:** Changed both handlers to `async def f(request: Request)` + `body = await request.json()`.

### Bug 2 — Missing `synced` column (design bug)

**Symptom:** `dec_keys: 404 {"detail":"Key IDs not found or already consumed"}`  
**Root cause:** `enc_keys` returned keys whose HTTP sync to the peer was still in-flight or queued for retry. Alice marked them `used=1`, but Bob's DB didn't have them yet.  
**Fix:** Added `synced INTEGER DEFAULT 0` column. `_pop()` filters `AND synced=1`. `_mark_synced()` is called only after receiving HTTP 200 from the peer's `/sync/` endpoint. Retry queue also calls `_mark_synced()` on belated success.

### Bug 3 — KMS startup order

**Symptom:** Even with Bug 2 fixed, synced count stayed 0 for 40+ seconds.  
**Root cause:** Alice started before Bob. All 4 sync retry attempts (at t≈4s, 5s, 7s, 9s) hit "connection refused" since Bob wasn't ready until t≈8s. Keys fell into the async retry queue. With Bug 1 also present, the retry loop was calling a broken `/sync/` endpoint.  
**Fix:** `test_local.py` now starts Bob (5s wait) before Alice (6s wait). Alice's first sync attempt at startup finds Bob already listening → succeeds on attempt 1 → keys marked synced immediately.

---

*Generated from codebase at sdn_03/ — 4/4 integration tests passing.*
