# Glorytun Server — OpenRC

Service scripts for running Glorytun server tunnels under OpenRC.

## Files

| File | Install to | Description |
|------|-----------|-------------|
| `glorytun-server` | `/etc/init.d/glorytun-server` | Init script |
| `glorytun-server.conf` | `/etc/conf.d/glorytun-server` | Configuration file |

## Installation

```sh
cp glorytun-server /etc/init.d/glorytun-server
chmod 755 /etc/init.d/glorytun-server

cp glorytun-server.conf /etc/conf.d/glorytun-server
```

## Configuration

Edit `/etc/conf.d/glorytun-server`. The key setting is `TUNNELS`, which lists
all tunnels to manage. Each entry has the format:

```
TUN_NAME:PORT:INTERFACE:KEYFILE[:TUN_LOCAL_IP]
```

| Field         | Description                                                          |
|---------------|----------------------------------------------------------------------|
| `TUN_NAME`    | Name of the TUN device to create (e.g. `tun0`)                      |
| `PORT`        | UDP port to listen on                                                |
| `INTERFACE`   | Network interface to bind to (IP is resolved at startup)             |
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

```sh
# Start all configured tunnels
rc-service glorytun-server start

# Stop all tunnels
rc-service glorytun-server stop

# Check status of all tunnels
rc-service glorytun-server status

# Enable at boot
rc-update add glorytun-server default

# Disable at boot
rc-update del glorytun-server default
```

## Notes

- The init script manages all tunnels defined in `TUNNELS` as a single service.
  Starting or stopping the service affects all tunnels.
- Each tunnel runs as a separate background process with a PID file in
  `/run/glorytun/`.
- The IP address is resolved from the named interface at startup. If the
  interface has no IPv4 address, that tunnel fails to start with an error.
- The `persist` option (default in `GLORYTUN_OPTS`) keeps the TUN device alive
  if glorytun exits, allowing a restart without disrupting the interface
  configuration.
- The init script waits for the TUN device, optionally assigns `TUN_LOCAL_IP`
  if specified in the tunnel spec, and brings the interface up. If you omit
  `TUN_LOCAL_IP`, configure the TUN interface via `/etc/network/interfaces`
  or a post-up script.

**Troubleshooting:** If `glorytun show` reports "no device", the control socket
is per-user. The service runs as root, so run `sudo glorytun show` (or
`sudo glorytun show dev tun0`) to query the tunnel status.
