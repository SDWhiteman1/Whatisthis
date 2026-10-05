# Hand-off: Common Operating Picture (COP) prototype for Nova Scotia emergency management

## Who this is for

The user works in **provincial Emergency Preparedness/Management in Nova Scotia, Canada**.
They are scoping and prototyping a way to track **resources** (fleet vehicles, mobile command
units) and **personnel** (DNR staff, SAR teams) across agencies. Many personnel work in
**rugged terrain under heavy tree cover**, where cellular coverage is poor or absent.

This is a **low-budget proof of concept**, not a production system. Its job is to show
leadership a working multi-agency live map plus real field data, to justify funding a larger
pilot. Production would likely move to provincial infrastructure later (possibly Esri/ArcGIS
or TAK), so build with **standard formats and clean integration points**.

## Current state (context from the user)

- **DEM** has AVL on its fleet and sees vehicles live on a map, but it is a **standalone
  system**, not integrated with other agencies. There is no common operating picture.
- **DNR** does not track its vehicle fleet. Staff use field mapping tools heavily, but not for
  live personnel tracking.
- Resource locations are mostly managed through **radio communications, check-ins and
  incident mapping**.
- **The core gaps:**
  1. No shared, cross-agency view of resources
  2. Vehicles in rural NS lose cellular coverage
  3. Personnel in the backcountry and under canopy cannot be tracked live (the hardest problem)
- The user wants **live tracking everywhere** as the long-term goal, and wants to explore
  **mesh radio for personnel** first.

## Constraints

- **Budget: very small.** Phase 0 is free apart from the droplet. Phase 1 hardware is about CA$150–300.
  Do not design anything that needs Starlink, satellite plans or commercial trackers to demo.
- **Hosting:** one DigitalOcean droplet, ideally in the **Toronto (TOR1)** region for Canadian
  data residency. The smallest size (~1 GB RAM) should be enough; keep the footprint lean.
- **Free map stack:** Leaflet + OpenStreetMap. No paid API keys.
- **Time zone:** America/Halifax (Atlantic).
- **Radio band:** LoRa at **915 MHz** (902–928 MHz, licence-exempt in Canada).

## Phased plan

### Phase 0: software prototype (build this first)
- **Traccar** (open source) as the ingestion server and database
- **Custom COP dashboard** on top of Traccar's REST + WebSocket API
- **Phones as vehicle trackers** via the free **Traccar Client** app (OsmAnd protocol, port 5055),
  a no-cost stand-in for real AVL hardware that still shows real rural dead-zone behaviour
- **Simulator** that generates realistic multi-agency traffic so the dashboard can be demoed
  without any hardware, for example:
  - DEM vehicles and a mobile command unit on highways (Halifax, Truro, Antigonish, Sydney)
  - DNR trucks on rural and resource roads
  - SAR / DNR personnel moving slowly through a backcountry search area
  - Simulated **coverage dropouts** with delayed backfill uploads
- Real Teltonika FMM00A trackers can be added later on port 5027 (Teltonika protocol);
  Traccar supports them natively, so no extra work is needed now beyond documenting it

### Phase 1: minimal mesh test (cheap hardware)
- 3 × **Heltec WiFi LoRa 32 V3 (915 MHz)** running **Meshtastic** firmware
- Personnel radios pair with a **phone over Bluetooth**; the phone supplies GPS, even with no cell
  signal. The radios don't need built-in GPS.
- One radio acts as the **gateway**. It uses its own WiFi (e.g. a phone hotspot or vehicle WiFi)
  to publish over **MQTT** to a broker on the droplet.
- Build a **Meshtastic → Traccar bridge**: subscribe to the MQTT position messages, decode them,
  and forward them to Traccar (e.g. via the OsmAnd HTTP endpoint) so mesh personnel appear on
  the same map as vehicles.
- Field test goal: measure **range per hop, delivery rate, latency and battery life under
  Nova Scotia canopy**. Logging and simple stats for this are part of the deliverable.

