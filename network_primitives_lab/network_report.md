# Network Primitives Lab — Report

Hands-on investigation of the networking stack underneath a single
HTTPS request: interface/routing config, subnetting math, a captured
TCP handshake, a TCP-vs-UDP comparison, and an end-to-end DNS trace.
All raw evidence (`.pcap` files, `dig` output) lives alongside this
report and is referenced below.

## Sub-documents

| Task | Document | Evidence |
| - | - | - |
| 0. Network inventory | [inventory.md](inventory.md) | Interfaces, MACs, default gateway, DNS servers, public IP |
| 1. Subnetting by hand | [subnetting.md](subnetting.md) | `10.42.0.0/16` split into 4x `/18`, verified with `ipcalc` |
| 2. TCP handshake capture | [tcp_handshake.md](tcp_handshake.md) | SYN/SYN-ACK/ACK annotated — raw capture: [handshake.pcap](handshake.pcap) |
| 3. TCP vs UDP | [tcp_vs_udp.md](tcp_vs_udp.md) | DNS (UDP) vs HTTP (TCP) compared — raw captures: [dns_query.pcap](dns_query.pcap), [http_request.pcap](http_request.pcap) |
| 4. DNS deep dive | [dns_report.md](dns_report.md) | Delegation chain explained — raw trace: [dns_trace.txt](dns_trace.txt) |

## How a browser request for `https://example.com` actually flows

```
 MY LAPTOP                                                              ORIGIN
┌───────────────────────────┐
│ Browser: "https://example.com"
└─────────────┬─────────────┘
              │
              │ 1. DNS RESOLUTION  (see dns_report.md)
              ▼
     ┌─────────────────┐        referral         ┌──────────────────┐
     │ Local resolver   │ ───────────────────────▶│ Root server (.)   │
     │ 10.6.3.254 (LAN) │◀─────────────────────── │ "ask a .com GTLD" │
     └────────┬─────────┘                         └──────────────────┘
              │                  referral          ┌──────────────────┐
              │ ───────────────────────────────────▶│ .com TLD server  │
              │◀─────────────────────────────────── │ "ask cloudflare"  │
              │                                     └──────────────────┘
              │                  authoritative       ┌──────────────────┐
              │ ───────────────────────────────────▶│ hera.ns.cloudflare │
              │◀─────────────────────────────────── │ .com (authoritative)│
              │        A 104.20.23.154               └──────────────────┘
              │        A 172.66.147.243
              ▼
     Browser now has: 104.20.23.154

              │
              │ 2. ROUTING  (see inventory.md)
              ▼
     ┌─────────────────┐   ARP/local subnet   ┌──────────────────┐
     │ en0 (10.6.0.242) │─────────────────────▶│ Default gateway   │
     │ my Wi-Fi NIC     │                      │ 10.6.3.254         │
     └─────────────────┘                      └─────────┬──────────┘
                                                          │ NAT: 10.6.0.242
                                                          │  → 82.122.123.13
                                                          ▼
                                              ┌────────────────────────┐
                                              │ ISP / internet routers  │
                                              │ (multiple hops, unseen  │
                                              │  in this lab — mtr/     │
                                              │  traceroute territory)  │
                                              └───────────┬─────────────┘
                                                           ▼
                                            ┌───────────────────────────┐
                                            │ Cloudflare edge            │
                                            │ 104.20.23.154:443           │
                                            └──────────────┬─────────────┘
              │                                            │
              │ 3. TCP HANDSHAKE (see tcp_handshake.md)    │
              │───────── SYN (seq=x) ──────────────────────▶
              │◀──────── SYN/ACK (seq=y, ack=x+1) ──────────
              │───────── ACK (ack=y+1) ─────────────────────▶
              │              [connection ESTABLISHED]
              │
              │ 4. TLS HANDSHAKE (not captured in this lab —
              │    ClientHello/ServerHello/cert exchange happen
              │    inside the TCP connection just opened, before
              │    any HTTP bytes are sent)
              │
              │ 5. HTTP REQUEST/RESPONSE (see tcp_vs_udp.md for the
              │    plain-HTTP/port-80 packet-level example)
              │───────── GET / HTTP/1.1 ────────────────────▶
              │◀──────── HTTP/1.1 200 OK + HTML body ────────
              │
              │ 6. TEARDOWN
              │───────── FIN ────────────────────────────────▶
              │◀──────── FIN/ACK ─────────────────────────────
              ▼
     Browser renders the page
```

Layer-by-layer summary:

1. **DNS** resolves the name to an IP through a chain of delegated
   authorities (root → `.com` → Cloudflare) — detailed in
   [dns_report.md](dns_report.md).
2. **IP routing** gets packets from my laptop's private address
   (`10.6.0.242`) to my gateway, which NATs them to my public IP
   (`82.122.123.13`) before they leave my network — detailed in
   [inventory.md](inventory.md).
3. **TCP** opens a reliable, ordered byte stream to the server with a
   three-way handshake — captured and annotated in
   [tcp_handshake.md](tcp_handshake.md).
4. **TLS** (not captured here) negotiates encryption inside that TCP
   connection before any HTTP data is exchanged — this is why the port
   is 443, not 80.
5. **HTTP** is the actual request/response exchanged over the now-open,
   now-encrypted connection — the plain-text mechanics of this step are
   shown on port 80 in [tcp_vs_udp.md](tcp_vs_udp.md), since HTTPS
   traffic itself is encrypted and unreadable in a packet capture.
6. **Teardown** closes the TCP connection with a FIN/ACK exchange from
   both sides.

## Lessons learned

The most surprising thing wasn't any single protocol detail but how
much *invisible* traffic surrounds a single "simple" request: capturing
DNS on port 53 with a loose filter pulled in a stream of background
lookups from macOS system services (Microsoft telemetry, Akamai,
mDNSResponder chatter) that had nothing to do with the `dig` command I
was actually running, and made me realize a modern laptop is
constantly resolving names on its own initiative. On the protocol side,
the biggest surprise was how much overhead TCP pays *before* useful
data even moves — 3 packets and a full round trip just to open a
connection, versus a DNS query that got its answer in a single
UDP datagram round trip — which makes very concrete why protocols like
QUIC/HTTP-3 exist to collapse that setup cost. Watching the ECN flags
(`ECE`/`CWR`) show up unprompted on every SYN was also unexpected: I
hadn't registered that ordinary `curl`/`dig` traffic on this machine is
already negotiating congestion-notification capability by default, not
just the "flags I studied" (SYN/ACK/FIN) — a reminder that RFC 793's
header has grown a lot of optional machinery that's silently in use on
every connection.
