# Obico for COSMOS (Centauri Carbon)

Fork of [moonraker-obico](https://github.com/TheSpaghettiDetective/moonraker-obico) focused on running Obico as a **LAN Docker companion** for [OpenCentauri COSMOS](https://docs.opencentauri.cc/) on the Elegoo Centauri Carbon.

The stock COSMOS board does not have enough CPU/RAM to run Obico’s Janus/ffmpeg stack locally. This setup keeps Moonraker and the webcam on the printer, and runs moonraker-obico on a separate Docker host on the same LAN.

```text
  Docker host (companion)              Centauri Carbon (COSMOS)
  ┌─────────────────────┐              ┌──────────────────────────┐
  │ moonraker-obico     │── Moonraker ─│ :80  Moonraker + Mainsail│
  │ Janus / ffmpeg      │── webcam ────│ /webcam/?action=stream   │
  └─────────┬───────────┘              └──────────────────────────┘
            │
            └── Obico server (cloud or self-hosted on LAN)
```

[Obico](https://www.obico.io) is an open-source smart 3D printing platform (failure detection, remote monitoring, etc.).

## Requirements

- OpenCentauri **COSMOS** on the Centauri Carbon (stock Elegoo firmware has no Moonraker)
- A Docker host on the **same LAN** as the printer
- Printer reachable from that host on port **80** (Moonraker + Mainsail + webcam)
- Obico account (app.obico.io or your own server)

Quick connectivity check from the Docker host:

```bash
curl -sS http://PRINTER_IP/server/info
curl -sI http://PRINTER_IP/webcam/?action=snapshot
```

## 1. Get the companion files

```bash
git clone https://github.com/mon5termatt/obico-cosmos.git
cd obico-cosmos

mkdir -p config logs
cp config/moonraker-obico.cfg.companion.sample config/moonraker-obico.cfg
```

Or copy just `docker-compose.yml` and `config/moonraker-obico.cfg.companion.sample` into a directory on the Docker host and work from there.

**Permissions:** the container runs as uid `1000`. Own the config/logs dirs accordingly or linking will fail to write `auth_token`:

```bash
sudo chown -R 1000:1000 config logs
```

## 2. Edit the config

Edit `config/moonraker-obico.cfg`:

1. Replace every `PRINTER_IP` with the printer’s LAN IP.
2. Set `[server] url`:
   - Obico Cloud: `https://app.obico.io`
   - Self-hosted on LAN: prefer the **LAN** URL (e.g. `http://10.2.0.45:3334`). Public HTTPS in front of Cloudflare often breaks Obico WebSockets even when the REST API works.
3. Do **not** set `auth_token` yet — the link step writes it.

COSMOS defaults in the sample:

| Setting | COSMOS value | Typical stock Klipper |
| --- | --- | --- |
| Moonraker port | `80` | `7125` |
| Webcam | `http://PRINTER_IP/webcam/?action=...` | `:8080/?action=...` |
| Tunnel dest | `PRINTER_IP:80` | `127.0.0.1:80` |

If Moonraker rejects the companion, either add the Docker host IP to Moonraker trusted clients, or set `[moonraker] api_key`.

## 3. Install Obico macros on the printer

The container cannot write to the printer. Copy [`include_cfgs/moonraker_obico_macros.cfg`](include_cfgs/moonraker_obico_macros.cfg) into the COSMOS Klipper config directory and include it:

```ini
[include moonraker_obico_macros.cfg]
```

Then reload/restart Klipper as usual.

## 4. Link to Obico

In the Obico app/web UI, add a **Klipper** printer and get a 6-digit code. On the Docker host:

```bash
docker run --rm -it \
  -v "$(pwd)/config/moonraker-obico.cfg:/opt/printer_data/config/moonraker-obico.cfg" \
  --entrypoint /opt/venv/bin/python \
  ghcr.io/thespaghettidetective/moonraker-obico:latest \
    -m moonraker_obico.link -c /opt/printer_data/config/moonraker-obico.cfg
```

Confirm `auth_token` was written into `config/moonraker-obico.cfg`. If linking loops after Obico says the printer was added, fix ownership on that file (see step 1) and use a **new** code.

## 5. Start the companion

```bash
docker compose up -d
docker compose logs -f moonraker-obico
```

You want Moonraker connected, printer linked, and a state update (e.g. `Offline -> Standby` / `Printing`). Check the Obico app for live status and webcam.

To build from this repo instead of the published image, uncomment `build: .` in `docker-compose.yml` and run `docker compose up -d --build`.

## Troubleshooting

| Symptom | Likely fix |
| --- | --- |
| Link asks for code forever; Obico already shows the printer | Config owned by root — `chown 1000:1000` the cfg, new link code |
| `api key is unset` looping / no Moonraker connect | Wrong port (use `80` on current COSMOS), or set `api_key` / trusted clients |
| HTTP to Obico works, but `Connections not ready` / `WS Closed` | WebSocket path broken (often Cloudflare). Point `[server] url` at the LAN Obico origin |
| Status OK, no video | Fix `[webcam]` URLs so they work **from the Docker host** |
| `OBICO_LINK_STATUS not configured as a macro` | Install macros on the printer (step 3) |
| Warnings for `machine/update/status` or `device_power` | Harmless — COSMOS does not expose those Moonraker components |
| `ensure_include_cfgs.sh exited with 1` | Expected in companion mode (no local `printer.cfg`) |

## Upstream / other installs

This fork is COSMOS-companion oriented. For installing moonraker-obico **on** a normal Pi/Klipper host, see upstream:

- [TheSpaghettiDetective/moonraker-obico](https://github.com/TheSpaghettiDetective/moonraker-obico)
- [Obico Klipper docs](https://www.obico.io/docs/user-guides/klipper-setup/)

Extra container notes: [run_as_container.md](run_as_container.md).
