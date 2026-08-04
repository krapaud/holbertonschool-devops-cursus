# TCP vs UDP — HTTP request vs DNS query

## Capture setup

### DNS (UDP)

```bash
sudo tcpdump -i en0 -n -s 0 -w dns_query.pcap 'udp port 53 and host 10.6.3.254'
dig example.com
```

`dns_query.pcap` captured 6 packets total: the 2 relevant to
`dig example.com` (query + answer), plus 4 unrelated background DNS
lookups issued by macOS itself during the same window (Microsoft
telemetry endpoint resolution via `mDNSResponder`). The filter could
not scope to `example.com` specifically ahead of time since the
domain being looked up isn't known to `tcpdump` — only the 2 packets
below are used for the analysis; the other 4 are noise from an
unrelated process sharing the same resolver.

### HTTP (TCP)

```bash
sudo tcpdump -i en0 -n -s 0 -w http_request.pcap 'tcp port 80 and host example.com'
curl http://example.com
```

`http_request.pcap` captured 11 packets — the entire TCP connection
lifecycle for one HTTP GET request.

## Raw packets

### DNS — 2 packets, no handshake

```console
11:08:52.417186 IP 10.6.0.242.52600 > 10.6.3.254.53:
  47709+ [1au] A? example.com. (40)

11:08:52.453690 IP 10.6.3.254.53 > 10.6.0.242.52600:
  47709 2/0/1 A 104.20.23.154, A 172.66.147.243 (72)
```

- Query issued directly, no connection setup.
- Response arrives **36.5 ms** later, matching `dig`'s own reported
  `Query time: 38 msec`.
- Query and response share the same transaction ID (`47709`) — that ID,
  not a TCP connection state, is how the client matches the response to
  its request.
- `2/0/1` = 2 answer records, 0 authority records, 1 additional record
  (the EDNS OPT pseudosection sent in the query, `[1au]`).
- Done. No teardown — UDP has no concept of a "connection" to close.

### HTTP — 11 packets, full TCP lifecycle

```console
11:07:32.845979 10.6.0.242.50752 > 104.20.23.154.80: Flags [SEW]   seq 1328620071                         len 0   (SYN)
11:07:32.855640 104.20.23.154.80 > 10.6.0.242.50752: Flags [S.E]   seq 485699919 ack 1328620072            len 0   (SYN/ACK)
11:07:32.855780 10.6.0.242.50752 > 104.20.23.154.80: Flags [.]     ack 1                                   len 0   (ACK — handshake done)
11:07:33.084691 10.6.0.242.50752 > 104.20.23.154.80: Flags [P.]    seq 1:75    ack 1                       len 74  (GET / HTTP/1.1 ...)
11:07:33.097099 104.20.23.154.80 > 10.6.0.242.50752: Flags [.]     ack 75                                  len 0   (ACK of the GET)
11:07:33.105057 104.20.23.154.80 > 10.6.0.242.50752: Flags [P.]    seq 1:870   ack 75                      len 869 (HTTP/1.1 200 OK + body)
11:07:33.105059 104.20.23.154.80 > 10.6.0.242.50752: Flags [P.]    seq 870:875 ack 75                      len 5   (trailing chunk terminator)
11:07:33.105148 10.6.0.242.50752 > 104.20.23.154.80: Flags [.]     ack 875                                 len 0   (ACK of response)
11:07:33.105678 10.6.0.242.50752 > 104.20.23.154.80: Flags [F.]    seq 75      ack 875                     len 0   (client FIN)
11:07:33.119025 104.20.23.154.80 > 10.6.0.242.50752: Flags [F.]    seq 875     ack 76                      len 0   (server FIN)
11:07:33.119145 10.6.0.242.50752 > 104.20.23.154.80: Flags [.]     ack 876                                 len 0   (final ACK)
```

- 3 packets just to **open** the connection (SYN / SYN-ACK / ACK)
  before a single byte of HTTP data is sent.
- 3 packets to **close** it (FIN / FIN-ACK / ACK) — a 4-way teardown
  here since the client's FIN and the server's last-data ACK were
  already merged/simultaneous.
- Every data segment is individually **acknowledged** (`ack` field
  incrementing byte-by-byte), and each side tracks a `seq`/`ack` pair.
- Total: 11 packets exchanged to fetch a ~495-byte HTML page — compared
  to 2 packets for a DNS answer of similar or smaller size.

## Side-by-side comparison

| | DNS query (UDP, port 53) | HTTP GET (TCP, port 80) |
| - | - | - |
| Protocol | UDP (RFC 768) | TCP (RFC 793) |
| Connection setup | None — client sends datagram directly | 3-way handshake (SYN, SYN/ACK, ACK) before any data |
| Total packets captured | 2 (query + response) | 11 (handshake, request, response, ACKs, teardown) |
| "Flags" concept | None — UDP header has no flags field | Present on every segment: `SEW` (SYN,ECE,CWR), `S.E` (SYN,ACK,ECE), `.` (ACK), `P.` (PSH,ACK), `F.` (FIN,ACK) |
| Sequence/ack numbers | None — no ordering state kept | Present on every packet; each byte is numbered and acknowledged |
| Delivery guarantee | Best-effort — no retransmission at the transport layer | Reliable — kernel retransmits unacked segments automatically |
| Matching request ↔ response | Application-level transaction ID in the DNS header (`47709`) | Transport-level: 4-tuple (src IP/port, dst IP/port) + seq/ack state |
| Teardown | None — datagram-based, nothing to close | Explicit FIN/ACK exchange from both sides |
| Overhead for this exchange | 40 + 72 = 112 bytes on the wire, 1 round trip | 64+60+52+126+52+921+57+52+52+52+52 = 1540 bytes on the wire, several round trips |
| Time to complete | ~36.5 ms (single round trip) | ~273 ms (`11:07:32.845979` → `11:07:33.119145`), spanning 5+ round trips |
| Retransmission on packet loss | Handled at the **application** layer — `dig`/the resolver library times out and re-sends the whole query if no answer arrives | Handled **transparently by the kernel** — only the missing segment(s) are re-sent, with exponential backoff, invisible to the application |

## Retransmission behavior (stretch goal — not run)

The stretch goal suggests dropping a packet with `tc qdisc` to observe
retransmission. `tc` is part of Linux's `iproute2` and has no
equivalent on macOS (the closest analogues are `pfctl`/`dnctl`, the
BSD packet filter + dummynet, which use a different mechanism for
traffic shaping/loss injection). This was skipped since it requires
Linux tooling not available on this host. Conceptually, based on
RFC 793 §3.7 and RFC 1035 §7.2:

- **TCP**: if the GET request or the response segments above were
  dropped, the sender's retransmission timer (RTO) would expire and the
  kernel would resend the unacknowledged segment automatically —
  `curl` would simply see a slower response, with no application-level
  awareness that a retransmission occurred.
- **DNS/UDP**: if the query or answer packet above were dropped, there
  is no transport-level recovery. `dig` itself implements a timeout
  (default 5s) and a retry count (default 2 additional tries) — a lost
  DNS query becomes a second, brand-new UDP datagram with the *same*
  transaction ID, entirely driven by the application, not the kernel.
