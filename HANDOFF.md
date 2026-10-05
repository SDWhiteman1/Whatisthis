# Hand-off: Real-time vehicle tracking dashboard (Nova Scotia)

## Goal

Build a self-hosted system that shows **3 Teltonika FMM00A GPS trackers** moving in
real time on a **map of Nova Scotia, Canada**, hosted on a **DigitalOcean droplet**.

## Hardware

- **Device:** Teltonika FMM00A, a plug-and-play OBD-II tracker (North American model,
  Canadian ISED certified).
- **Connectivity:** LTE Cat M1 / NB-IoT. Each unit needs an LTE-M/IoT data SIM
  (Bell, Telus or Rogers in Canada). Expected data use is about 5–20 MB/month per device.
- **Protocol:** Teltonika Codec 8 / Codec 8 Extended over TCP (binary AVL packets;
  the device sends an IMEI handshake first, and the server must ACK the record count).
- **Features to surface:** GNSS position (< 3 m), speed, heading, ignition,
  OEM odometer and fuel level read via OBD, and the internal battery / external voltage.
- **Offline buffering:** the device stores records (128 MB) when out of coverage and
  uploads them later, so the backend must handle **late, out-of-order timestamps**.
- **Configuration:** Teltonika Configurator (USB) or the Teltonika mobile app (Bluetooth):
  APN, server host/port, TCP, Codec 8E, and the reporting intervals.

## Recommended architecture

```
[FMM00A x3] --LTE-M--> internet --> [Droplet]
                                      ├─ Traccar (receiver + DB + REST/WebSocket API)
                                      ├─ Custom web dashboard (Leaflet + OpenStreetMap)
                                      └─ Reverse proxy (Caddy or Nginx) + Let's Encrypt HTTPS
```

- **Traccar** (open source) handles Teltonika decoding natively on **port 5027 (TCP)**.
  Use it as the ingestion layer rather than writing a raw Codec 8 decoder.
- **Custom dashboard** built on Traccar's REST + WebSocket API:
  - Map centred on and bounded to Nova Scotia (roughly lat 43.4–47.1, lon −66.4 to −59.7)
  - Live markers that update in real time, labelled per vehicle, showing heading and speed
  - Vehicle list sidebar with online/offline status and last-seen time
  - Trip history / route playback for a chosen date range
  - Mobile-friendly
- Everything runs via **Docker Compose** so it deploys with a few commands.

## Requirements and constraints

- Smallest droplet size (~1 GB RAM) should be enough for 3 devices; keep the footprint lean.
- **Security:** the dashboard must require a login (live locations are sensitive), use HTTPS,
  and keep secrets in `.env` (never committed). The firewall should allow only 22, 80, 443
  and 5027/TCP.
- Use a free map stack (Leaflet + OSM tiles); no paid API keys.
- Use Atlantic time (America/Halifax) in the UI.
- Privacy: if any vehicles are driven by employees, they must be notified
  (Nova Scotia / PIPEDA). Add a note in the README; no code impact.

## Deliverables

1. `docker-compose.yml`: Traccar, the dashboard and a reverse proxy with automatic HTTPS
2. The dashboard source code (frontend; a small backend/proxy only if needed)
3. `.env.example` with all required variables
4. `README.md` with:
   - Droplet setup (Docker install, firewall/UFW, domain DNS)
   - Deployment steps
   - **Device configuration guide** for the FMM00A (APN, server, port 5027, TCP,
     Codec 8E, suggested intervals: ~10–30 s moving, ~5–15 min parked)
   - Registering each device in Traccar by its **IMEI**
   - Troubleshooting (device not connecting, no GPS fix, port blocked)
5. A way to test without hardware, e.g. a script that simulates 3 vehicles driving
   around Nova Scotia (Halifax, Truro, Sydney) through Traccar's OsmAnd protocol
   on port 5055, so the dashboard can be verified before the trackers arrive

## Open questions to confirm with the user

- Traccar's built-in UI alone, or Traccar plus a custom dashboard? (Recommended: custom dashboard.)
- Is there a domain name for the droplet, or IP-only for now?
- Which carrier/SIM provider? (Needed for the APN in the config guide.)
- Personal or business use? Any alerts wanted (geofences, speeding, ignition on/off)?
- Single shared login, or multiple users?

## Status

- No code exists yet; the repository is empty.
- Hardware has not been purchased yet (sourcing from a Canadian distributor).
