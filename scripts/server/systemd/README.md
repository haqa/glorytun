# Glorytun Server — systemd

Service files for running Glorytun server tunnels under systemd.

## Files

| File | Install to | Description |
|------|-----------|-------------|
| `glorytun-server@.service` | `/etc/systemd/system/glorytun-server@.service` | Template service unit |
| `glorytun-server-start` | `/usr/local/lib/glorytun/glorytun-server-start` | Helper script called by the service |
| `glorytun-server` | `/etc/default/glorytun-server` | Configuration file |

## Installation

```sh
cp glorytun-server@.service /etc/systemd/system/
systemctl daemon-reload

mkdir -p /usr/local/lib/glorytun
cp glorytun-server-start /usr/local/lib/glorytun/
chmod 755 /usr/local/lib/glorytun/glorytun-server-start

cp glorytun-server /etc/default/glorytun-server
```

## Configuration

Edit `/etc/default/glorytun-server`. The key setting is `TUNNELS`, which lists
all tunnels to manage. Each entry has the format:

```
TUN_NAME:PORT:INTERFACE:KEYFILE[:TUN_LOCAL_IP]
```

| Field         | Description                                                          |
|---------------|----------------------------------------------------------------------|
| `TUN_NAME`    | Name of the TUN device to create (e.g. `tun0`)                      |
| `PORT`        | UDP port to listen on                                                |
| `INTERFACE`   | Network interface to bind to (IP is resolved at startup)            |
| `KEYFILE`     | Path to the secret key file for this tunnel                          |
| `TUN_LOCAL_IP`| Optional. CIDR address for the tunnel (e.g. `10.99.0.1/30`). The script waits for the TUN device, assigns this IP, and brings the interface up. Omit to configure elsewhere. |

Example with two tunnels on different interfaces, each with its own key and IP:

```sh
TUNNELS="tun0:5000:eth0:/etc/glorytun/tun0.key:10.99.0.1/30 tun1:5000:eth1:/etc/glorytun/tun1.key:10.99.1.1/30"
```

Example with two tunnels on the same interface using different ports and a
shared key:

```sh
TUNNELS="tun0:5000:eth0:/etc/glorytun/key:10.99.0.1/30 tun1:5001:eth0:/etc/glorytun/key:10.99.1.1/30"
```

Generate a key file with:

```sh
mkdir -p /etc/glorytun
glorytun keygen > /etc/glorytun/key
chmod 600 /etc/glorytun/key
```

Optional settings in the config file:

| Variable       | Default                        | Description                        |
|----------------|--------------------------------|------------------------------------|
| `GLORYTUN_BIN` | `/usr/local/bin/glorytun`      | Path to the glorytun binary        |
| `GLORYTUN_OPTS`| `persist`                      | Extra options passed to `glorytun bind` |

## Usage

This is a template unit. The instance name (after the `@`) is the tunnel name,
which must match an entry in the `TUNNELS` variable.

```sh
# Enable and start a tunnel
systemctl enable --now glorytun-server@tun0

# Enable and start multiple tunnels
systemctl enable --now glorytun-server@tun0 glorytun-server@tun1

# Stop a tunnel
systemctl stop glorytun-server@tun0

# Check status
systemctl status glorytun-server@tun0

# View logs
journalctl -u glorytun-server@tun0

# Follow logs in real time
journalctl -fu glorytun-server@tun0

# Disable a tunnel (won't start at boot)
systemctl disable glorytun-server@tun0
```

## Notes

- Each tunnel runs as an independent systemd service. You can start, stop,
  restart, and inspect logs for each tunnel individually.
- The service automatically restarts on failure after a 5-second delay.
- The helper script reads `/etc/default/glorytun-server`, finds the matching
  tunnel entry, resolves the IP from the named interface, and execs glorytun.
- The IP address is resolved from the named interface at startup. If the
  interface has no IPv4 address, the service fails with an error visible in
  `journalctl`.
- The `persist` option (default in `GLORYTUN_OPTS`) keeps the TUN device alive
  if glorytun exits, allowing a restart without disrupting the interface
  configuration.
- The unit includes systemd hardening (read-only filesystem, restricted
  capabilities). If you encounter permission issues, check the
  `ProtectSystem`, `ReadWritePaths`, and `CapabilityBoundingSet` settings in
  the unit file.
- The helper script waits for the TUN device, optionally assigns `TUN_LOCAL_IP`
  if specified in the tunnel spec, and brings the interface up. If you omit
  `TUN_LOCAL_IP`, configure the TUN interface via systemd-networkd or elsewhere.

**Troubleshooting:** If `glorytun show` reports "no device", glorytun uses
different control socket paths depending on whether `/run/user/0` exists. The
service hides `/run/user` so it uses `/run/glorytun.0`. If your root session has
`/run/user/0` (e.g. from logind), run glorytun in the same sandbox:

```sh
systemd-run --scope -p InaccessiblePaths=/run/user -G glorytun show
```
