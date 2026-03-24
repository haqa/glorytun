# Glorytun Client — OpenRC

Service scripts for running a Glorytun multipath client tunnel under OpenRC.

## Files

| File | Install to | Description |
|------|-----------|-------------|
| `glorytun-client` | `/etc/init.d/glorytun-client` | Init script |
| `glorytun-client.conf` | `/etc/conf.d/glorytun-client` | Configuration file |

## Installation

```sh
cp glorytun-client /etc/init.d/glorytun-client
chmod 755 /etc/init.d/glorytun-client

cp glorytun-client.conf /etc/conf.d/glorytun-client
```

## Configuration

Edit `/etc/conf.d/glorytun-client`. The main settings are:

### Tunnel

| Variable | Default | Description |
|----------|---------|-------------|
| `TUN_DEV` | `tun0` | TUN device name |
| `SERVER_ADDR` | — | Remote server IP address |
| `SERVER_PORT` | — | Remote server UDP port |
| `KEYFILE` | — | Path to the shared key file |
| `TUN_LOCAL_IP` | — | Local tunnel IP in CIDR notation (e.g. `10.99.0.2/30`). Leave empty to configure addressing elsewhere. |

### Interfaces

`INTERFACES` is a space-separated list of `INTERFACE:GATEWAY` pairs. Each
interface becomes a separate path through the tunnel. The gateway is the
next-hop for that interface — needed because the default route may point
elsewhere and source-based policy routing must know where to send each
interface's traffic.

```sh
INTERFACES="eth1:192.168.1.254 eth2:10.0.0.1 eth3:172.16.0.1 eth4:10.1.0.1"
```

### Path settings

| Variable | Default | Description |
|----------|---------|-------------|
| `RATE_MODE` | `fixed` | Rate mode: `fixed` or `auto` |
| `RATE_TX` | `12500k` (`fixed`) / large cap (`auto`) | Transmit limit per path (bytes/sec with suffix). With `auto`, this is a **ceiling** for probing, not a fixed bitrate. If unset under `auto`, a large default cap is used (see `RATE_AUTO_*_MAX`). |
| `RATE_RX` | (same as `RATE_TX`) | Receive limit advertised to the peer; same semantics as `RATE_TX`. |
| `RATE_AUTO_TX_MAX` | `12500000k` | When `RATE_MODE=auto` and `RATE_TX` is unset/empty, use this ceiling. |
| `RATE_AUTO_RX_MAX` | `12500000k` | When `RATE_MODE=auto` and `RATE_RX` is unset/empty, use this ceiling. |
| `BEAT` | `5s` | Keepalive beat interval |
| `LOSSLIMIT` | `30%` | Loss percentage threshold for path degradation |
| `PATH_STATE` | `up` | Initial state: `up` or `backup` |

The init script always passes explicit `tx`/`rx` to `glorytun path`. Commenting out `RATE_TX`/`RATE_RX` in the config does **not** omit them — `${VAR:-default}` still applies. Under `fixed`, the default stays ~100 Mbit; under `auto`, the default ceiling is much higher so variable links (e.g. LTE) are not capped at ~100 Mbit unless you set a lower `RATE_TX`/`RATE_RX`.

### Routing

| Variable | Default | Description |
|----------|---------|-------------|
| `RT_TABLE_START` | `101` | First routing table number (incremented per interface) |
| `RP_FILTER` | `2` | Reverse path filter mode (2 = loose, recommended) |

### Glorytun binary

| Variable | Default | Description |
|----------|---------|-------------|
| `GLORYTUN_BIN` | `/usr/local/bin/glorytun` | Path to glorytun |
| `GLORYTUN_OPTS` | `persist` | Extra options for `glorytun bind` |

## Key generation

```sh
mkdir -p /etc/glorytun
glorytun keygen > /etc/glorytun/key
chmod 600 /etc/glorytun/key
```

Copy the same key to the server via a secure channel.

## Usage

```sh
# Start the tunnel, policy routes, and all paths
rc-service glorytun-client start

# Stop everything and clean up routes
rc-service glorytun-client stop

# Show tunnel and path status
rc-service glorytun-client status

# Enable at boot
rc-update add glorytun-client default

# Disable at boot
rc-update del glorytun-client default
```

## What the init script does

On **start**, in order:

1. Starts `glorytun bind to <server> <port> ...` in the background.
2. Waits for the TUN device to appear (up to 5 seconds).
3. Assigns `TUN_LOCAL_IP` to the TUN device and brings it up.
4. Sets `rp_filter` to loose mode on all interfaces.
5. For each interface in `INTERFACES`:
   - Resolves the interface's current IPv4 address.
   - Creates a source-based policy route through the specified gateway.
   - Registers the path with glorytun (state, rate, beat, loss limit).

On **stop**, in reverse:

1. Brings each path down via `glorytun path ... down`.
2. Removes the source-based policy routes and flushes routing tables.
3. Stops the glorytun process.

## Notes

- Interface IP addresses are resolved at startup. If an interface has no IPv4
  address (e.g. DHCP hasn't completed), that path is skipped with a warning.
- The `status` command shows both tunnel status (`glorytun show`) and path
  status (`glorytun path`).
- For DHCP environments where addresses change, consider a DHCP hook script
  that brings the old path down and registers the new address (see
  `MULTIPATH-EXAMPLE.md`).
