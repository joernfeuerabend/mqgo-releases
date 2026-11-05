# Running behind a tunnel

mqgo can run behind a tunnel or reverse proxy that terminates TLS
(Cloudflare Tunnel, Tailscale Serve/Funnel, nginx, …). Then mqgo itself
serves plain HTTP, and the tunnel provides HTTPS to the browser.

## Settings

```toml
tls_mode = "off"
webgui_listen_address = "127.0.0.1"   # the tunnel connects locally
webgui_port = 8443
public_url = "https://mq.example.com" # the address the browser uses
```

- **`tls_mode = "off"`** serves plain HTTP. On a loopback address
  (`127.0.0.1`, `::1`) that is all. On any other address mqgo refuses to
  start unless **`allow_insecure_http = true`** is set as well – passwords
  and sessions would then travel in clear text to anyone between the
  browser and the host, so only do that if the tunnel's connector runs on
  another machine of a network you trust.
- **`public_url`** is required behind a tunnel. The browser sends the
  tunnel's HTTPS address as `Origin` with every form, and mqgo refuses
  forms whose origin is not its own (protection against cross-site
  requests). `public_url` adds that one origin; the check itself stays.
- mqgo warns about TLS off in the startup log and on the Admin access →
  Settings and Security pages, and about a missing `public_url`.

## Loopback-only view

With `webgui_listen_address` on loopback (`127.0.0.0/8`, `::1` or
`localhost`), as behind a tunnel or with an SSH tunnel
(`ssh -L 8443:127.0.0.1:8443 host`), the admin UI can only be reached
from the host itself. Admin access → Security and Activity then show the
note "mqgo listens on loopback only (…). Access is only possible from this
host, e.g. through an SSH tunnel. The protections below stay active." and
fold their cards to one summary line each; Admin access → Settings folds
TLS & domain, Brute-force protection, Connections & limits and the HTTP
server group. A card or section with content (a locked address, a throttled
account, dropped connections, a rate-limited address, an error or a value
other than the default) stays open, warnings stay visible, and every
protection works unchanged. With any other listen address everything is
shown open, as before.

## Sessions

The session cookie is always `Secure`. Browsers send it over HTTPS and on
`http://localhost`, so behind an HTTPS tunnel and locally everything works
– but plain HTTP to a LAN address (`http://192.168.1.10:8443`) does not
allow a login.

## Brute-force protection

mqgo does not evaluate `X-Forwarded-For` (a header anyone can set). Behind
a tunnel every request therefore comes from the tunnel's address, and the
lockout per address would lock out everyone at once after a few failed
admin logins. If that is a problem:

```toml
bruteforce_client_lockout = false
```

The per-account throttle stays active: after many failed logins for one
username, only one password check per interval runs for it. mqgo never
turns the lockout off by itself; while it is off, the Security page and
the startup log say so. Two-factor authentication for every admin is the
better protection in this setup.

## Client address list

`webgui_allowed_clients` (see
[Client address list](configuration.md#client-address-list)) does not help
behind a tunnel. It sees the address of the TCP connection, which behind a
tunnel is the tunnel's, not the browser's:

- A tunnel connector on this host (`cloudflared`, `tailscale serve`, nginx
  on the same machine) connects from `127.0.0.1` or `::1`. Loopback is
  always allowed, so the list has no effect on anything that comes through
  the tunnel. That is deliberate: loopback stays open so that an SSH
  tunnel can never be locked out.
- A connector on another host connects from that host's address. Listing
  it allows everyone who comes through the tunnel; leaving it out allows
  no one.

Restrict access in the tunnel or proxy instead (Cloudflare Access,
Tailscale ACLs, the proxy's own allow rules). While the list is set and
`public_url` or `allow_insecure_http` is set, the Settings page (next to
the field) and the Security page show this warning.
