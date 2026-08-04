# DNS Deep Dive

## 1. End-to-end trace: `dig +trace example.com`

Full raw output saved in [dns_trace.txt](dns_trace.txt), generated with:

```bash
dig +trace example.com > dns_trace.txt
```

`+trace` tells `dig` to bypass the local recursive resolver's normal
one-shot behavior and instead walk the delegation chain itself,
starting from a root server and following referrals down to the
authoritative server — showing every hop along the way instead of just
the final answer.

### Servers identified at each stage

| Stage | Zone | Servers returned | Server that actually answered this hop |
| - | - | - | - |
| Root | `.` | `a.root-servers.net` … `m.root-servers.net` (13 root servers) | `10.6.3.254` (local resolver, already caches root hints) |
| TLD | `com.` | `a.gtld-servers.net` … `m.gtld-servers.net` (13 `.com` GTLD servers) | `k.root-servers.net` (`193.0.14.129`) |
| Authoritative | `example.com.` | `hera.ns.cloudflare.com.`, `elliott.ns.cloudflare.com.` | `b.gtld-servers.net` (`192.33.14.30`) |
| Final answer | `example.com.` | `A 172.66.147.243`, `A 104.20.23.154` | `hera.ns.cloudflare.com` (`172.64.32.162`) |

Each line in the trace also carries `DS`/`RRSIG` records — these are
DNSSEC signatures proving the chain of trust from the root down to
`example.com`, confirming each delegation is cryptographically
authenticated, not just referred.

## 2. The delegation chain, explained

DNS resolution is not one server "knowing" the answer — it's a chain
of referrals, each server pointing to the next, more specific
authority:

1. **Root zone (`.`)** — My resolver already has the 13 root server
   addresses hardcoded (root hints), so this "hop" was answered
   locally in 12 ms without a real network round trip to a root
   server. The root servers don't know where `example.com` is, but
   they know who is authoritative for `.com`, and hand back that list
   of 13 GTLD servers.

2. **TLD zone (`com.`)** — `dig` asked one of those root-listed
   servers (`k.root-servers.net`) "who handles `example.com`?". The
   root server doesn't answer that directly either — it doesn't run
   `.com`, it just knows who does. It returns a **referral**: the list
   of 13 `.com` GTLD servers, plus a `DS` record (the DNSSEC delegation
   signer) that lets the resolver verify the next hop is genuinely
   delegated, not spoofed.

3. **Authoritative zone (`example.com.`)** — `dig` then asked one of
   those GTLD servers (`b.gtld-servers.net`). This server *does* know
   about `example.com` — not its IP address, but which nameservers are
   authoritative for it: Cloudflare's `hera.ns.cloudflare.com` and
   `elliott.ns.cloudflare.com`. This is the actual **delegation**: the
   `.com` zone has handed off responsibility for everything under
   `example.com` to Cloudflare's nameservers.

4. **Final answer** — `dig` finally asked `hera.ns.cloudflare.com`
   directly, and *this* server is authoritative — it holds the real
   `A` records and answers with the actual IP addresses
   (`172.66.147.243`, `104.20.23.154`), signed with `RRSIG`.

In short: **each server in the chain only knows the next server down**,
never the final answer, except the last one. Resolution is a sequence
of "I don't know, but ask them" referrals, from the most general zone
(`.`) to the most specific one (`example.com.`), which is exactly what
"delegation" means in DNS — each parent zone delegates authority over
a subtree to a specific set of nameservers, recorded as `NS` records
plus (with DNSSEC) `DS`/`RRSIG` records proving the delegation is
authentic.

My local resolver (`10.6.3.254`) normally does all 4 of these hops
itself and caches the results — `+trace` is what forces `dig` to
perform (and show) every hop itself instead of trusting the resolver's
single cached answer.

## 3. Record types for a domain of my choice: `wikipedia.org`

Commands used: `dig +noall +answer wikipedia.org <TYPE>` for each type
(`+noall +answer` trims the output down to just the answer section).

### A (IPv4 address)

```console
$ dig +noall +answer wikipedia.org A
wikipedia.org.		151	IN	A	185.15.58.224
```

Maps the domain to an IPv4 address.

### AAAA (IPv6 address)

```console
$ dig +noall +answer wikipedia.org AAAA
wikipedia.org.		100	IN	AAAA	2a02:ec80:600:ed1a::1
```

Same purpose as `A`, but for IPv6 — Wikipedia is dual-stacked.

### MX (mail exchange)

```console
$ dig +noall +answer wikipedia.org MX
wikipedia.org.		300	IN	MX	10 mx-in1001.wikimedia.org.
wikipedia.org.		300	IN	MX	10 mx-in2001.wikimedia.org.
```

Tells other mail servers where to deliver email for `@wikipedia.org`.
Both entries share priority `10`, meaning either can be used —
typically for load-balancing/redundancy rather than strict failover
(a lower number would mean higher priority).

### TXT (free-form text)

```console
$ dig +noall +answer wikipedia.org TXT
wikipedia.org.		600	IN	TXT	"yandex-verification: 35c08d23099dc863"
wikipedia.org.		600	IN	TXT	"v=spf1 include:_cidrs.wikimedia.org ~all"
wikipedia.org.		600	IN	TXT	"google-site-verification=AMHkgs-4ViEvIJf5znZle-BSE2EPNFqM1nDJGRyn2qk"
```

Arbitrary text used for domain verification (Yandex, Google Search
Console) and, notably, an **SPF record** (`v=spf1 ...`) — this tells
receiving mail servers which hosts are allowed to send email claiming
to be `@wikipedia.org`, an anti-spoofing measure.

### NS (nameserver / delegation)

```console
$ dig +noall +answer wikipedia.org NS
wikipedia.org.		165830	IN	NS	ns0.wikimedia.org.
wikipedia.org.		165830	IN	NS	ns1.wikimedia.org.
wikipedia.org.		165830	IN	NS	ns2.wikimedia.org.
```

The authoritative nameservers for the `wikipedia.org` zone itself —
this is the same kind of record seen in the delegation chain above,
just for Wikimedia instead of Cloudflare.

### CNAME (canonical name / alias)

`wikipedia.org` (the zone apex) cannot have a `CNAME` — RFC 1034 §3.6.2
forbids a `CNAME` from coexisting with other record types (like the
`NS`/`SOA` records every zone apex must have), so a bare apex domain
is never a valid CNAME target for the standard. A subdomain shows it
correctly instead:

```console
$ dig +noall +answer www.wikipedia.org CNAME
www.wikipedia.org.	86400	IN	CNAME	dyna.wikimedia.org.
```

`www.wikipedia.org` is an **alias** for `dyna.wikimedia.org` — clients
resolving `www.wikipedia.org` are transparently redirected (at the DNS
level) to look up `dyna.wikimedia.org` instead, which is where the
actual `A`/`AAAA` records live:

```console
$ dig www.wikipedia.org
;; ANSWER SECTION:
www.wikipedia.org.	86386	IN	CNAME	dyna.wikimedia.org.
dyna.wikimedia.org.	166	IN	A	185.15.58.224
```

One query for `www.wikipedia.org` returns both the alias and its
resolved address in a single response — the resolver follows the
`CNAME` automatically.
