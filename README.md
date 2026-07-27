# NISSI Protocol Studio

Professional protocol engineering suite for IEC 60870-5-104.

## Status

- [x] Phase 1 — Folder structure
- [x] Phase 2 — Electron shell
- [x] Phase 3 — FastAPI backend (config/logging/connection managers, project + connection REST API, WebSocket live feed, IEC104 master engine)
- [x] Phase 4 — Landing page (protocol/mode dropdowns, auto-named projects, light/dark theme)
- [x] Phase 5 — Dashboard (toolbar, tabbed workspace, connection config, telegram/statistics tables, live WebSocket feed)
- [x] Phase 6 — Sidebar (rolled into the Tags/System/Channel tab structure)
- [x] Phase 7 — Ribbon (toolbar diagnostics: Ping, Interface Info, Device Manager)
- [~] Phase 8 — IEC104 module (master + slave engines and General Interrogation are live; full command set staged)
- [ ] Phase 10 — Telegram monitor (connection-event log works; full ASDU/PDU hex+decoded view pending)
- [~] Phase 11 — Tag table (CRUD works via modal editor; live polling pending)
- [ ] Phase 12 — Reports
- [x] Phase 13 — Settings (grouped IEC104 settings, persisted to the database)
- [x] Phase 14 — Packaging (electron-builder configured — see below)

`[~]` = partially implemented; the working pieces are real, the rest is staged with clear "arrives in Phase X" notices in the UI rather than fake functionality.

## Project Structure

```
NISSI-Protocol-Studio/
├── backend/
│   ├── main.py                   # FastAPI entrypoint — boots config, logging, DB, routers
│   ├── api/
│   │   ├── schemas.py             # Pydantic request/response models
│   │   └── routes/
│   │       ├── projects.py         # Project CRUD + connection params
│   │       ├── connections.py      # Connect / disconnect / status / General Interrogation
│   │       ├── tags.py             # Tag CRUD
│   │       ├── diagnostics.py      # Ping / interface info
│   │       ├── system.py           # Health check + app config
│   │       └── websocket.py        # /ws/live — pushes tag updates, status changes
│   ├── core/
│   │   ├── config_manager.py      # JSON config load/save
│   │   ├── logging_manager.py     # Rotating file + console logging
│   │   ├── connection_manager.py  # Routes connect/disconnect to the right protocol engine
│   │   └── websocket_manager.py   # Broadcasts events to connected dashboard clients
│   ├── protocols/
│   │   └── iec104/                 # master.py, slave.py, monitor.py, parser.py, points.py
│   ├── database/
│   │   └── models.py              # SQLAlchemy models: Project, Connection, Tag, EventLog
│   ├── config/                     # app_config.json (generated on first run)
│   ├── logs/                       # nissi.log (rotating)
│   ├── reports/
│   └── sessions/
├── frontend/
│   ├── index.html                  # Landing screen
│   ├── dashboard.html              # Engineering workspace
│   ├── css/
│   │   ├── tokens.css               # Shared design tokens
│   │   ├── light.css / dark.css     # Theme palettes
│   │   ├── landing.css              # Landing page layout
│   │   └── dashboard.css            # Workspace layout
│   └── js/
│       ├── api.js                   # REST client for the FastAPI backend
│       ├── app.js                   # Landing page logic
│       ├── wizard.js                # Project creation wizard
│       └── dashboard.js             # Workspace logic
├── electron/
│   ├── main.js                     # Spawns the backend (dev or packaged paths), opens the window
│   └── preload.js                  # Exposes window.nissi (app info, API/WS URLs, Device Manager IPC)
├── package.json                    # Includes electron-builder packaging config
├── requirements.txt
└── README.md
```

## Running in development

```powershell
cd "Nissi protocol studio"
venv\Scripts\Activate.ps1
pip install -r requirements.txt
npm install
npm start
```

Electron spawns `backend/main.py` automatically using the project's `venv`
Python interpreter, waits for `/api/system/health`, then opens the
landing page. Backend logs are prefixed `[backend]` in the same terminal.

To run just the backend for API testing:
```powershell
venv\Scripts\Activate.ps1
python backend/main.py
```
API available at `http://127.0.0.1:8642` (interactive docs at `/docs`).

## Modbus TCP / RTU

NISSI implements Modbus TCP and Modbus RTU master (client) and slave
(server/outstation) modes alongside IEC 60870-5-104, sharing the same
project/connection/tag database and dashboard UI.

### Feature summary

