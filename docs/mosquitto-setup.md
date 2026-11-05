# Mosquitto setup

`sudo mqgo install` installs mqgo **and** sets up Mosquitto in the same
run: it brings any Mosquitto 2.0 starting point to the dynamic security
plugin and connects mqgo to it. The procedure is always the same:

```sh
sudo ./mqgo install   # plan, questions; then URL, fingerprint, setup token
# in the browser: create the first admin
```

At the end the plugin is active, mqgo is connected, and existing clients
can do what they could before – or every difference was listed under
**IMPACT ON EXISTING CLIENTS** and confirmed. There is no "set it up by
hand" and no credentials to enter. Before it changes anything it backs up
every file involved, it rolls back on any error, and it can undo the
setup later. (Until 2026-11-01 this was the separate command
`mqgo mosquitto-setup`; it no longer exists.)

```sh
sudo mqgo install --dry-run           # the plan and its impact, change nothing (no root needed)
sudo mqgo install                     # interactive: plan, impact, questions, then the changes
sudo mqgo mosquitto-backups           # the backups taken so far
sudo mqgo mosquitto-restore           # undo the setup that ran last
```

Options of `install` for the Mosquitto part:

| Option | Meaning |
|---|---|
| `--dry-run` | Show the plan, the backup path and the impact; change nothing. |
| `--yes` | No questions. Needs `--anonymous=…` when anonymous clients are allowed, `--accept-impact` when the plan changes what existing clients can do, and `--install-mosquitto` when Mosquitto is missing. |
| `--accept-impact` | Accept the changes listed under IMPACT ON EXISTING CLIENTS without a question. |
| `--anonymous=keep\|drop` | What happens to anonymous clients (below). |
| `--install-mosquitto` | With `--yes`: install Mosquitto with apt-get if it is missing. |
| `--no-mosquitto` | Leave Mosquitto alone (broker on another host). |
| `--mosquitto-conf` | Default `/etc/mosquitto/mosquitto.conf`. |
| `--dynsec-file` | Default `/var/lib/mosquitto/dynamic-security.json` (for a new plugin configuration). |

`mqgo mosquitto-restore` takes `--backup <name>`, `--dry-run`, `--yes`,
`-config` and `--mosquitto-conf`.

The commands it runs are fixed, without a shell: `apt-get update`,
`apt-get install -y mosquitto`, `apt-get purge -y <packages>` (only the
packages this run installed, names from `dpkg-query`), `dpkg-query`,
`mosquitto -h` (version), `systemctl stop|restart|is-active mosquitto`,
`systemctl cat mqgo` and `systemctl try-restart mqgo`, `usermod -a -G` and
`gpasswd -d` (log access, below), `userdel`/`groupdel mosquitto` (only if
a purge left the user it created) and mqgo itself as the user `mqgo` to
store the broker credentials. mqgo's own service never runs as root, and
the admin UI has no button that would need root.

## Starting points

| Starting point | What install does |
|---|---|
| **No Mosquitto** | Debian/Ubuntu: step 1 of 2 – `apt-get update`, `apt-get install -y mosquitto` after its own question (`--yes` needs `--install-mosquitto`); then the plan for the package's configuration (step 2) with its own question. If step 2 is not confirmed or fails, the packages this run installed are purged again (the package's purge also removes the user `mosquitto`, its directories and logs). Other systems: a message with the package manager commands, nothing changed. |
| **No authentication** (Ubuntu's default: no listener, local-only, anonymous) | Anonymous question (below). Impact: clients that connected with any username (there was no password file) are rejected now – the plugin accepts a username only for its own clients. |
| **`password_file` / `acl_file`** | Import (below). |
| **Plugin already active** | If mqgo's stored account works: nothing to do. Otherwise Mosquitto is stopped (the plugin rewrites its file while it runs), backed up, and the client `mqgo-admin` with its own role (the rights of the admin role of `mosquitto_ctrl dynsec init`, hash format of Mosquitto 2.0) is added to the plugin's file; existing clients, roles, groups and the default access stay unchanged. If a client or role `mqgo-admin` exists, the account is `mqgo-admin-2` (`-3`, …). Then Mosquitto starts, the account is verified and stored. |
| **`per_listener_settings true`** | The plugin can be loaded only once (Mosquitto 2.0, `plugins/dynamic-security/plugin.c`), so with per-listener settings it would serve one listener. install switches to `per_listener_settings false` and merges the listeners (below); what cannot be reproduced is listed as impact. |
| **Several `password_file`/`acl_file` lines** | With `per_listener_settings true`, one per listener: all are merged (below). In one scope (no per-listener settings) Mosquitto 2.0 refuses to start (`src/conf.c`: "Duplicate password_file value in configuration"); only a reload uses the last line. install imports the last line and lists the others as impact. |

### Merging listeners (`per_listener_settings true`)

- **Clients:** the union of all password files. A user in several files
  with different passwords keeps the password of the first listener (in
  the order of the configuration); the other one no longer works (impact).
- **Rights:** the union of the user's rights on every listener whose
  password file has it (a listener without `acl_file` means every topic).
  Users that could connect only on some listeners can connect on all of
  them now (impact).
- **Anonymous clients:** one `allow_anonymous` for every listener; with
  *keep*, anonymous clients get the union of their previous rights on
  every listener – also on listeners that rejected them before (impact).
- **`psk_file`, `auto_id_prefix`, `allow_zero_length_clientid`, other
  plugins:** the first listener's value (plugins: the plugins of the first
  listener that has any) applies to every listener; the others are
  commented out (impact).
