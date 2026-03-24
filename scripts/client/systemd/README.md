# Glorytun Client — systemd

Service files for running a Glorytun multipath client tunnel under systemd.

## Files

| File | Install to | Description |
|------|-----------|-------------|
| `glorytun-client.service` | `/etc/systemd/system/glorytun-client.service` | Service unit |
| `glorytun-client-start` | `/usr/local/lib/glorytun/glorytun-client-start` | Startup script (tunnel + routing + paths) |
| `glorytun-client-teardown` | `/usr/local/lib/glorytun/glorytun-client-teardown` | Teardown script (routes + paths cleanup) |
| `glorytun-client` | `/etc/default/glorytun-client` | Configuration file |

## Installation

```sh
cp glorytun-client.service /etc/systemd/system/
systemctl daemon-reload

mkdir -p /usr/local/lib/glorytun
cp glorytun-client-start /usr/local/lib/glorytun/
cp glorytun-client-teardown /usr/local/lib/glorytun/
chmod 755 /usr/local/lib/glorytun/glorytun-client-start
chmod 755 /usr/local/lib/glorytun/glorytun-client-teardown

cp glorytun-client /etc/default/glorytun-client
```

## Configuration

Edit `/etc/default/glorytun-client`. The main settings are:

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
| `RATE_TX` | `12500k` (`fixed`) / large cap (`auto`) | Transmit limit per path (bytes/sec with suffix). With `auto`, this is a **ceiling** for probing. Unset under `auto` uses a large script default or `RATE_AUTO_TX_MAX`. |
| `RATE_RX` | (same pattern) | Receive limit; same semantics as `RATE_TX`. |
| `RATE_AUTO_TX_MAX` | `12500000k` | Ceiling when `RATE_MODE=auto` and `RATE_TX` is unset/empty. |
| `RATE_AUTO_RX_MAX` | `12500000k` | Ceiling when `RATE_MODE=auto` and `RATE_RX` is unset/empty. |
| `BEAT` | `5s` | Keepalive beat interval |
| `LOSSLIMIT` | `30%` | Loss percentage threshold for path degradation |
| `PATH_STATE` | `up` | Passed to `glorytun path … set` (`up` or `down`) |

Commenting out `RATE_TX`/`RATE_RX` does not remove limits: the start script uses `${VAR:-default}`. Use `auto` with unset rates (or raise `RATE_TX`/`RATE_RX`) so mobile links are not stuck at ~100 Mbit.

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
# Enable and start the tunnel
systemctl enable --now glorytun-client

# Stop the tunnel (teardown runs automatically)
systemctl stop glorytun-client

# Check status
systemctl status glorytun-client

# View logs
journalctl -u glorytun-client

# Follow logs in real time
journalctl -fu glorytun-client

# Disable at boot
systemctl disable glorytun-client
```

## What the scripts do

### glorytun-client-start (ExecStart)

1. Sources `/etc/default/glorytun-client`.
2. Starts `glorytun bind to addr <server> port <port> ...` in the background.
3. Waits for the TUN device to appear (up to 5 seconds).
4. Assigns `TUN_LOCAL_IP` to the TUN device and brings it up.
5. Sets `rp_filter` to loose mode on all interfaces.
6. For each interface in `INTERFACES`:
   - Resolves the interface's current IPv4 address.
   - Creates a source-based policy route via the specified gateway.
   - Registers the path with glorytun (state, rate, beat, loss limit).
7. Waits on the glorytun process (so systemd tracks the main PID).

### glorytun-client-teardown (ExecStopPost)

Runs after the service stops (including on failure/restart):

1. Brings each path down via `glorytun path dev … addr … set down`.
2. Removes source-based policy rules and flushes routing tables.

This ensures policy routes don't linger if the service crashes.

## Notes

- Interface IP addresses are resolved at startup. If an interface has no IPv4
  address (e.g. DHCP hasn't completed), that path is skipped with a warning
  visible in `journalctl`.
- The service automatically restarts on failure after a 5-second delay. The
  teardown script runs before each restart to clean up stale routes.
- The unit includes systemd hardening (read-only filesystem, restricted
  capabilities). If you encounter permission issues, check the
  `ProtectSystem`, `ReadWritePaths`, and `CapabilityBoundingSet` settings in
  the unit file.
- For DHCP environments where addresses change, consider a DHCP hook script
  that brings the old path down and registers the new address (see
  `MULTIPATH-EXAMPLE.md`).
- Unlike the server scripts (which use a template unit per tunnel), the client
  is a single service managing one tunnel with multiple paths.
