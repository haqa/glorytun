# Multipath Tunnel Example

This example sets up a Glorytun multipath tunnel between two machines:

- **Machine A** — a site device with four network interfaces (`eth1`–`eth4`),
  each receiving a DHCP address.
- **Machine B** — a cloud concentrator with a single public IP address.

All four WAN links on Machine A are bonded into one tunnel, providing
aggregated bandwidth and failover.

## Network Topology

```
Machine A (site)                          Machine B (cloud concentrator)
┌─────────────────────┐                   ┌─────────────────────┐
│                     │                   │                     │
│  eth1  192.168.1.x ─┼───── path 1 ─────┼─▶                   │
│  eth2  10.0.0.x    ─┼───── path 2 ─────┼─▶  eth0  203.0.113.1│
│  eth3  172.16.0.x  ─┼───── path 3 ─────┼─▶                   │
│  eth4  10.1.0.x    ─┼───── path 4 ─────┼─▶                   │
│                     │                   │                     │
│          tun0 ──────┼───── tunnel ──────┼────── tun0          │
└─────────────────────┘                   └─────────────────────┘
```

The example addresses below are placeholders. Replace them with the actual
DHCP-assigned addresses on Machine A and the real public IP of Machine B.

## Prerequisites

Glorytun creates TUN interfaces via `/dev/net/tun` on Linux. This requires:

1. **The `tun` kernel module must be loaded.** Check and load it on both
   machines:

```sh
lsmod | grep tun
modprobe tun
```

To make it persistent across reboots:

```sh
echo tun >> /etc/modules-load.d/tun.conf
```

2. **`/dev/net/tun` must exist.** On most distributions this is created
   automatically when the module loads. If it is missing:

```sh
mkdir -p /dev/net
mknod /dev/net/tun c 10 200
chmod 0666 /dev/net/tun
```

3. **Root privileges (or `CAP_NET_ADMIN`).** Creating TUN devices and
   configuring interfaces requires elevated permissions. Run glorytun as root
   or grant the capability:

```sh
setcap cap_net_admin+ep /usr/local/bin/glorytun
```

4. **A key directory.** Create it on both machines:

```sh
mkdir -p /etc/glorytun
```

## Step 1 — Generate a Shared Key (once, on either machine)

```sh
glorytun keygen > /etc/glorytun/key
chmod 600 /etc/glorytun/key
```

Copy `/etc/glorytun/key` to the other machine via a secure channel (e.g. `scp`).
Both sides must use the same key.

## Step 2 — Start the Tunnel on Machine B (concentrator)

Machine B listens on its public IP, port 5000. No `to` is specified, so it
runs in server mode — it accepts incoming connections from any peer.

`glorytun bind` creates the TUN interface and runs in the foreground. Use `&`
to background it, or run it in a dedicated terminal / systemd service.

```sh
glorytun bind 203.0.113.1 5000 dev tun0 keyfile /etc/glorytun/key persist &
```

The `persist` flag keeps the TUN interface alive if the process exits. Without
it, the interface disappears when glorytun stops.

Wait a moment for the interface to appear, then assign an IP and bring it up
(glorytun creates the interface but does not configure addressing or link state):

```sh
ip addr add 10.99.0.1/30 dev tun0
ip link set tun0 up
```

## Step 3 — Start the Tunnel on Machine A (site)

Machine A binds to all local addresses (default `0.0.0.0`) and connects to
Machine B's public IP using `to`.

```sh
glorytun bind to 203.0.113.1 5000 dev tun0 keyfile /etc/glorytun/key persist &
```

Wait for the interface, then configure it:

```sh
ip addr add 10.99.0.2/30 dev tun0
ip link set tun0 up
```

At this point the tunnel is up with a single default path. The next steps
configure source-based routing and register the additional interfaces as
separate paths.

