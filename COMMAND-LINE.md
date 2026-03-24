# Glorytun Command-Line Reference

Glorytun uses the [argz](https://github.com/angt/argz) library for declarative,
table-driven command-line parsing. Arguments are keyword-based (not prefixed with
`--`). The special keyword `help` can be appended after any command or nested
context to display its usage line.

```
glorytun <command> [options...]
```

## Commands

Running `glorytun` with no arguments (or an unknown command) prints this list.

| Command   | Description                    |
|-----------|--------------------------------|
| `list`    | List tunnel control socket paths |
| `show`    | Show tunnel info or errors     |
| `bench`   | Run a crypto benchmark         |
| `bind`    | Start a new tunnel             |
| `set`     | Change tunnel timing options   |
| `keygen`  | Generate a new secret key      |
| `path`    | Show or configure paths        |
| `version` | Show version and libsodium     |

---

## `glorytun bind`

Start a new tunnel. Creates a tunnel device, establishes the encrypted UDP
transport, and enters the main packet-forwarding loop.

```
glorytun bind keyfile FILE [dev NAME]
              [from addr IPADDR port PORT]
              [to addr IPADDR port PORT]
              [chacha] [persist]
```

| Option | Type | Description |
|--------|------|-------------|
| `keyfile FILE` | string | **Required.** Path to the secret key file (hex). |
| `dev NAME` | string | Tunnel interface name to create or use. |
| `from addr … port …` | nested | Local UDP bind address and port (IPv4 or IPv6). |
| `to addr … port …` | nested | Remote peer address and port (client / outbound). |
| `chacha` | flag | Force ChaCha20-Poly1305 instead of AEGIS-256 when available. |
| `persist` | flag | Keep the tunnel device after the process exits. |

Both `from` and `to` use a nested form: the keyword `from` or `to`, then
`addr` and an IP address, then `port` and a port number. Omit `from` to use the
defaults (IPv4 `0.0.0.0`, UDP port `5000`). Omit `to` for server mode (listen
only).

**Defaults**

- Local bind defaults to `0.0.0.0:5000` if `from` is omitted.
- If AEGIS-256 is not available on the platform, ChaCha20-Poly1305 is used
  automatically unless `chacha` forces the fallback.
- On `SIGHUP`, persist mode is enabled on the device before exit (graceful
  reload).

**Examples**

Server — listen on all interfaces, port 5000:

```sh
glorytun bind keyfile /etc/glorytun/key dev tun0 persist
```

Server — listen on a specific address:

```sh
glorytun bind keyfile /etc/glorytun/key dev tun0 persist \
  from addr 203.0.113.1 port 5000
```

Client — connect to a remote peer:

```sh
glorytun bind keyfile /etc/glorytun/key dev tun0 persist \
  to addr 203.0.113.1 port 5000
```

---

## `glorytun show`

Show information about a running tunnel.

```
glorytun show [dev NAME] [errors]
```

| Option     | Type | Description |
|------------|------|-------------|
| `dev NAME` | string | Tunnel device. If omitted and only one tunnel exists, it may be auto-selected (see `glorytun list`). |
| `errors`   | flag | Show per-error counters instead of status. |

**Without `errors`**, prints: tunnel name, local and remote address.port, PID,
MTU, and cipher (`aegis256` or `chacha20poly1305`).

**With `errors`**, prints counters for `decrypt`, `clocksync`, and `keyx`, each
with last source address and port when present.

---

## `glorytun set`

Change timing-related properties of a running tunnel.

```
glorytun set [dev NAME] [kxtimeout TIME] [timetolerance TIME] [keepalive TIME]
```

| Option | Type | Description |
|--------|------|-------------|
| `dev NAME` | string | Tunnel device. |
| `kxtimeout TIME` | time | Key exchange (rotation) timeout. |
| `timetolerance TIME` | time | Clock sync tolerance between peers. |
| `keepalive TIME` | time | Keepalive interval. |

All time values accept [time suffixes](#time-suffixes). Values are applied
together in one control message; omitted options keep their previous values on
the daemon side as implemented.

---

## `glorytun path`

Manage paths on a running tunnel (multipath). You must select a tunnel with
`dev`, optionally narrow the path with `addr` (local IP) and `to` (remote), then
either **show** status or **set** configuration.

**View path status (default)**

```
glorytun path [dev NAME] [addr IP] [to addr IP port PORT]
              [show [mtu|rtt|stat]]
```

With no `set` keyword, the daemon returns path status. Optional `show` picks a
view: `mtu` (MTU probe columns), `rtt` (RTT / RTT variance), `stat` (TX/RX rates
and loss). Without `show`, a default wide status table is printed.

**Change path settings**

```
glorytun path [dev NAME] [addr IP] [to addr IP port PORT] set
              [up|down]
              [rate [fixed|auto] [tx RATE] [rx RATE]]
              [beat TIME] [pref N] [losslimit PERCENT]
```

| Option | Type | Description |
|--------|------|-------------|
| `dev NAME` | string | Tunnel device (**required** for path operations). |
| `addr IP` | IPv4/IPv6 | Select path by local source address. |
| `to addr … port …` | nested | Select path by remote address (optional filter). |
| `set` | context | Introduces path configuration keywords below. |
| `up` / `down` | choice | Enable or disable the path (mutually exclusive). |
| `rate` | nested | `fixed` or `auto` (mutually exclusive), plus `tx` / `rx` caps ([size suffixes](#size-suffixes)). |
| `beat TIME` | time | Internal heartbeat / probe interval. |
| `pref N` | integer | Path preference (0–127); higher values bias scheduling. |
| `losslimit PERCENT` | percent | Loss threshold to treat the path as too lossy (0–100; optional `%`). |

The `path set` form is required to change state or rates. There is **no**
separate `backup` keyword on the CLI; the daemon may still use internal backup
semantics. To favour some paths over others, use different `pref` and `tx` /
`rx` values.

**Examples**

```sh
glorytun path dev tun0
glorytun path dev tun0 addr 192.168.1.100
glorytun path dev tun0 show rtt
glorytun path dev tun0 addr 192.168.1.100 set up beat 5s losslimit 30% \
  rate fixed tx 12500k rx 12500k
glorytun path dev tun0 addr 192.168.1.100 set down
```

With **`rate auto`**, `tx` and `rx` are ceilings: the implementation probes up to
those limits.

---

## `glorytun bench`

Run a cryptographic throughput benchmark (min / mean / max Mbps over packet
sizes).

```
glorytun bench [fallback]
```

| Option | Type | Description |
|--------|------|-------------|
| `fallback` | flag | Benchmark ChaCha20-Poly1305 instead of AEGIS-256. |

If `fallback` is omitted and AEGIS-256 is available, it is benchmarked. If
AEGIS-256 is not available, ChaCha20-Poly1305 is used.

---

## `glorytun keygen`

Print a new random 256-bit secret key as 64 hex characters. No options.

```
glorytun keygen
```

---

## `glorytun list`

Print paths to control sockets for running tunnels (one line per socket), for
use with `show` / `set` / `path`.

```
glorytun list
```

---

## `glorytun version`

Print the program version and linked libsodium version.

```
glorytun version
```

---

## Value Types

### Time suffixes

Options that take time support:

| Suffix | Meaning |
|--------|---------|
| `ms` | milliseconds |
| *(none)*, `s` | seconds (value scaled to milliseconds internally) |
| `m` | minutes |
| `h` | hours |
| `d` | days |
| `w` | weeks |

### Size suffixes

Rate and size options accept SI/binary prefixes and `bit` / `byte` units; see
the argz helpers in the build. Examples: `100Mbit`, `12500k`, `1GiB`.

### Percent suffix

`losslimit` accepts an optional `%` suffix (e.g. `30%`) or a raw 0–100 value.

### Pipe-separated choices

Where this document lists `a|b` (e.g. `fixed|auto`, `up|down`), only one of the
group may be set; duplicates produce an error.

---

## Help

```sh
glorytun bind help
glorytun path help
glorytun set help
glorytun show help
glorytun bench help
glorytun list help
glorytun version help
glorytun keygen help
```

---

## Option parsing rules

- **Named options** (`dev`, `keyfile`, `addr`, …) are keywords followed by
  values where required.
- **Flags** (`persist`, `chacha`, `fallback`, `errors`) need no value.
- **Nested groups** (`from` / `to` with `addr` and `port`; `path` … `set` with
  `rate`, `up`, …) consume following tokens until the group is complete.
- **Order** of top-level keywords is generally flexible; use `help` on each
  command to see the exact usage string the binary expects.
