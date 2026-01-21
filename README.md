
# M3 Telematics System — Minimal, Scalable Design

## 1. Goal

Design a **simple, robust telematics pipeline** for the BMW E46 M3 that:

- Runs **without iOS apps or Apple developer accounts**
- Buffers data locally on a **Raspberry Pi (Debian)** using SQLite
- Writes data **directly to a TimescaleDB hypertable** on Phoenix over VPN
- Scales cleanly from *hello‑world* to *years of telemetry*
- Can be incrementally extended without rewrites

This document intentionally starts minimal and grows in small, safe steps.

---

## 2. High‑Level Architecture

```
Car (ELM327 / sensors)
        |
        v
Raspberry Pi (Debian)
- SQLite buffer (durable queue)
- Batch flush logic
        |
        |  (VPN / Tailscale)
        v
Phoenix
- PostgreSQL + TimescaleDB
- Hypertable for samples
- Retention + compression
```

No Flask, no REST server, no reverse proxy.  
**The database protocol *is* the ingestion API.**

---

## 3. Why This Scales

### Inserts
- TimescaleDB handles **millions → tens of millions of rows** easily.
- Inserts are append‑only and chunked by time.

### Storage
Rule of thumb:
- ~100–300 bytes per sample (JSONB + indexes)
- 10 Hz logging → ~1–2 GB / month
- 50 Hz logging → ~5–10 GB / month

A 1–2 TB disk = **years of data**.

### Bottlenecks
- Disk capacity (not CPU)
- SD card wear on Pi (mitigated by batching)

---

## 4. Phoenix Setup (TimescaleDB)

### 4.1 Table + Hypertable

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE samples (
  ts        TIMESTAMPTZ NOT NULL,
  device_id TEXT        NOT NULL,
  sample_id UUID        NOT NULL,
  payload   JSONB       NOT NULL,
  PRIMARY KEY (device_id, sample_id)
);

SELECT create_hypertable('samples', 'ts', if_not_exists => TRUE);
CREATE INDEX ON samples (device_id, ts DESC);
```

### 4.2 Retention Policy

Drop old raw data automatically:

```sql
SELECT add_retention_policy('samples', INTERVAL '12 months');
```

Optional later:
- Compression after 7–30 days
- Downsampling tables (1 Hz forever)

---

## 5. Pi‑Side Buffer (SQLite)

### 5.1 Purpose
- Survive loss of connectivity
- Batch writes to protect SD card
- Guarantee at‑least‑once delivery

### 5.2 SQLite Queue Schema

```sql
CREATE TABLE queue (
  sample_id TEXT PRIMARY KEY,
  ts        TEXT NOT NULL,
  device_id TEXT NOT NULL,
  payload   TEXT NOT NULL,
  sent      INTEGER NOT NULL DEFAULT 0
);
```

### 5.3 Size Limits (Decision)
- Target max offline window: **~7 days**
- SQLite cap: **1–2 GB**
- Delete oldest `sent=1` rows first
- Drop oldest unsent rows only as last resort

---

## 6. Minimal Hello‑World Flow

### Step 1 — Enqueue a Sample (Pi)

```python
payload = {"msg": "hello world"}
INSERT INTO queue (...)
```

### Step 2 — Flush Batch to Phoenix

```sql
INSERT INTO samples (ts, device_id, sample_id, payload)
VALUES (...)
ON CONFLICT DO NOTHING;
```

### Step 3 — Mark as Sent

```sql
UPDATE queue SET sent=1 WHERE sample_id=?;
```

That’s it. The data flow is validated.

---

## 7. Increment Plan (Safe Steps)

### Increment 1 — Hello World (now)
- SQLite buffer
- Timescale hypertable
- One INSERT batch

### Increment 2 — Periodic Loop
- Enqueue at fixed rate
- Flush every N seconds
- Backoff on DB failure

### Increment 3 — Real Telemetry
- ELM327 BLE reader
- Add RPM / speed / temps to payload

### Increment 4 — Data Hygiene
- SQLite cleanup job
- Phoenix compression + retention

### Increment 5 — Analytics
- Grafana dashboards
- Track sessions / trips

Each step is independent and reversible.

---

## 8. Design Principles

- **Append‑only everywhere**
- **At‑least‑once delivery**
- **Idempotent inserts**
- **Local durability > network reliability**
- **Schema‑light at ingestion, structured at query time**

---

## 9. Why This Is the Right Foundation

- Minimal moving parts
- Easy to debug with `psql` and `sqlite3`
- No app servers to babysit
- Grows naturally into a serious telemetry system

This is not a prototype — it’s a clean long‑term architecture.

---

*End of document*