### Later (out of scope now, design for it)
- Ingest DEM's existing AVL feed (vendor API or export, details unknown)
- Satellite devices (e.g. Garmin inReach) for personnel; Iridium fallback for vehicles
- Starlink plus a mesh gateway on mobile command units
- Tiered vehicle connectivity (Tier 1 command units, Tier 2 off-grid trucks with external
  antennas, Tier 3 general fleet)
- **TAK / Cursor-on-Target output** and **Esri feature service / GeoJSON output**, so partner
  agencies can consume the feed. A GeoJSON endpoint is a cheap win to include now.

## Dashboard requirements

- Map centred on Nova Scotia (roughly lat 43.4–47.1, lon −66.4 to −59.7)
- Markers styled by **agency** (DEM, DNR, SAR, other) and **type** (vehicle, command unit,
  person), with a legend and filters
- Live updates over WebSocket; heading and speed shown for vehicles
- **Staleness is first-class:** a "last seen X min ago" label, with markers fading or greying
  when stale (configurable thresholds), and an "offline > N min" indicator or alert
- **Backfilled track segments** drawn when a device reconnects and uploads stored positions
  (handle late and out-of-order timestamps correctly)
- Track history / playback for a selected time range
- Shows which **transport** each position came from (cellular, mesh, later satellite)
- Login required (live locations of staff are sensitive); HTTPS; mobile-friendly

## Security and governance

- HTTPS via a reverse proxy (Caddy preferred for automatic Let's Encrypt)
- Secrets live in `.env` (never committed); provide `.env.example`
- Firewall: allow only 22, 80, 443, 5055/TCP (OsmAnd), 5027/TCP (Teltonika, optional) and the
  MQTT port (use TLS and authentication on MQTT)
- Use a non-default Meshtastic channel name and key; note that Meshtastic encryption has not
  been reviewed by government security
- The README should note the governance work needed before any production use: a data-sharing
  agreement between DEM and DNR, a privacy impact assessment (NS FOIPOP), labour relations
  consultation, and the recommended **"track only while deployed or activated"** policy.
  Use only **simulated or volunteer test data** in the prototype; no real staff tracking.

## Deliverables

1. `docker-compose.yml`: Traccar, the dashboard, an MQTT broker (e.g. Mosquitto), the
   Meshtastic bridge and Caddy
2. Dashboard source code
3. Multi-agency simulator script (with dropouts and backfill)
4. Meshtastic → Traccar bridge service
5. `.env.example`
6. `README.md` covering:
   - Droplet setup (Docker, UFW, DNS/domain optional) and deployment
   - Setting up **Traccar Client** on a phone as a vehicle tracker
   - Flashing and configuring the **Heltec V3 / Meshtastic** nodes (web flasher, region
     `US` = 915 MHz, channel, gateway WiFi + MQTT settings, phone pairing for GPS)
   - Running the **canopy field test** and reading the results
   - Adding a real Teltonika FMM00A later (APN, server, port 5027, Codec 8E, IMEI registration)
   - Troubleshooting
7. Keep it simple enough for a non-developer to deploy by following the README

## Useful links

- Traccar server: https://www.traccar.org/ · Traccar Client app: https://www.traccar.org/client/
- Meshtastic: https://meshtastic.org/ · Web flasher: https://flasher.meshtastic.org/
- Meshtastic local groups: https://meshtastic.org/docs/community/local-groups/
- Nova Scotia Meshtastic community (Telegram): https://t.me/MeshtNovaScotia
- ISED National Broadband Data (coverage):
  https://ised-isde.canada.ca/site/high-speed-internet-canada/en/universal-broadband-fund/national-broadband-data-information

## Open questions (ask the user before or during the build)

- Is there a domain name for the droplet, or IP-only for now?
- Who is the DEM AVL vendor, and does it offer an API or export? (This affects the later integration.)
- Which field mapping tools does DNR use (e.g. ArcGIS Field Maps)? Is the province an Esri shop?
- Is the TMR2 radio system able to report GPS from personnel radios? (A potential no-hardware win
  to investigate outside this build.)
- Target position update interval and the "stale" thresholds for personnel and vehicles

## Status

- The repo contains only this file. No code yet.
- No hardware purchased. The user may get help or borrow nodes from the Nova Scotia Meshtastic
  community before buying.
- **Start with Phase 0.**