**Master (Client)** — TCP and RTU:
- Connect / disconnect, with auto-reconnect (`Auto reconnect` + `Reconnect (ms)`
  in the Settings tab — pymodbus retries the connection in the background
  using this interval; the dashboard's connection-status indicator reflects
  the real, live connection state, including a drop/recovery it didn't cause
  itself, e.g. a cable pulled and replugged).
- Read Coils (FC01), Read Discrete Inputs (FC02), Read Holding Registers
  (FC03), Read Input Registers (FC04).
- Write Single Coil (FC05), Write Single Register (FC06), Write Multiple
  Coils (FC15), Write Multiple Registers (FC16).
- Manual read/write and a scan loop from the **Poll** tab.
- Optional **server-side polling**: set `Poll interval (ms)` in the Modbus
  Settings group (Settings tab → Master) to a value `> 0` and every
  configured Tag is read on that interval by the backend itself — Tag
  values update (and broadcast over WebSocket) even with the dashboard's
  Poll tab closed, the same way an IEC104 Slave auto-reports independent of
  which tab is open. Leave it at `0` (default) to disable and rely solely on
  the manual/Poll-tab reads, exactly like before this feature was added.
- Timeout (`Timeout (ms)`) and per-call error logging.

**Slave (Server/Outstation)** — TCP and RTU:
- TCP: listens on a configurable bind IP + port. RTU: listens on the
  selected serial port.
- In-memory register datastore (Coils / Discrete Inputs / Holding
  Registers / Input Registers), seeded from the project's saved Tags,
  answering FC01/02/03/04/05/06/15/16.
- TCP accepts multiple concurrent master connections natively.
- Every write from a master updates the Tag's `last_value` in the database
  and pushes a `tag_update` WebSocket event immediately.

**RTU-specific**: serial port / baud rate / parity / stop bits / data bits
/ slave (unit) ID are all configurable from the toolbar and Settings tab.
CRC is validated by pymodbus's RTU framer on every frame (see
`backend/tests/test_modbus_rtu_engine.py` for an automated CRC-integrity
check). RTS line control (`Disable`/`Enable`/`Handshake`/`Toggle`) is
applied best-effort after connecting.

### Tag mapping fields

Each Tag maps a Modbus register to a project point:

| Field | Where it lives | Notes |
|---|---|---|
| `tag_name` | `Tag.name` | |
| `address` | `Tag.address` | Register address (e.g. `0` for Holding Register 40001, zero-based) |
| `function_code` | derived from `Tag.data_type` | `coil`/`discrete_input`/`holding_register`/`input_register` — the actual FC (1/2/3/4 for reads, 5/6/15/16 for writes) is chosen automatically based on this and whether it's a single or multi-value operation |
| `data_type` | `Tag.data_type` | as above |
| `slave_id` | `Tag.slave_address` | Modbus unit/slave ID this tag belongs to |
| `scaling_factor` | `Tag.scaling_factor` | Engineering-unit scaling: displayed/read value = raw register value × `scaling_factor`; writing a value encodes raw = value ÷ `scaling_factor`. Default `1.0` (no-op) |
| `byte_order` **and** `word_order` | `Tag.endian` (Tag modal label: **Byte order**) | One field, four values, each a byte-order × word-order combination: `little` (AB CD), `big` (CD AB), `little_swap` (BA DC), `big_swap`/Modicon (DC BA). Kept as a single field rather than two independent ones — Float/Double/Long formats only ever need one of these four canonical combinations, and splitting it into two knobs would let an engineer pick an internally-contradictory pair with no valid on-wire meaning. |
| `protocol_type` | `Project.protocol` | `modbus_tcp` or `modbus_rtu` — a project-level setting, not per-tag, since every tag in a project always uses that project's protocol |

`format` (`signed`/`unsigned`/`hex`/`binary`/`float`/`swapped_fp`/`float_inv`/
`long`/`long_inv`/`double`/`double_inv`) controls how many registers a tag
spans and how they're decoded/encoded — see
`backend/protocols/modbus/format_codec.py`.

CSV/XLSX Tag export and import both round-trip `slave_address`, `format`,
`endian`, and `scaling_factor` (previously export dropped these columns).

### Logging

Every Modbus connect, disconnect, read, write, exception, and timeout is
written to the `modbus_logs` table and shown (most recent first) on the
dashboard's **Diagnostics** tab (visible for Modbus projects only), with a
Refresh button. Query it directly via
`GET /api/connections/{project_id}/modbus-logs`.

### Automated tests

```powershell
backend\venv\Scripts\python.exe -m pytest backend\tests -q
```