**Troubleshooting:** If `glorytun bind` prints `couldn't create tun device`,
check that the `tun` module is loaded and `/dev/net/tun` exists (see
[Prerequisites](#prerequisites)).

## Step 4 — Source-Based Policy Routing on Machine A

Linux routes all outbound packets based on the **destination** in the main
routing table. Without additional configuration, all four paths' packets exit
through whichever interface has the default route — even though Glorytun sets
the correct source IP via `IP_PKTINFO`. This means only one path works and the
rest are DEGRADED.

The fix is **source-based policy routing**: a separate routing table per
interface, with a rule that selects the table based on the packet's source
address.

Retrieve the current DHCP addresses and gateways:

```sh
ETH1_IP=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1)
ETH2_IP=$(ip -4 -o addr show eth2 | awk '{print $4}' | cut -d/ -f1)
ETH3_IP=$(ip -4 -o addr show eth3 | awk '{print $4}' | cut -d/ -f1)
ETH4_IP=$(ip -4 -o addr show eth4 | awk '{print $4}' | cut -d/ -f1)

ETH1_GW=$(ip -4 route show dev eth1 | awk '/default/ {print $3}')
ETH2_GW=$(ip -4 route show dev eth2 | awk '/default/ {print $3}')
ETH3_GW=$(ip -4 route show dev eth3 | awk '/default/ {print $3}')
ETH4_GW=$(ip -4 route show dev eth4 | awk '/default/ {print $3}')
```

Create a routing table and policy rule for each interface:

```sh
ip route add default via $ETH1_GW dev eth1 table 101
ip rule add from $ETH1_IP table 101

ip route add default via $ETH2_GW dev eth2 table 102
ip rule add from $ETH2_IP table 102

ip route add default via $ETH3_GW dev eth3 table 103
ip rule add from $ETH3_IP table 103

ip route add default via $ETH4_GW dev eth4 table 104
ip rule add from $ETH4_IP table 104
```

Disable strict reverse path filtering (which can drop packets arriving on
"unexpected" interfaces in a multi-homed setup):

```sh
sysctl -w net.ipv4.conf.all.rp_filter=2
sysctl -w net.ipv4.conf.default.rp_filter=2
```

Value `2` is loose mode — it checks the source is reachable via *any* interface
rather than requiring the *same* interface the packet arrived on.

Verify that each source address routes through the correct interface:

```sh
ip route get 203.0.113.1 from $ETH1_IP   # should show dev eth1
ip route get 203.0.113.1 from $ETH2_IP   # should show dev eth2
ip route get 203.0.113.1 from $ETH3_IP   # should show dev eth3
ip route get 203.0.113.1 from $ETH4_IP   # should show dev eth4
```

To make these rules persistent across reboots, add them to your network
configuration (e.g. `/etc/network/interfaces` post-up scripts, NetworkManager
dispatcher, or netplan routing policies).

## Step 5 — Add Paths for Each Interface on Machine A

Each `path` command tells Glorytun to use a specific local address (bound to a
physical interface) as a separate path. The local IP address is given as a bare
positional argument (no keyword prefix).

The `$ETH1_IP` .. `$ETH4_IP` variables were already set in the previous step.

Register each path, bring it up, and **set a rate**. A rate is required for data
to flow — without it, only control messages are exchanged and the path stays at
0 bytes/sec throughput. Use `rate fixed tx <rate> rx <rate>` with the link speed,
or `rate auto tx <rate> rx <rate>` for dynamic adjustment with an upper bound.

**Heterogeneous links:** Glorytun assigns each packet to a path using weights
proportional to each path’s configured `tx` rate (see `mud_select_path` in the
codebase). If every path uses the same `tx`/`rx` (e.g. the default `12500k` from
the OpenRC/systemd templates), traffic is split **evenly** across paths — not
according to real link speed. A slow link then sees a disproportionate share of
traffic, loss rises above `losslimit`, and that path stops being used (`ok` goes
false). Throughput can collapse to something closer to **one surviving path**
than the sum of all links. For 100 / 10 / 40 / 40 Mbit paths, set **per-path**
`tx`/`rx` in proportion to each capacity (or use `auto` with a **per-path**
ceiling matched to that link).

Here each link is assumed to be 100 Mbit (12,500 KB/s):

```sh
glorytun path $ETH1_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH2_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH3_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH4_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
```

The `rx` rate is communicated to the server via control messages, so the server
automatically learns the maximum rate it can send back on each path. You only
need to configure rates on the client side.

## Step 6 — Verify

Check the tunnel status:

```sh
glorytun show dev tun0
```

List all paths and their state:

```sh
glorytun path dev tun0
```

Check a specific path:

```sh
glorytun path $ETH1_IP dev tun0
```

## Optional — Rate Limiting

If the WAN links have known bandwidth caps, apply fixed rate limits to prevent
congestion. For example, if eth1 and eth2 are 100 Mbit links and eth3 and eth4
are 50 Mbit links:

```sh
glorytun path $ETH1_IP dev tun0 rate fixed tx 12500k rx 12500k
glorytun path $ETH2_IP dev tun0 rate fixed tx 12500k rx 12500k
glorytun path $ETH3_IP dev tun0 rate fixed tx 6250k rx 6250k
glorytun path $ETH4_IP dev tun0 rate fixed tx 6250k rx 6250k
```

Rate values are in bytes/sec (with size suffixes). To specify in bits, use the
bit suffix: `tx 100Mbit rx 100Mbit`.

Alternatively, use `rate auto` to let Glorytun detect available bandwidth
dynamically:

```sh
glorytun path $ETH1_IP dev tun0 rate auto
```

## Optional — Backup Paths for Failover

To designate some links as backups that only activate when primary paths
degrade, use `backup` instead of `up`:

```sh
glorytun path $ETH1_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH2_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH3_IP dev tun0 backup beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
glorytun path $ETH4_IP dev tun0 backup beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
```

Here eth1 and eth2 are active paths. If they both degrade, eth3 and eth4 take
over automatically.

## Optional — Tuning Tunnel Properties

Adjust key exchange timeout, time tolerance, keepalive, and traffic class on a
running tunnel:

```sh
glorytun set dev tun0 kxtimeout 1h timetolerance 10s keepalive 25s
```

Set a DSCP traffic class for tunnel packets (e.g. CS6 for network control):

```sh
glorytun set dev tun0 tc CS6
```

## Handling DHCP Renewals

When a DHCP lease changes the IP on an interface, the old path becomes stale.
To update, bring the old path down and register the new address:

```sh
OLD_IP="192.168.1.100"
NEW_IP="192.168.1.200"

glorytun path $OLD_IP dev tun0 down
glorytun path $NEW_IP dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k
```

This can be automated with a DHCP hook script.

## Summary of Commands

| Step | Machine | Command |
|------|---------|---------|
| Key generation | Either | `glorytun keygen > /etc/glorytun/key` |
| Start tunnel | B (server) | `glorytun bind 203.0.113.1 5000 dev tun0 keyfile /etc/glorytun/key persist` |
| Start tunnel | A (client) | `glorytun bind to 203.0.113.1 5000 dev tun0 keyfile /etc/glorytun/key persist` |
| Add path | A | `glorytun path <local-ip> dev tun0 up beat 5s losslimit 30% rate fixed tx 12500k rx 12500k` |
| Check tunnel | Either | `glorytun show dev tun0` |
| Check paths | Either | `glorytun path dev tun0` |