- Every changed line is commented out with the reason, not deleted.

Every entry of the impact block – restriction or widening – needs the
confirmation phrase.

## What it does

1. **Installs Mosquitto** if it is missing (above).
2. **Finds Mosquitto:** `mosquitto.conf` and its `include_dir`, the version
   (2.0 or later) and `mosquitto_dynamic_security.so`.
3. **Plugin active already:** adds mqgo's account (above), or does
   nothing when mqgo is connected – an update of a finished installation
   changes nothing.
4. **Takes stock:** `allow_anonymous`, `password_file`, `acl_file`,
   `per_listener_settings`, listeners, other plugins, `log_dest file`.
5. **Plans and shows the changes:**
   - `dynamic-security.json` (owner `mosquitto`, mode 0600) with the
     account `mqgo-admin` (random password, the rights of the admin role
     `mosquitto_ctrl dynsec init` creates) and the default access of
     `dynsec init`,
   - the clients and ACLs of `password_file`/`acl_file` (below),
   - `plugin` and `plugin_opt_config_file` in
     `<include_dir>/mqgo-dynsec.conf`, or at the end of `mosquitto.conf`
     without `include_dir`; `password_file` and `acl_file` are commented
     out,
   - read access to Mosquitto's log file for mqgo (below),
   - the path of the backup it will take, and
   - the block **IMPACT ON EXISTING CLIENTS** (below).
6. **Asks** (below), unless it is a dry run.
7. **Backs up** (below) – if that fails, it stops before any change.
8. **Writes, restarts Mosquitto and checks** the control API with the new
   account, then **stores** the credentials in mqgo's database (as the
   user `mqgo`) and restarts the mqgo service if it got a new group; the
   running service connects within seconds.
9. **On any failure after the first change** – including the credentials
   that cannot be stored – it restores every file from the backup byte for
   byte with owner and mode, removes what it created, takes back the group
   membership and mqgo's stored credentials, restarts Mosquitto, purges
   packages it installed in this run and reports the error. The admin
   password is never printed. mqgo itself stays installed; `install` can
   be run again.
10. **Reports** the result, repeats the accepted changes and the path of
    the backup, shows how to undo it, and prints mqgo's setup URL,
    fingerprint and token. mqgo's Broker page shows the same (when,
    backup, accepted changes, `sudo mqgo mosquitto-restore`).

## Backup (snapshot)

