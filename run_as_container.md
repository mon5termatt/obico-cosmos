# Use the container Image

## COSMOS / remote companion (recommended for Centauri Carbon)

Run moonraker-obico on a LAN Docker host so the printer's limited CPU does not
have to run Janus/ffmpeg. Moonraker and the webcam stay on the printer
(OpenCentauri COSMOS); Obico connects over the network.

### 1. Configure

On the Docker host, create a companion directory and copy these files from this
repo (or clone the repo and work from its root):

- `docker-compose.yml`
- `config/moonraker-obico.cfg.companion.sample`

```bash
mkdir -p moonraker-obico-companion/config moonraker-obico-companion/logs
cd moonraker-obico-companion

# From wherever you have this repo checked out:
cp /path/to/moonraker-obico/docker-compose.yml .
cp /path/to/moonraker-obico/config/moonraker-obico.cfg.companion.sample config/
cp config/moonraker-obico.cfg.companion.sample config/moonraker-obico.cfg
```

Edit `config/moonraker-obico.cfg` and replace every `PRINTER_IP` with your
printer's LAN address. Current COSMOS serves Moonraker on port `80` and the
webcam at `/webcam/?action=...` (not `:7125` / `:8080`). Do **not** set
`auth_token` yet. For a self-hosted Obico server on the same LAN, prefer the
LAN URL (e.g. `http://10.x.x.x:3334`) if Cloudflare breaks WebSockets.

If Moonraker requires an API key or only trusts certain clients, either:

- Add your Docker host's LAN IP to Moonraker's trusted clients, or
- Set `[moonraker] api_key` in the companion config

### 2. Install Obico macros on the printer

The container cannot write to the printer's filesystem. On the COSMOS printer,
copy [`include_cfgs/moonraker_obico_macros.cfg`](include_cfgs/moonraker_obico_macros.cfg)
into the Klipper config directory and add an include, for example:

```ini
[include moonraker_obico_macros.cfg]
```

Then reload Klipper / restart as you normally would after a config change.

### 3. Link your printer to Obico

Open the Obico web UI or app and obtain a 6-digit verification code for a
`Klipper`-type printer. From the companion directory on the Docker host:

```bash
docker run --rm -it \
  -v "$(pwd)/config/moonraker-obico.cfg:/opt/printer_data/config/moonraker-obico.cfg" \
  --entrypoint /opt/venv/bin/python \
  ghcr.io/thespaghettidetective/moonraker-obico:latest \
    -m moonraker_obico.link -c /opt/printer_data/config/moonraker-obico.cfg
```

Confirm `config/moonraker-obico.cfg` now contains `[server] auth_token`.

### 4. Start the companion

```bash
docker compose up -d
```

### 5. Verify

```bash
docker compose logs -f moonraker-obico
```

You should see a successful Moonraker connection. In the Obico app the printer
should show as online, and the webcam preview should update. If status works but
video does not, fix the `[webcam]` URLs so they are reachable **from the Docker
host** (not `127.0.0.1` on the printer).

To rebuild from this repo instead of the published image, uncomment `build: .`
in `docker-compose.yml` and run `docker compose up -d --build`.

---

## Link your printer (manual path)

1. Create a copy of `moonraker-obico.cfg.sample` in a directory of your choice.

```bash
cp moonraker-obico.cfg.sample /opt/mydir/moonraker-obico.cfg
```

2. Set the `[moonraker].host` and `[moonraker].port` in the newly created config file, according to the [Documentation](https://www.obico.io/docs/user-guides/moonraker-obico/config/), making sure to **not** set the `[server].auth_token`.

3. Open the Obico Webinterface or App to obtain the *6-digit verification code* for a `Klipper`-Type Printer.

4. Replace `/opt/mydir/moonraker-obico.cfg` in the following command with the path to your configuration file and run it. Enter the Code obtained in the previous step, when promted.

```bash
docker run --rm -it \
  -v /opt/mydir/moonraker-obico.cfg:/opt/printer_data/config/moonraker-obico.cfg \
  --entrypoint /opt/venv/bin/python \
  ghcr.io/thespaghettidetective/moonraker-obico:latest \
    -m moonraker_obico.link -c /opt/printer_data/config/moonraker-obico.cfg
```

5. Check that your configuration file now contains a value for `[server].auth_token`

## Run the application

Given that your moonraker-obico.cfg now contains a valid `[server].auth_token`, a container may be started using the following command:

```bash
docker run -d \
  --name moonraker-obico \
  --privileged \
  -v /opt/mydir/moonraker-obico.cfg:/opt/printer_data/config/moonraker-obico.cfg \
  ghcr.io/thespaghettidetective/moonraker-obico:latest
```
