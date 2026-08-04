# Network Inventory

> Host: macOS (Darwin 25.5.0, macOS 26.5.2)
> Generated: 2026-08-04
>
> Note: this host runs macOS, which has no `ip` command (iproute2 is
> Linux-only). The commands below use the macOS/BSD equivalents; the
> Linux command each one replaces is noted in parentheses so the report
> stays reproducible on either platform.

## 1. Hostname

```console
$ hostname
MacBook-Pro-de-Mickael.local
```

## 2. Network interfaces (IPv4 / IPv6 / MAC)

Command used: `ifconfig` (macOS equivalent of `ip addr` / `ip link`).

Only interfaces that are `UP` and carry an address are listed; the full
`ifconfig` output also includes inactive virtual/companion interfaces
(`anpi0-3`, `en1-en9`, `bridge0`, `ap1`, `gif0`, `stf0`) which are not
in active use on this host.

| Interface | Status | MAC address | IPv4 | IPv6 |
| - | - | - | - | - |
| `lo0` | active (loopback) | - | `127.0.0.1/8` | `::1/128`, `fe80::1%lo0/64` |
| `en0` | active (Wi-Fi, primary) | `46:15:63:69:ee:9a` | `10.6.0.242/22` (broadcast `10.6.3.255`) | `fe80::1485:11b6:92da:18f6%en0/64` (link-local, secured) |
| `awdl0` | active (Apple Wireless Direct Link) | `8a:c7:6c:23:28:b0` | - | `fe80::88c7:6cff:fe23:28b0%awdl0/64` |
| `llw0` | active (low-latency WLAN) | `8a:c7:6c:23:28:b0` | - | `fe80::88c7:6cff:fe23:28b0%llw0/64` |
| `utun0` | active (VPN/system tunnel) | - | - | `fe80::404e:6bae:17d8:5a70%utun0/64` |
| `utun1` | active (VPN/system tunnel) | - | - | `fe80::4895:d1cf:f13b:f976%utun1/64` |
| `utun2` | active (VPN/system tunnel) | - | - | `fe80::f1e1:b897:57e6:b05c%utun2/64` |
| `utun3` | active (VPN/system tunnel) | - | - | `fe80::ce81:b1c:bd2c:69e%utun3/64` |

`en0` is the primary interface: it holds the only routable IPv4 address
and the default route goes out through it (see below). No global-scope
IPv6 address is assigned on this host — only link-local (`fe80::/10`)
addresses are present.

## 3. Default gateway

Command used: `netstat -rn -f inet` (macOS equivalent of `ip route`).

```console
Destination        Gateway            Flags               Netif Expire
default            10.6.3.254         UGScg                 en0
```

- Default gateway: `10.6.3.254`
- Reached via: `en0`
- Local subnet: `10.6.0.0/22` (netmask `255.255.252.0`, matches `en0`'s
  `10.6.0.242/22`)

## 4. DNS servers

Command used: `scutil --dns` (macOS does not consult `/etc/resolv.conf`
directly for resolution — that file is auto-generated for
compatibility only and macOS prints a notice saying so; `resolvectl` is
Linux/systemd-specific and not available here).

```console
resolver #1
  search domain[0] : hbtn.fr
  nameserver[0]     : 10.6.3.254
  if_index          : 16 (en0)
```

- DNS server: `10.6.3.254` (same host as the default gateway — typical
  of a router/DHCP server also acting as local resolver)
- Search domain: `hbtn.fr`

The auto-generated `/etc/resolv.conf` agrees:

```console
search hbtn.fr
nameserver 10.6.3.254
```

## 5. Public IP address

Command used: `curl -s https://ifconfig.me`

```console
82.122.123.13
```

This is the NAT/ISP-facing public IPv4 address, distinct from the
private `10.6.0.242` assigned to `en0` — traffic from this host is
NAT-translated by the gateway/router before reaching the internet.

## Reproduction

All of the above can be regenerated with:

```bash
hostname
ifconfig
netstat -rn -f inet
scutil --dns
cat /etc/resolv.conf
curl -s https://ifconfig.me
```

(On Linux, substitute `ip addr` / `ip link` for `ifconfig`, `ip route`
for `netstat -rn -f inet`, and `resolvectl status` for `scutil --dns`.)