Each run takes a complete snapshot before it changes anything:
`/var/backups/mqgo/mosquitto-<UTC time>/` (owner root, mode 0700, also for
`/var/backups/mqgo`). It holds a copy of `mosquitto.conf`, every file of
`include_dir`, the password and ACL files, the dynamic security file, the
logrotate file and the files the setup will create (recorded as "did not
exist", so a restore removes them), and `manifest.json` with, per file, the
path, SHA-256, size, owner, group, mode and whether it existed, plus the
Mosquitto version, the time, mqgo's version and the group memberships it
changes. Every copy is read back and compared with its checksum. With less
than twice the needed space free, or any error, the setup stops and
nothing is changed.

mqgo never deletes a snapshot. Remove old ones by hand
(`sudo rm -r /var/backups/mqgo/mosquitto-<time>`) once you no longer need
them – but keep the oldest one as long as you might want to go back to the
original state. Snapshots hold Mosquitto's configuration including the
password hashes of its clients. `mqgo uninstall -purge` keeps them (they
are the way back for Mosquitto) and deletes only mqgo's broker credentials
in them; `-purge -purge-backups` removes `/var/backups/mqgo` as well.

## Impact on existing clients

The plan always ends with this block, in the dry run and in the real run:

```
IMPACT ON EXISTING CLIENTS
  every client, anonymous ones included: loses read and write access on devices/%c/state (acl_file pattern with %c: …)
  ...
```

or `none: existing clients keep what they could do before.` Entries are
every difference to the previous behaviour, in either direction:
`%c` patterns (the plugin of Mosquitto 2.0 does not substitute client IDs),
`$6$` passwords (clients disabled), ACL rules for users not in the password
file, rules that cannot be expressed, `--anonymous=drop`, usernames that
were accepted without a password file, other `plugin`/`auth_plugin` lines
(their combination with dynsec is not checked), file lines Mosquitto
ignored, and everything that changes when listeners are merged.

- **Interactive:** with entries, the setup continues only after the
  exact input `YES, APPLY`; anything else cancels with nothing changed in
  Mosquitto (packages installed by this run are purged again).
- **`--yes`:** with entries it also needs `--accept-impact`, otherwise
  it stops (exit code ≠ 0, nothing changed).
- **No terminal and no `--yes`:** it stops (exit code ≠ 0, nothing changed).
- **No entries:** the usual question (or `--yes`).

## Anonymous clients

With `allow_anonymous true` (also Mosquitto 2.0's default when no listener
is configured) it warns: with the plugin, anonymous clients only get the
default access (no publishing, no subscribing). It offers to keep their
previous rights:

- **keep:** a group `anonymous` becomes the plugin's anonymous group, with
  the role `anonymous-legacy`: the anonymous rules of the `acl_file` (its
  `topic` lines before the first `user` line, and patterns without `%u`/`%c`)
  with the `acl_file` semantics (below; the group also holds the role
  `imported-acl-baseline`, priority -1000), or full access without an
  `acl_file`. The role has priority -100 in the group, so a role an admin
  adds to the anonymous group later is checked first.
- **drop:** `allow_anonymous false`; listed as impact. Without any
  `listener` line Mosquitto 2.0 runs in local-only mode (port 1883 on the
  loopback addresses) and accepts anonymous clients regardless of
  `allow_anonymous`. The setup then adds `listener 1883 127.0.0.1` (and
  `listener 1883 ::1` if the host has IPv6 loopback), the addresses that
  were bound before – nothing is opened to the network.

With `--yes` or without a terminal, the choice must be given as
`--anonymous=keep|drop`.

## Import of password_file and acl_file

- Every user of the password file becomes a client. Passwords hashed with
  PBKDF2-SHA512 (`$7$`, Mosquitto 2.0's default) are taken over: the
  clients keep their passwords. Older SHA-512 hashes (`$6$`, Mosquitto 1.6
  and earlier) cannot be verified by the plugin: these clients get a random
  password and are **disabled** (impact); their description in mqgo says
  so. Set a new password and enable them on their client page.
- **Without `acl_file`** every authenticated client could do everything
  (except the plugin's own `$CONTROL` topics after the setup). Every
  imported client gets the role `imported-full-access`, which reproduces
  that: publish, subscribe and receive on `#` and `$SYS/#`.
- **With `acl_file`** the rules of each user, plus the `pattern` rules with
  `%u` replaced by the username, become the role `client-role-<user>`,
  translated with the semantics of Mosquitto 2.0's `acl_file`: there a
  subscription is always accepted and the `read` rules filter what is
  delivered. The role grants `subscribePattern` and `publishClientReceive`
  for the `read` topics, `publishClientSend` for `write`, and denies for
  `deny` lines (higher priority). The rest – every subscription accepted,
  nothing else delivered, `$SYS` and `$CONTROL` not delivered unless a rule
  names them – is the role `imported-acl-baseline`, assigned through the
  group `imported-acl-baseline` (membership priority -1000).
- **Priorities:** imported roles are assigned with priority -100, below the
  -1 the admin UI presets, and the baseline group comes last. The plugin
  checks a client's own roles before its groups, by priority. So roles and
  groups an admin assigns later with the preset priority are checked
  first: their allows and denies work for imported clients as for new
  ones. Removing the baseline group or its role gives the imported clients
  additional rights (the Role and Group pages say so).
- Patterns with `%c` cannot be carried over (the plugin of Mosquitto 2.0
  does not substitute them); rules for users that are not in the password
  file are not imported. Both are listed as impact.

## Mosquitto log access

The Live Log page shows Mosquitto's log file (`mosquitto_log`). Ubuntu's
and Debian's package write it with mode 0600 (`mosquitto:mosquitto`), and
`/etc/logrotate.d/mosquitto` uses `nocreate`, so after each rotation
Mosquitto creates the new file with 0600 again. If the configuration has
`log_dest file <path>` and the user `mqgo` exists, the setup therefore
plans:

1. `usermod -a -G <group of the file> mqgo`,
2. `chmod 0640 <path>` (group read, nothing for others),
3. in `/etc/logrotate.d/mosquitto`: `nocreate` → `create 0640 mosquitto
   <group>`, so that the rotated file stays readable,
4. `systemctl try-restart mqgo`, so that the service gets the new group.

All of this is in the plan, the snapshot and the restore. Verified on
Mosquitto 2.0.18 with `logrotate -f` (test 14.11); without step 3 the new
file is 0600 again.

## Restore

```sh
sudo mqgo mosquitto-backups
sudo mqgo mosquitto-restore --dry-run              # what would happen
sudo mqgo mosquitto-restore                        # undo the setup that ran last
sudo mqgo mosquitto-restore --backup mosquitto-20261101T143642Z --yes
```

Without `--backup` the restore takes the **newest** `before-setup`
snapshot: the state before the setup that ran last. It does so only while
a setup is active (the dynamic security plugin is in the configuration);
otherwise it stops with an error that points to `mqgo mosquitto-backups`, and
nothing changes. `mosquitto-backups` marks that snapshot `[default]` and the
oldest `before-setup` snapshot `[original]`; the original state – and any
other snapshot – is restored only with `--backup <name>`.

The restore

1. verifies every checksum of the chosen snapshot and refuses a damaged one
   (nothing changed),
2. shows what it will write or remove and warns that everything changed
   since then in the plugin (clients, groups, roles, ACLs) is lost,
3. asks; a "no", or no terminal without `--yes`, ends with exit code ≠ 0
   and nothing changed,
4. backs up the current state first as its own snapshot (kind
   `before-restore`, including mqgo's broker credentials), so the restore
   itself can be undone with `mosquitto-restore --backup <that snapshot>`,
5. writes the files byte for byte with owner and mode, removes the files
   that did not exist, restores the group memberships,
6. restarts Mosquitto and checks that it runs,
7. deletes the credentials of the account the setup created from mqgo's database (or, when
   restoring a `before-restore` snapshot, puts back the credentials of
   that time; if `uninstall -purge` deleted them, mqgo shows "not
   configured" until an account is entered on the Broker page) and
   restarts mqgo if its groups changed.

If Mosquitto does not start after the restore, it prints the error and the
next steps (`journalctl -u mosquitto`, the snapshot of the state before the
restore) and deletes nothing. ACL comments in mqgo's database are kept;
they may show as orphaned afterwards.

### Restore by hand

Without mqgo (e.g. after it was uninstalled), as root:

```sh
cd /var/backups/mqgo/mosquitto-<time>
cat manifest.json            # path, sha256, owner, group, mode, existed – per file
sha256sum 0*                 # compare with the manifest
```

For each entry of `files` in `manifest.json`:

- `"existed": true` – copy the file named in `copy` back to `path`
  (`install -o <owner> -g <group> -m <mode> <copy> <path>`); for the log
  file (`"content_restored": false`) only set owner and mode,
- `"existed": false` – delete `path` (e.g. `/etc/mosquitto/conf.d/mqgo-dynsec.conf`
  and `/var/lib/mosquitto/dynamic-security.json`).

For each entry of `groups` with `"member": false`: `gpasswd -d <user>
<group>`. Then `systemctl restart mosquitto`, `systemctl restart mqgo`, and
on mqgo's Broker page remove the stored broker account.

## Restoring after an installed Mosquitto

`mosquitto-restore` restores configuration files; it does not remove a
Mosquitto that install installed. Restoring the before-setup snapshot of
such a run gives the package's default configuration back.
