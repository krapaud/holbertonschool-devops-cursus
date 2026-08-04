# TCP Three-Way Handshake — example.com over HTTPS

## Capture setup

```bash
sudo tcpdump -i en0 -n -s 0 -w handshake.pcap 'host example.com and tcp port 443'
```

(`-i any` from the original instructions does not exist on macOS — `any`
is a Linux-only pseudo-interface — so the capture was run on `en0`, the
active Wi-Fi interface identified in [inventory.md](inventory.md).)

In a second terminal, while the capture was running:

```bash
curl -I https://example.com
```

`example.com` resolved to two Cloudflare-fronted addresses
(`104.20.23.154` and `172.66.147.243`); the capture and the connection
below both used `104.20.23.154`.

- Client: `10.6.0.242` (this host, `en0`), ephemeral port `50732`
- Server: `104.20.23.154`, port `443` (HTTPS)
- Total packets captured: 26 (full connection: handshake, TLS/HTTP
  exchange, and the 4-way FIN teardown)

Read back with:

```bash
tcpdump -r handshake.pcap -n -v
```

## The three-way handshake

### Packet 1 — SYN (client → server)

```
11:04:41.978946 IP 10.6.0.242.50732 > 104.20.23.154.443:
  Flags [SEW], seq 3896645866, win 65535,
  options [mss 1460, wscale 6, sackOK, TS val 1036960064 ecr 0],
  length 0
```

| Field | Value | Meaning |
| - | - | - |
| Flags | `SEW` = **SYN, ECE, CWR** | Client opens the connection and, via ECE+CWR both set on the SYN, requests **ECN** (Explicit Congestion Notification) support (RFC 3168) |
| Sequence number | `3896645866` | Client's Initial Sequence Number (ISN), randomly chosen |
| Ack number | — | none — no data has been received yet |
| Window size | `65535` | Client's initial receive window (before scaling is applied) |
| Options | `mss 1460` | Max segment size the client can receive |
| | `wscale 6` | Window scaling factor of 2^6 = 64, to be applied to future window values |
| | `sackOK` | Selective ACK supported |
| | `TS val 1036960064 ecr 0` | TCP timestamp option; `ecr 0` because no prior timestamp was echoed |
| Length | `0` | SYN carries no payload |

### Packet 2 — SYN/ACK (server → client)

```
11:04:41.988504 IP 104.20.23.154.443 > 10.6.0.242.50732:
  Flags [S.E], seq 2469979034, ack 3896645867, win 65535,
  options [mss 1400, sackOK, TS val 1715828597 ecr 1036960064, wscale 13],
  length 0
```

| Field | Value | Meaning |
| - | - | - |
| Flags | `S.E` = **SYN, ACK, ECE** | Server acknowledges the client's SYN and accepts ECN (ECE set alone on a SYN/ACK = "yes, I support ECN", per RFC 3168 — CWR is not set here) |
| Sequence number | `2469979034` | Server's own ISN |
| Ack number | `3896645867` | = client ISN + 1 (`3896645866 + 1`), confirming receipt of the SYN |
| Window size | `65535` | Server's initial receive window (before scaling) |
| Options | `mss 1400` | Max segment size the server can receive |
| | `sackOK` | Selective ACK supported |
| | `TS val 1715828597 ecr 1036960064` | Server's timestamp, echoing the client's timestamp from packet 1 |
| | `wscale 13` | Window scaling factor of 2^13 = 8192 |
| Length | `0` | SYN/ACK carries no payload |

Round-trip so far: **9.558 ms** (`11:04:41.988504 - 11:04:41.978946`).

### Packet 3 — ACK (client → server)

```
11:04:41.988626 IP 10.6.0.242.50732 > 104.20.23.154.443:
  Flags [.], ack 1, win 2061,
  options [TS val 1036960074 ecr 1715828597],
  length 0
```

| Field | Value | Meaning |
| - | - | - |
| Flags | `.` = **ACK** only | Final leg of the handshake, no SYN — connection is now `ESTABLISHED` |
| Sequence number | (relative `1`, not shown) | Client's next byte to send = ISN + 1 |
| Ack number | `1` (relative) | tcpdump shows sequence/ack numbers **relative to the ISN** by default once it has seen the SYN; in absolute terms this is `2469979034 + 1 = 2469979035`, confirming the server's SYN |
| Window size | `2061` | Client's advertised window; this is the **scaled** value (raw window x `wscale 6` from packet 1) |
| Options | `TS val 1036960074 ecr 1715828597` | Client's timestamp, echoing the server's timestamp from packet 2 |
| Length | `0` | Pure ACK, no payload — the actual TLS ClientHello is sent in the next packet |

Time since SYN: **0.122 ms** after the SYN/ACK — the client replies almost
immediately.

## Summary

```
Client (10.6.0.242:50732)              Server (104.20.23.154:443)
        |------ SYN, seq=3896645866 ------------->|
        |<--- SYN/ACK, seq=2469979034, -----------|
        |          ack=3896645867                 |
        |------ ACK, ack=2469979035 -------------->|
   [connection ESTABLISHED — next packet: TLS ClientHello]
```

- The handshake completed in **9.68 ms** total (SYN to final ACK).
- Both sides negotiated `wscale`, `sackOK`, and TCP timestamps — all
  standard modern TCP options, none of which appear in the base TCP
  header itself (RFC 793) but are carried in the variable-length
  options field.
- The `E`/`W` flags on the SYN and `E` on the SYN/ACK show this
  connection negotiated **ECN** (RFC 3168) in addition to the plain
  three-way handshake — visible corroboration is the `ECT(0)` mark
  (`tos 0x2`) set in the IP header of the data packets that follow.
- After the handshake, the capture shows the TLS record layer starting
  immediately (`Flags [P.]`, `seq 1:322`, `length 321` — the
  ClientHello) and the ACK teardown/FIN exchange at the end of the
  `curl -I` request/response.
