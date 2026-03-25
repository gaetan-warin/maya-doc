# Water Device Configuration Architecture

**Last updated:** 2026-03-18

## The Problem

Today, credentials and device config are tangled together in different places per integration. Adding a device requires touching 2-3 systems and sometimes redeploying code.

| | Masgrau | Shayp | Lynx |
|---|---|---|---|
| **Where are credentials?** | ETL credential store (`modbus` type: host, port, groups) | ETL credential store (`bearer` type) or env var | Env vars on Windows machine |
| **Where is device config?** | Baked INTO the credential `groups` field: `PG1:Nord:100:masgrau-infinitum-pg1` | **Hardcoded in Python source** (`SHAYP_DEVICES` list) | Env vars (`LYNX_CLUB_IDENTIFIER`, `LYNX_UNIT`, etc.) |
| **Where is tenant mapping?** | Maya back office (`connected_water_meter_devices` table) | Maya back office (same table) | Maya back office (`lynx_club_configs` table) |
| **To add a device** | 1. Create in back office  2. Edit credential store `groups` string | 1. Create in back office  2. Edit `.py` source  3. Redeploy | 1. Create in back office  2. Install new agent on-prem |

The root cause: **two different concerns are mixed**.

---

## The Two Concerns

### 1. CREDENTIALS = "How do I connect to an external system?"

Secrets that give access to a data source. These are **per-connection**, not per-device.

| Integration | Credential | Scope |
|---|---|---|
| **Masgrau** | Modbus TCP host + port | One credential per physical Modbus gateway (one gateway may serve multiple pumps) |
| **Shayp** | Bearer token for Shayp API (`wall-e.cyc2.be`) | One token for ALL Shayp meters globally |
| **Lynx** | SQL Server host + user + password | One credential per on-prem Lynx server (one per club) |

**Where they should live:** ETL credential store (encrypted SQLite). These are infrastructure secrets — the ETL needs them to connect, Maya doesn't.

### 2. DEVICES = "What should I fetch and for which customer?"

Business configuration that maps external identifiers to Maya tenants. These are **per-device**.

| Integration | Device config | Links to |
|---|---|---|
| **Masgrau** | `device_reference_id` + `register_base` (which Modbus registers to read) | A tenant + water source in Maya |
| **Shayp** | `shayp_meter_id` + `tz_offset` (which meter to query from Shayp API) | A tenant + water source in Maya |
| **Lynx** | `club_identifier` + `unit` + `timezone` + `irrigation_day_start` | A tenant in Maya |

**Where they should live:** Maya back office (the Water Devices page we built). This is business config — admins manage it, and the ETL pulls it.

---

## Target Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MAYA BACK OFFICE                          │
│                                                             │
│  Water Devices page                                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Name          │ Type    │ Tenant       │ Config     │    │
│  │───────────────│─────────│──────────────│────────────│    │
│  │ PG1 - Nord    │ Masgrau │ Infinitum    │ reg:100    │    │
│  │ PG2 - Centre  │ Masgrau │ Infinitum    │ reg:200    │    │
│  │ Naxhelet      │ Shayp   │ Naxhelet GC  │ N1C98504  │    │
│  │ Adare Manor   │ Lynx    │ Adare Manor  │ gallons,   │    │
│  │               │         │              │ 15:55, ... │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  GET /api/v2/etl/water-devices?type=masgrau                 │
│  → returns devices + tenant mapping + device-specific config│
│    (NO secrets — just business config)                      │
└─────────────────────────────────────────────────────────────┘
          ▲                              │
          │ ETL pulls device list        │ ETL pushes readings
          │ on every run                 ▼