Modbus-specific test files (`backend/tests/test_modbus_*.py`):
- `test_modbus_format_codec.py` — encode/decode round trips for every
  format × byte-order combination, plus scaling-factor helpers.
- `test_modbus_broadcasting_block.py` — slave write → WebSocket broadcast
  callback wiring.
- `test_modbus_tcp_loopback.py` — a **real** `ModbusTcpMasterEngine`
  talking to a **real** `ModbusTcpSlaveEngine` over `127.0.0.1`. This is
  the automated stand-in for the two-laptop TCP scenario below — same wire
  protocol, same PDUs, just without a second physical host.
- `test_modbus_rtu_engine.py` — RTU config defaults, pre-connect error
  handling, and a CRC16 frame-integrity check via pymodbus's own RTU framer.
- `test_modbus_tags_import_export.py` — CSV/XLSX column round-trip.
- `test_modbus_models_migration.py` — `modbus_logs` table + `scaling_factor`
  column provisioning, including on an already-existing database.

### Manual hardware tests (not automatable in CI — run these yourself)

**Two-laptop Modbus TCP test** (matches the task scenario exactly):

1. **Laptop A** — create a `modbus_tcp` / `slave` project. In the toolbar,
   set bind IP to `192.168.1.100`, port `502` (needs to be reachable on
   your LAN — adjust the IP to your actual NIC address if different). Add
   a Holding Register Tag at address `0` (= "40001") with a Value. Connect.
2. **Laptop B** — create a `modbus_tcp` / `master` project, IP
   `192.168.1.100`, port `502`. Connect (status indicator should show
   Connected). Open the **Poll** tab, select "Write Single Register", start
   address `0`, write `1234`. Switch to "Read Holding Registers", address
   `0`, count `1`, Read — confirm the grid shows `1234`.
3. Check **Laptop A**'s Statistic tab / Diagnostics tab — the write should
   appear there too (slave-side broadcast).

**RTU loopback / USB-RS485 test**:

Either install [com0com](https://sourceforge.net/projects/com0com/) to
create a virtual COM pair (e.g. `COM10`↔`COM11`), or wire two USB-RS485
converters together (A↔A, B↔B, common ground).

1. Create a `modbus_rtu` / `slave` project on one COM port (e.g. `COM11`),
   matching baud/parity/stop/data bits on both ends (defaults: 9600 8N1).
   Add a Holding Register Tag at address `0`. Connect.
2. Create a `modbus_rtu` / `master` project on the other COM port (`COM10`),
   same serial settings. Connect. Use the Poll tab to write a value to
   address `0`, then read it back — confirm it matches, and confirm the
   Diagnostics tab logs the read/write with no `exception`/`timeout`
   entries (a CRC or framing failure resulting from a serial-settings
   mismatch would show up there as an `exception`).

## Building a Windows installer (.exe)

The app packages into a real NSIS installer using **electron-builder**.
The Python backend's virtual environment is bundled alongside the app
(not compiled to a standalone exe), so the install is larger but far less
likely to break than a PyInstaller build with native extensions like `c104`.

### Build the installer
```powershell
npm install
npm run dist
```

This produces `dist\NISSI Protocol Studio-Setup-0.1.0.exe`. Running it
installs the app to the user's chosen directory (not admin-elevated —
`perMachine: false`), creates Start Menu and desktop shortcuts, and
bundles:
- The Electron shell + frontend (packed into `app.asar`)
- `backend/` (Python source) copied as a plain resource folder
- `venv/` (the entire virtual environment, including `c104`)
  copied as a plain resource folder

### Where the installed app stores its data
Program Files (or wherever it's installed) isn't writable for a
non-admin install, so the packaged app's database, config, and logs go to
the OS-standard per-user app data folder instead:
```
C:\Users\<you>\AppData\Roaming\nissi-protocol-studio\
```
This only applies to the packaged build — in development (`npm start`),
everything still lives under `backend\database`, `backend\config`,
`backend\logs` exactly as before.

### Quick test without building a full installer
```powershell
npm run dist:dir
```
Packages the app into `dist\win-unpacked\` without wrapping it in an
installer — a fast way to confirm the bundled `venv` and backend launch
correctly before generating the full `.exe`.

### Known limitation
The bundled `venv` makes the installer noticeably larger (`c104`
isn't tiny). If install size becomes a problem, the
PyInstaller route (compiling `backend/main.py` into a standalone
`nissi-backend.exe` with no Python install required on the target
machine) is the next step.
"# nissi-protocol-studio-104-only" 
"# nissi-protocol-studio-104-modbus" 
