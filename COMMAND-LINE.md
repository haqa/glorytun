# Glorytun Command-Line Reference

Glorytun uses the [argz](https://github.com/angt/argz) library for declarative,
table-driven command-line parsing. Arguments are positional keywords (not prefixed
with `--`). The special keyword `help` can be appended after any command or
subcommand to display its usage line.

```
glorytun <command> [options...]
```

## Commands

Running `glorytun` with no arguments (or an unknown command) prints this list.

| Command   | Description                    |
|-----------|--------------------------------|
| `show`    | Show tunnel info               |
| `bench`   | Start a crypto bench           |
| `bind`    | Start a new tunnel             |
| `set`     | Change tunnel properties       |
| `keygen`  | Generate a new secret key      |
| `path`    | Manage paths                   |
| `version` | Show version                   |

---

## `glorytun bind`

Start a new tunnel. Creates a tunnel device, establishes the encrypted UDP
transport, and enters the main packet-forwarding loop.

```
glorytun bind [IPADDR] [PORT] [to [IPADDR] [PORT]] [dev NAME]
              keyfile FILE [chacha] [persist]
```

| Option          | Type     | Description                                                  |
|-----------------|----------|--------------------------------------------------------------|
| `IPADDR`        | positional | Local IP address to bind (IPv4 or IPv6). Default: `0.0.0.0` |
| `PORT`          | positional | Local port to bind. Default: `5000`                          |
| `to IPADDR`     | positional | Remote peer IP address (within the `to` sub-option)          |
| `to PORT`       | positional | Remote peer port (within the `to` sub-option). Default: same as bind port |
| `dev NAME`      | string   | Name of the tunnel device to create/use                      |
| `keyfile FILE`  | string   | **Required.** Path to the secret key file                    |
| `chacha`        | flag     | Force ChaCha20-Poly1305 cipher instead of AEGIS-256          |
| `persist`       | flag     | Keep the tunnel device alive after the process exits         |

The `IPADDR` and `PORT` values are **positional** — they are given as bare
values without a preceding keyword. The local bind address/port appear directly
after `bind`. The remote peer address/port follow the `to` keyword.

**Defaults:**
- Bind address defaults to `0.0.0.0` (all interfaces), port `5000`.
- If `to` is omitted, the tunnel runs in server mode (no outbound peer).
- If AEGIS-256 is not available on the platform, ChaCha20-Poly1305 is used
  automatically regardless of the `chacha` flag.
- On `SIGHUP`, the tunnel device is set to persist before exiting (for graceful
  reload).

**Examples:**

Server (listen on all interfaces, port 5000):

```sh
glorytun bind keyfile /etc/glorytun/key persist
```

Server (listen on a specific address and port):

```sh
glorytun bind 203.0.113.1 5000 keyfile /etc/glorytun/key persist
```

Client (bind locally, connect to remote peer):

```sh
glorytun bind 192.168.1.100 5000 to 203.0.113.1 5000 dev tun0 keyfile /etc/glorytun/key persist
```

---

## `glorytun show`

Show information about a running tunnel.

```
glorytun show [dev NAME] [bad]
```

| Option     | Type   | Description                                              |
|------------|--------|----------------------------------------------------------|
| `dev NAME` | string | Tunnel device name. Auto-detected if only one tunnel exists |
| `bad`      | flag   | Show error counters instead of general status            |

**Without `bad`**, displays: tunnel name, server/client mode, PID, bind
address and port, peer address and port (client only), MTU, and cipher.

**With `bad`**, displays per-error-type counters: `decrypt`, `difftime`,
and `keyx`, along with the last source address and port for each.

Output is formatted as human-readable when stdout is a terminal, or as a
single-line machine-readable format otherwise.

---

## `glorytun set`

Change properties of a running tunnel.

```
glorytun set [dev NAME] [tc CS|AF|EF] [kxtimeout SECONDS]
             [timetolerance SECONDS] [keepalive SECONDS]
```

| Option                   | Type   | Description                                    |
|--------------------------|--------|------------------------------------------------|
| `dev NAME`               | string | Tunnel device name                             |
| `tc CS\|AF\|EF`         | DSCP   | Traffic class / DSCP marking for tunnel packets |
| `kxtimeout SECONDS`     | time   | Key exchange (rotation) timeout                |
| `timetolerance SECONDS` | time   | Clock synchronisation tolerance between peers  |
| `keepalive SECONDS`     | time   | Keepalive interval                             |

### Traffic class values

The `tc` option accepts DSCP class names:

| Format   | Examples                      | Description               |
|----------|-------------------------------|---------------------------|
| `CS0`–`CS7` | `CS0`, `CS6`              | Class Selector            |
| `AF`*xy* | `AF11`, `AF21`, `AF43`        | Assured Forwarding (class 1–4, drop 1–3) |
| `EF`     | `EF`                          | Expedited Forwarding      |

All time values accept [time suffixes](#time-suffixes). The default unit (no
suffix) is seconds.

---

## `glorytun path`

Manage network paths for a running tunnel. A tunnel can have multiple paths
(multipath) between local and remote addresses.

When only a filter is given (or no arguments), path status is displayed.
When state or configuration options are given, the specified path is modified.

```
glorytun path [IPADDR] [dev NAME] [up|backup|down]
              [rate [fixed|auto] [tx BYTES/SEC] [rx BYTES/SEC]]
              [beat SECONDS] [losslimit PERCENT]
```

| Option              | Type       | Description                                           |
|---------------------|------------|-------------------------------------------------------|
| `IPADDR`            | positional | Select/identify path by its local IP address          |
| `dev NAME`          | string     | Select tunnel device                                  |
| `up\|backup\|down`  | choice     | Set path state (mutually exclusive, one of the three) |
| `rate`              | sub        | Rate limiting sub-options (see below)                 |
| `beat SECONDS`      | time       | Internal heartbeat/probe interval. Accepts [time suffixes](#time-suffixes) |
| `losslimit PERCENT` | percent    | Loss percentage threshold to disable the path (0–100). Accepts `%` suffix |

The `IPADDR` value is **positional** — given as a bare IP address without a
preceding keyword. It identifies which path to operate on by its local address.

### Path states

| State    | Description                                               |
|----------|-----------------------------------------------------------|
| `up`     | Path is active and carries traffic                        |
| `backup` | Path is kept alive but only used if primary paths degrade |
| `down`   | Path is disabled                                          |

### `path rate`

```
glorytun path IPADDR dev NAME rate [fixed|auto] [tx BYTES/SEC] [rx BYTES/SEC]
```

| Option        | Type   | Description                                                   |
|---------------|--------|---------------------------------------------------------------|
| `fixed\|auto` | choice | Fixed rate or dynamic rate detection (mutually exclusive)     |
| `tx BYTES/SEC` | bytes | Maximum transmission rate. Accepts [size suffixes](#size-suffixes) |
| `rx BYTES/SEC` | bytes | Maximum reception rate. Accepts [size suffixes](#size-suffixes)    |

With **`rate auto`**, `tx` / `rx` are still maximum (ceiling) values: the
implementation probes and adjusts the path rate up to those limits. If a
service script always passes a modest ceiling (e.g. ~100 Mbit), throughput
will not grow beyond that even in auto mode.

### Displaying path status

When no state-changing options are given, `path` displays status for all paths
(or for the path matching `IPADDR` if specified). Output includes: state,
bind/public/peer addresses and ports, MTU, RTT, rate mode, loss limit, beat
interval, and TX/RX statistics.

**Examples:**

Show all paths:

```sh
glorytun path dev tun0
```

Show a specific path:

```sh
glorytun path 192.168.1.100 dev tun0
```

Bring a path up:

```sh
glorytun path 192.168.1.100 dev tun0 up beat 5s losslimit 30%
```

Set fixed rate limits on a path:

```sh
glorytun path 192.168.1.100 dev tun0 rate fixed tx 100Mbit rx 100Mbit
```

Disable a path:

```sh
glorytun path 192.168.1.100 dev tun0 down
```

---

## `glorytun bench`

Run a cryptographic performance benchmark.

```
glorytun bench [aes|chacha]
```

| Option        | Type   | Description                                       |
|---------------|--------|---------------------------------------------------|
| `aes\|chacha` | choice | Select cipher to benchmark (mutually exclusive)   |

If neither is specified and AEGIS-256 is available, it is benchmarked by
default. If AEGIS-256 is not available, ChaCha20-Poly1305 is benchmarked
automatically. Specifying `aes` on a platform without AEGIS-256 support
produces an error.

Outputs min/mean/max throughput in Mbps for various packet sizes (20–1450 bytes).

---

## `glorytun keygen`

Generate a new random 256-bit secret key and print it as a hexadecimal string.
Takes no options.

```
glorytun keygen
```

The output (64 hex characters) can be saved to a file for use with `bind keyfile`.

---

## `glorytun version`

Print the Glorytun version, or the linked libsodium version.

```
glorytun version [libsodium]
```

| Option      | Type | Description                            |
|-------------|------|----------------------------------------|
| `libsodium` | flag | Print the libsodium version instead    |

---

## Value Types

### Positional arguments

Some options have no keyword name — their values are given directly as bare
arguments. In the argz table these have `name = NULL` and are identified by
their type (e.g. `IPADDR`, `PORT`). If a bare value does not match the
expected type, argz reports an error like:

```
error: `badvalue' is not a valid IPADDR for bind
```

### Time suffixes

Options that accept time values (marked `SECONDS` in usage) support the
following suffixes:

| Suffix | Unit         | Equivalent          |
|--------|--------------|---------------------|
| `ms`   | milliseconds | 1 ms                |
| `s`    | seconds      | 1,000 ms            |
| `m`    | minutes      | 60,000 ms           |
| `h`    | hours        | 3,600,000 ms        |
| `d`    | days         | 86,400,000 ms       |
| `w`    | weeks        | 604,800,000 ms      |

If no suffix is given, the value is interpreted as **seconds** (multiplied by
1000 internally to milliseconds).

### Size suffixes

Options that accept size values (marked `BYTES/SEC` in usage) support SI and
binary prefixes combined with byte/bit units:

**Prefixes:**

| Prefix     | SI (×1000)      | Binary (×1024)          |
|------------|-----------------|-------------------------|
| `k` or `K` | 1,000           | 1,024 (with `i`)       |
| `m` or `M` | 1,000,000       | 1,048,576 (with `i`)   |
| `g` or `G` | 1,000,000,000   | 1,073,741,824 (with `i`) |

**Units:**

| Unit                        | Meaning                        |
|-----------------------------|--------------------------------|
| `B`, `byte`, `bytes`        | bytes                          |
| `b`, `bit`, `bits`          | bits (value divided by 8)      |
| *(no unit after prefix)*    | bytes                          |

**Examples:** `100Mbit` = 100 megabits, `1GiB` = 1 gibibyte, `500k` = 500,000 bytes.

### Percent suffix

The `losslimit` option accepts a `%` suffix (e.g. `50%`). The suffix is
optional; the raw integer (0–100) is also accepted.

### Pipe-separated choices

Options written as `a|b|c` in this documentation (e.g. `up|backup|down`,
`aes|chacha`, `fixed|auto`) accept exactly one of the listed keywords.
Specifying more than one produces an error.

---

## Help

The keyword `help` can be used after any command to display its usage line:

```sh
glorytun bind help
glorytun path help
glorytun set help
glorytun bench help
glorytun version help
```

---

## Option Parsing Rules

Glorytun uses the argz library which follows these conventions:

- **Named options** (e.g. `dev NAME`, `keyfile FILE`) match a keyword followed
  by a value.
- **Positional arguments** (e.g. bare `IPADDR`, `PORT`) are matched by type
  against the next argument without any keyword prefix.
- **Flag options** (e.g. `chacha`, `persist`, `bad`) are set by their mere
  presence with no following value.
- **Choice options** (e.g. `up|backup|down`) accept exactly one of the listed
  alternatives. Setting two from the same choice produces an "already set" error.
- **Nested sub-options** (e.g. `to`, `rate`) introduce a nested option context
  that consumes subsequent arguments.
- **Unknown options** cause an error and usage display.
- **Order is flexible** — named options can appear in any order, but positional
  arguments are consumed in declaration order.