┌─────────────────────────────────────────────────────────────┐
│                    ETL ORCHESTRATOR                          │
│                                                             │
│  Credential Store (encrypted SQLite)                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Name              │ Type    │ Fields                 │    │
│  │───────────────────│─────────│────────────────────────│    │
│  │ Infinitum Modbus  │ modbus  │ host:10.0.1.50 port:502│   │
│  │ Shayp API         │ bearer  │ bearer_token: xxx...   │    │
│  │ Adare Lynx DB     │ sqlsrv  │ host:10.0.2.1 user:... │   │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Credentials = HOW to connect (secrets, per-connection)     │
│  Devices     = WHAT to fetch  (pulled from Maya, per-device)│
└─────────────────────────────────────────────────────────────┘
```

---

## How Each Integration Would Work

### Masgrau (Modbus connected meters)

**Credential store** (one entry per Modbus gateway):
```
Type: modbus
Name: "Infinitum Living Modbus"
Fields: { host: "10.0.1.50", port: 502, timeout: 10 }
```

Note: **no `groups` field anymore** — devices are pulled from Maya.

**Maya back office** (one entry per pump):
```
Type: masgrau
Name: "PG1 - Nord"
Tenant: Infinitum Living
device_reference_id: masgrau-infinitum-pg1
register_base: 100              ← new field (device-specific Modbus config)
credential_name: infinitum      ← links to which credential to use
```

**ETL agent flow:**
```python
# 1. Pull device list from Maya
devices = GET /api/v2/etl/water-devices?type=masgrau
# → [
#   { device_reference_id: "masgrau-infinitum-pg1", register_base: 100,
#     credential_name: "infinitum" },
#   { device_reference_id: "masgrau-infinitum-pg2", register_base: 200,
#     credential_name: "infinitum" },
# ]

# 2. Group devices by credential_name
# → { "infinitum": [pg1, pg2, pg3] }

# 3. For each group, get connection from local credential store
cred = get_credentials("masgrau", "modbus", name="infinitum")
# → { host: "10.0.1.50", port: 502, timeout: 10 }

# 4. Connect and read each device's registers
for device in group:
    reading = read_modbus(cred.host, cred.port, device.register_base)
    POST /api/v2/water/hourly-records { device_reference_id, reading }
