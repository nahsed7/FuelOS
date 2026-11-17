# FuelOS · Hongyang CMD1687SK-GA44 Digital Twin

An interactive 3D digital twin and live-monitoring dashboard for a **Hongyang A-Series CMD1687SK-GA44** fuel dispenser. It covers two operating sides (A and B) and four dispensing positions (A1, A2, B1, B2). It runs in the browser with Three.js, and a small Node.js backend streams telemetry over Socket.IO.

> [!IMPORTANT]
> **All telemetry is simulated by default.** FuelOS is not connected to a physical dispenser. The physical-device mode is a **read-only scaffold** that shows every position as *Offline* until a manufacturer-documented protocol decoder is added. See [Physical integration](#physical-integration-read-only).

![3D dispenser view](docs/images/3d-dispenser.png)

---

## Features

### 3D digital twin
- **Reconstruction of the Hongyang A-Series cabinet**, based on the manufacturer's catalogue photo:
  - white logo canopy on two chamfered graphite columns
  - angled "ship-type" side wings
  - black-glass display head and white lower door with the wave graphic
  - vents, screws, door locks and an identification plate
- **Four separate dispensing positions** on opposite faces. Each has its own backlit LCD (sale / litres / price), membrane keypad, status lamp, nozzle holster, nozzle and hose.
- **Interactive nozzles.** Click a nozzle to lift it, drag it within a limited working area, then double-click it (or click its holster) to put it back. Each hose connects at the nozzle's swivel and keeps its length, so slack hangs or lies on the ground.
- **Working 3D keypads.** Type a preset, choose `L` (litres) or `$` (amount), then press `OK` to authorise a sale.
- **Camera controls:** orbit, zoom and pan, plus presets for front (A), rear (B), left, right, top, close-up and nozzle. Transitions are smooth and you can turn on auto-rotate.
- **View modes:**
  - **Normal**
  - **Technical:** see-through casing showing pumps, flowmeters, pipework and mainboards, with animated product flow
  - **Exploded**
- **Materials and lighting:** physically based materials, environment reflections, soft shadows and a forecourt pump island with safety kerb and bollards.

![Technical view](docs/images/technical-view.png)

### Monitoring dashboard
| Page | What it shows |
|---|---|
| **Overview** | Live KPIs, status of all four positions, pump activity, unit summary |
| **3D Dispenser** | The twin plus a control panel for the selected position |
| **Live Monitoring** | Per-position readouts, 2-minute flow-rate charts, pump state, status event feed |
| **Transactions** | Completed sales with CSV export |
| **Equipment** | Identity, GA44 hardware configuration, position mapping, what is confirmed vs. estimated |
| **Diagnostics** | Data-source link, Socket.IO state, latency, server stats, renderer stats, backend log |
| **Settings** | Simulation flow rate, product names and prices, 3D quality options |

Every status has its own colour **and** a text label and icon:

| Status | Meaning |
|---|---|
| 🟢 Available | Ready, nozzle in holster |
| 🟠 Nozzle removed / Paused | Needs attention |
| 🔵 Dispensing | Product flowing |
| 🟢 Completed | Sale finished |
| 🔴 Fault | Fault active |
| ⚪ Offline | No data |

![Live monitoring](docs/images/live-monitoring.png)

### Realistic simulation
- **State machine for each position:** idle → nozzle removed → dispensing ⇄ paused → completed, plus fault and offline states you can trigger yourself.
- **Calculated values.** Volume comes from the flow rate. Amount is volume × the unit price locked when the sale starts, so the figures always agree. The totalizer increases by exactly the volume dispensed.
- **Flow behaviour:** soft-start ramp, slow-down near the preset, and an exact stop at the preset. Flow drops when two positions share a pump.
- **Starting totalizers:** A1 `777.166 L`, A2 `777.583 L`, B1 `780.641 L`, B2 `776.158 L`.
- **Persistence:** totalizers, prices and transactions are saved to `data/` and survive restarts.

---

## Quick start

**Requirements:** Node.js 18+ (tested on Node 22) and a browser with WebGL.

```bash
git clone <your-repo-url> hongyang-ga44-fuelos
cd hongyang-ga44-fuelos
npm install
npm start
```

Open **http://localhost:3000**.

| Command | Description |
|---|---|
| `npm start` | Simulation mode (default) |
| `npm run start:physical` | Read-only physical mode (positions show *Offline* until a decoder is configured) |
| `npm run dev` | Auto-restart on changes (nodemon) |
| `npm run check` | Syntax-check the backend |

### Environment variables
| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `3000` | HTTP port |
| `HOST` | `0.0.0.0` | Bind address |
| `DATA_SOURCE` | `simulation` | `simulation` or `physical` |
| `DATA_DIR` | `./data` | Simulator persistence directory |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |
| `STALE_AFTER_MS` | `5000` | No data for this long → all positions marked offline |

---

## Architecture

```
            ┌──────────────────────┐
            │ Simulation adapter   │──┐
            └──────────────────────┘  │   exactly one adapter per process
            ┌──────────────────────┐  │   (simulated and physical data never mix)
            │ Physical adapter     │──┤
            │ (read-only scaffold) │  │
            └──────────────────────┘  ▼
                            ┌────────────────────┐   ┌─ REST  /api/*
     connection monitor ──▶ │  TelemetryStore    │──▶┤
                            │ (single source of  │   └─ Socket.IO "telemetry"
                            │  truth, normalised)│              │
                            └────────────────────┘              ▼
                                         Dashboard pages + 3D twin (same client feed)
```

```
server.js                      Express + Socket.IO wiring
src/
  config.js                    Equipment identity, position mapping, defaults (tagged CONFIRMED / ESTIMATE)
  logger.js                    Levelled logger (also feeds the Diagnostics page)
  persistence.js               Atomic JSON-file storage
  telemetry/schema.js          Canonical telemetry schema + status enum
  telemetry/store.js           TelemetryStore: the single source of truth
  telemetry/connection-monitor.js  Stale-data detection
  adapters/simulation-adapter.js   Simulation state machine and flow physics
  adapters/physical-adapter.js     Read-only hardware scaffold (no decoder registered)
  routes/api.js                REST API
public/
  index.html, css/style.css
  js/main.js                   Router + shared context
  js/telemetry-client.js       Socket.IO client with reconnect + REST fallback
  js/views/*.js                One module per page
  js/twin/*.js                 Three.js viewer, dispenser geometry, nozzle/hose, LCD/keypad, materials
docs/PHYSICAL_INTEGRATION.md   Hardware integration plan and open questions
```

Rendering, telemetry, simulation and hardware integration are kept separate. The 3D viewer never calls the backend itself: it receives telemetry and reports user actions through callbacks.

---

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/state` | Full telemetry snapshot |
| `GET` | `/api/equipment` | Identity, position mapping, adapter info |
| `GET` | `/api/transactions?position=A1&limit=100` | Completed sales |
| `GET` | `/api/diagnostics` | Uptime, link state, recent log |
| `POST` | `/api/positions/:id/:action` | `lift` `hang` `start` `pause` `resume` `stop` `reset` `fault` `clear-fault` `offline` `online` |
| `POST` | `/api/settings` | `{ "nominalFlowLpm": 45, "products": { "P1": { "name": "…", "unitPrice": 300 } } }` |

Start a sale with a preset:
```bash
curl -X POST localhost:3000/api/positions/A1/lift
curl -X POST localhost:3000/api/positions/A1/start \
     -H 'Content-Type: application/json' -d '{"preset":{"type":"volume","value":20}}'
```

**Legacy endpoints are kept** for older FuelOS clients:
- `POST /api/start/:id`
- `POST /api/stop/:id`
- `POST /api/reset-session/:id`
- `POST /api/settings` with `{ flowRateLpm, channel, price }`

**Socket.IO events:**
- `telemetry`: full snapshot, sent on every change (about 4 Hz while dispensing)
- `transaction`: each completed sale

In physical mode, every command endpoint returns **HTTP 403**.

---

## Equipment data: confirmed vs. estimated

| Item | Value | Source |
|---|---|---|
| Manufacturer / model / serial | Hongyang · CMD1687SK-GA44 · 21051034 | Nameplate ✅ |
| GA44 hardware | 2 products, 2 pumps, 4 flowmeters, 4 nozzles, 2 mainboards, 4 keyboards, 4 displays | [Hongyang A-Series table](https://www.chinahongyang.com/fuel-dispensers/a-series/) ✅ |
| Dimensions (≈ 1060 × 520 × 2520 mm) | Scaled from the catalogue photo | Estimate ⚠️ |
| Pump assignment | Pump 1 → A1 + B1, Pump 2 → A2 + B2 (one pump per product) | Estimate ⚠️ |
| Mainboard assignment | One per side (MB-A, MB-B) | Estimate ⚠️ |
| Internal layout and hose length | Schematic | Estimate ⚠️ |

The four nozzles are **not** four separate pump assemblies: two pumps serve four flowmeters, as the manufacturer's GA44 configuration specifies. Estimates are labelled as such throughout the UI.

---

## Physical integration (read-only)

No hardware connection is implemented, and the app makes no claim of one. The physical adapter:

- opens **no** port and generates **no** data unless a documented decoder is registered
- **rejects every control command:**
  - no start or stop, presets or price changes
  - no relay, valve, motor or pump control
  - no calibration writes
- accepts only **manufacturer-documented** protocols, with no reverse engineering

**Information still needed before a real connection:**
1. Hongyang's protocol document for this unit's controller and mainboards
2. The physical interface on the mainboards (RS-485 / RS-232 / current loop / Ethernet) and its pin-out
3. Whether a forecourt controller or POS is already connected, and that system's documented API
4. The pump addresses and nozzle numbering set on each mainboard
5. A confirmed product-to-nozzle and pump-to-meter mapping
6. Hazardous-area wiring requirements, plus installation by a qualified dispenser technician

Details: [`docs/PHYSICAL_INTEGRATION.md`](docs/PHYSICAL_INTEGRATION.md)

---

## Safety notice

This software is a **visualisation and simulation tool**. Actions in the 3D view and dashboard change the simulator only. Never connect it to live fuel-dispensing equipment except through manufacturer-documented, read-only interfaces, installed by qualified personnel with the site owner's authorisation.

## Acknowledgements

- [Three.js](https://threejs.org/) for rendering
- [Socket.IO](https://socket.io/) for real-time transport
- [Express](https://expressjs.com/) for the HTTP server
- Hongyang and the HONG YANG name and logo are the property of their owner. This project is an independent digital twin of an owned unit and is not affiliated with or endorsed by the manufacturer.

## License
ISC (see `package.json`).

