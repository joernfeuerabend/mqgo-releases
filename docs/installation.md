# Installation

mqgo is one static binary for Linux (amd64, arm64). It manages a Mosquitto
2.0 or later broker through the dynamic security plugin. One command
installs mqgo, sets up Mosquitto and connects the two:

```sh
sudo ./mqgo install   # plan, questions; then URL, fingerprint, setup token
# in the browser: create the first admin
```

The procedure is the same whether Mosquitto is missing, unconfigured, uses
`password_file`/`acl_file`, already runs the dynamic security plugin, or
uses `per_listener_settings`. At the end the plugin is active, mqgo is
connected to it, and existing clients can do what they could before – or
every difference was listed in the plan and confirmed.

## Install as a service

On the broker host (Ubuntu 24.04 or any Linux with systemd):

```sh
sudo ./mqgo install --dry-run   # show everything, change nothing
sudo ./mqgo install
```

`install` prints its plan first and then:

- copies the binary to `/usr/local/bin/mqgo`,
- creates the system user and group `mqgo`,
- writes `/etc/mqgo/mqgo.conf` from the example (only if it does not exist
  yet; an existing config is never overwritten) with `data_dir =
  "/var/lib/mqgo"` and log files in `/var/log/mqgo`,
- writes a hardened systemd unit (`/etc/systemd/system/mqgo.service`): the
  service runs as `mqgo`, never as root, may write only its data, config
  and log directories, and may bind ports below 1024 (443 for ACME),
- enables and starts the service,
- **sets up Mosquitto** with its own plan and questions (below and
  [Mosquitto setup](mosquitto-setup.md)): installs it if it is missing
  (Debian/Ubuntu, `apt-get`), sets up the dynamic security plugin or adds
  an admin account for mqgo to an active one, backs up every file first and
  rolls back on any error, stores mqgo's broker account in its database,
- prints the admin URLs, the certificate fingerprint and the setup token.

Options for the Mosquitto part:

| Option | Meaning |
|---|---|
| `--dry-run` | Show the whole plan, including the Mosquitto plan and its impact; change nothing (no root needed). |
| `--yes` | No questions. Needs `--anonymous=…` when anonymous clients are allowed, `--accept-impact` when the plan changes what existing clients can do, and `--install-mosquitto` when Mosquitto is missing. |
| `--no-mosquitto` | Do not touch Mosquitto: the broker runs on another host. Enter its address under **Settings → Broker** and its dynsec admin account on the **Broker** page. |
| `--mosquitto-conf`, `--dynsec-file` | Other paths than `/etc/mosquitto/mosquitto.conf` and `/var/lib/mosquitto/dynamic-security.json`. |

Without a terminal and without `--yes`, install changes nothing in
Mosquitto (unless the setup is complete) and ends with exit code ≠ 0;
mqgo itself is installed then. If the Mosquitto part fails or is not
confirmed, everything it changed is undone and install says so; mqgo stays
installed, and running `install` again retries.

Running `install` again with a newer binary updates mqgo; configuration
and data are kept. On a finished setup – the plugin active and mqgo
connected – it changes nothing in Mosquitto. When mqgo's `mqtt_host` is
not a loopback address (a broker on another host), install skips the
Mosquitto part as well.

`sudo mqgo uninstall` removes binary and unit and keeps configuration and
data; `-purge` removes them too (`-yes` skips the question). `-root DIR`
prefixes every path and skips the user, systemd and Mosquitto steps, to
look at the generated files without root.

## First start: setup wizard

The service starts in setup mode while no admin account exists. `install`
prints, at its end:

- the admin UI's URLs (all addresses of the host, port 8443),
- the SHA-256 fingerprint of its self-signed certificate,
- a one-time setup token, valid for an hour.

`sudo mqgo setup-info` prints them again (and a new token once the old one
has expired). Open the setup URL, check that the browser shows the printed
fingerprint, and create the first admin account. You are signed in on
**My account**, which offers two-factor authentication (optional, see
[Administration](administration.md#my-account-two-factor-authentication)).

The admin UI listens on all interfaces. To keep it private, set
`webgui_listen_address = "127.0.0.1"` (Settings → Network & listeners) and
use an SSH tunnel: `ssh -N -L 8443:localhost:8443 user@broker-host`. Or
list the addresses allowed to connect in `webgui_allowed_clients` (same
section, see [Client address list](configuration.md#client-address-list));
loopback, and with it the SSH tunnel, is always allowed.

## Connection to the broker

`install` stores the broker account in mqgo's database; the running
service connects within seconds. For a broker on another host
(`--no-mosquitto`), enter the address under **Settings → Broker** (default
`localhost:1883`) and the dynsec admin account (the one `mosquitto_ctrl
dynsec init` created) on the **Broker** page.

The **Broker** page and the Dashboard show the connection state:
connected, not connected (mqgo retries every 3 s), authentication failed,
or *dynsec not active* (the broker answers, the plugin does not – run
`sudo mqgo install` on the broker host).

## Domain & certificate

By default (`tls_mode = "selfsigned"`) the admin UI uses its self-signed
certificate on port 8443, and browsers warn. With a domain that points to
the host and port 443 reachable from the internet, mqgo can get a Let's
Encrypt certificate itself:

1. Create an A record for the domain; forward external TCP port 443 to the
   host if it is behind NAT.
2. **Settings → TLS & domain**: set `tls_mode` to `acme`, enter the domain
   (and optionally a contact e-mail), select **Apply**.
3. mqgo checks DNS and port 443, places a trial order at Let's Encrypt
   staging, then the real order, and moves the admin UI to port 443 – no
   restart. The old listener closes 30 s later; sign in again at
   `https://<domain>/` (sessions are bound to the address).

Nothing is written to the config file unless the certificate was obtained.
Renewal is automatic. Other names (IP address, `localhost`) keep getting
the self-signed certificate.

To run mqgo behind Cloudflare Tunnel, Tailscale or a reverse proxy that
terminates TLS, see [Running behind a tunnel](tunnel.md).