```

### Shayp (HTTP API connected meters)

**Credential store** (one entry — global Shayp API access):
```
Type: bearer
Name: "Shayp API"
Fields: { bearer_token: "eyJ..." }
```

**Maya back office** (one entry per meter):
```
Type: shayp
Name: "Naxhelet"
Tenant: Naxhelet Golf Club
device_reference_id: N5E4435C
shayp_meter_id: N1C98504        ← new field (Shayp's internal meter ID)
tz_offset: 2                    ← new field (device timezone offset)
```

**ETL agent flow:**
```python
# 1. Pull device list from Maya
devices = GET /api/v2/etl/water-devices?type=shayp
# → [{ device_reference_id: "N5E4435C", shayp_meter_id: "N1C98504", tz_offset: 2 }]

# 2. Get Shayp API token from local credential store
cred = get_credentials("shayp", "bearer")

# 3. For each device, query Shayp API
for device in devices:
    data = GET https://wall-e.cyc2.be/claws/shayp/get?meter={device.shayp_meter_id}
    convert(data, device.tz_offset)
    POST /api/v2/water/hourly-records { device_reference_id, readings }
```

**No more hardcoded `SHAYP_DEVICES` list.** Add a meter in back office → ETL picks it up.

### Lynx (Toro irrigation system)

**Credential store** (one entry per club's on-prem SQL Server):
```
Type: sqlserver     ← new credential type
Name: "Adare Manor Lynx DB"
Fields: { host: "10.0.2.1", port: 1433, database: "lynx_main",
          user: "maya_ro", password: "xxx" }
```

**Maya back office** (one entry per club — already exists):
```
Type: lynx
Name: "Adare Manor"
Tenant: Adare Manor
club_identifier: adare-manor
unit: gallons
timezone: Europe/Dublin
irrigation_day_start: 15:55
api_key: lynx_xxx...            ← for pushing back to Maya
credential_name: adare-manor    ← links to which SQL Server credential
```

**ETL agent flow (centralized, replaces per-club agent installs):**
```python
# 1. Pull club list from Maya
clubs = GET /api/v2/etl/water-devices?type=lynx
# → [{ club_identifier: "adare-manor", unit: "gallons", timezone: "Europe/Dublin",
#       irrigation_day_start: "15:55", api_key: "lynx_xxx...",
#       credential_name: "adare-manor" }]

# 2. For each club, get SQL Server credentials from local store
for club in clubs:
    cred = get_credentials("lynx", "sqlserver", name=club.credential_name)

    # 3. Connect to on-prem SQL Server and fetch data
    actuals = fetch_actuals(cred.host, cred.user, cred.password, ...)
    scheduled = fetch_scheduled(...)
    zones = reconcile(actuals, scheduled, club.irrigation_day_start)

    # 4. Push to Maya
    POST /api/v2/lynx/sync
      headers: { X-Lynx-Api-Key: club.api_key }
      body: { club_identifier, records: zones }
```

**Benefits for Lynx:**
- One centralized agent serves multiple clubs (no per-club Windows install)
- Add a club in back office → agent discovers it on next run
- SQL Server credentials stay local (on-prem secrets never touch cloud)

---

## What Needs to Change

### Phase 1: Maya Back Office (extend current form)

Add integration-specific config fields to the Water Devices form:

| Integration | New fields to add |
|---|---|
| **Masgrau** | `register_base` (int), `credential_name` (text — links to ETL credential) |
| **Shayp** | `shayp_meter_id` (text), `tz_offset` (int) |
| **Lynx** | `credential_name` (text — already has all other fields) |

These fields are stored in a JSON `config` column or as dedicated columns on the device tables.

### Phase 2: Maya API (new endpoint)

```php
// routes/api/water.php
Route::get('etl/water-devices', [WaterDeviceConfigController::class, 'index'])
    ->middleware('etl-api-key');  // service-to-service auth, not user JWT
```

Returns active devices grouped by type with all config the ETL needs. **No secrets in the response** — secrets stay in the ETL credential store.

### Phase 3: ETL Agent Changes

| Agent | Change |
|---|---|
| `masgrau.py` | Replace `load_clients()` with Maya API call. Remove `groups` from credential schema. |
| `shayp.py` | Replace hardcoded `SHAYP_DEVICES` with Maya API call. |
| `lynx/main.py` | Replace env-var-per-club with Maya API call. Support multi-club in one agent. |

Add new credential type to ETL:
```python
# models.py CREDENTIAL_SCHEMAS
"sqlserver": [
    {"name": "host", "label": "Host", "type": "text", "required": True},
    {"name": "port", "label": "Port", "type": "number", "default": "1433"},
    {"name": "database", "label": "Database", "type": "text", "default": "lynx_main"},
    {"name": "user", "label": "Username", "type": "text", "required": True},
    {"name": "password", "label": "Password", "type": "password", "required": True},
],
```

### Phase 4: Simplify Masgrau credential

Current `modbus` credential schema has a `groups` field that mixes connection + device config:
```
groups: "PG1:Nord:100:masgrau-infinitum-pg1,PG2:Centre:200:masgrau-infinitum-pg2"
```

New schema — connection only:
```
host: "10.0.1.50"
port: 502
timeout: 10
```

Device info (register_base, device_reference_id) moves to Maya back office.

---

## The Resulting Flow (for all three)

```
ADMIN adds device in back office
  → Stored in Maya DB (tenant, device config, status)
  → For Lynx: API key generated and stored

ETL agent starts a run
  → Calls GET /api/v2/etl/water-devices?type={masgrau|shayp|lynx}
  → Gets list of active devices with their config
  → Groups by credential_name
  → For each group:
      → Loads connection secrets from local credential store
      → Connects to external system (Modbus / Shayp API / SQL Server)
      → Reads data for each device
      → POSTs to Maya API (hourly-records or lynx/sync)

Scheduled commands do the rest (aggregation, promotion)
  → water_readings created → visible on Water Page
```

**One flow. Admin creates device in one place. ETL discovers it automatically.**

---

## Migration Path

1. Build the Maya API endpoint and add config fields to back office form
2. Update ETL agents to support Maya API as a config source
3. Keep env-var/credential-store fallback during transition (`load_from_maya() || load_from_env()`)
4. Migrate existing devices: create them in back office, verify ETL picks them up
5. Remove hardcoded config from agents
6. Update credential store: remove `groups` from `modbus` schema, add `sqlserver` type

**No big bang.** Each agent can be migrated independently with fallback to current behavior.

---

## Security Notes

- **Maya API for device config** uses a service API key (not user JWT) — the ETL authenticates as a service
- **Connection secrets** (Modbus IPs, SQL Server passwords, Shayp tokens) **never leave the ETL** — they stay in the local encrypted credential store
- **Lynx API keys** are the one exception: generated in Maya, stored in Maya, returned to ETL via the config endpoint. This is by design — the Lynx agent needs the key to push data back to Maya
- The `GET /api/v2/etl/water-devices` endpoint should be read-only and rate-limited
