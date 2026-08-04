# Subnetting: 10.42.0.0/16 into 4 equal subnets

## 1. Manual calculation

### Starting point

- Network: `10.42.0.0/16`
- Mask: `255.255.0.0` = `11111111.11111111.00000000.00000000`
- Total addresses: 2^16 = 65536

### How many bits to borrow

To split into **4 equal subnets**, we need `4 = 2^2` subnets, so we
borrow **2 bits** from the host portion:

```
/16  ->  /18   (16 + 2 = 18)
```

New mask: `255.255.192.0`

```
11111111.11111111.11000000.00000000
                   ^^
                   borrowed bits (2 bits = 4 possible subnets)
```

Size of each subnet: 2^(32-18) = 2^14 = **16384 addresses**.

### The 2 borrowed bits live in the 3rd octet

The 3rd octet of the original network is `00000000` (0). The 2 leftmost
bits of that octet are now part of the network portion, so they can
take values `00`, `01`, `10`, `11` — i.e. the 3rd octet steps by
`2^(8-2) = 64` for each new subnet:

| Borrowed bits | 3rd octet (binary) | 3rd octet (decimal) |
| - | - | - |
| `00` | `00000000` | 0 |
| `01` | `01000000` | 64 |
| `10` | `10000000` | 128 |
| `11` | `11000000` | 192 |

### Resulting subnets (manual math)

For each subnet: network address (borrowed bits + host bits = 0),
broadcast address (borrowed bits + host bits = 1), usable range
(network + 1 to broadcast - 1), usable hosts = 2^14 - 2 (subtract
network and broadcast addresses).

| # | CIDR | Network address | Broadcast address | Usable host range | Usable hosts |
| - | - | - | - | - | - |
| 1 | `10.42.0.0/18`   | `10.42.0.0`   | `10.42.63.255`  | `10.42.0.1` – `10.42.63.254`   | 16382 |
| 2 | `10.42.64.0/18`  | `10.42.64.0`  | `10.42.127.255` | `10.42.64.1` – `10.42.127.254` | 16382 |
| 3 | `10.42.128.0/18` | `10.42.128.0` | `10.42.191.255` | `10.42.128.1` – `10.42.191.254`| 16382 |
| 4 | `10.42.192.0/18` | `10.42.192.0` | `10.42.255.255` | `10.42.192.1` – `10.42.255.254`| 16382 |

Sanity check: `4 subnets x 16384 addresses = 65536 = 2^16`, matches
the original `/16` block exactly, with no overlap and no gap.

## 2. Verification with `ipcalc`

Command used for each subnet: `ipcalc <cidr>`

```console
$ ipcalc 10.42.0.0/18
Address:   10.42.0.0            00001010.00101010.00 000000.00000000
Netmask:   255.255.192.0 = 18   11111111.11111111.11 000000.00000000
Wildcard:  0.0.63.255           00000000.00000000.00 111111.11111111
=>
Network:   10.42.0.0/18         00001010.00101010.00 000000.00000000
HostMin:   10.42.0.1            00001010.00101010.00 000000.00000001
HostMax:   10.42.63.254         00001010.00101010.00 111111.11111110
Broadcast: 10.42.63.255         00001010.00101010.00 111111.11111111
Hosts/Net: 16382                 Class A, Private Internet
```

```console
$ ipcalc 10.42.64.0/18
Address:   10.42.64.0           00001010.00101010.01 000000.00000000
Netmask:   255.255.192.0 = 18   11111111.11111111.11 000000.00000000
Wildcard:  0.0.63.255           00000000.00000000.00 111111.11111111
=>
Network:   10.42.64.0/18        00001010.00101010.01 000000.00000000
HostMin:   10.42.64.1           00001010.00101010.01 000000.00000001
HostMax:   10.42.127.254        00001010.00101010.01 111111.11111110
Broadcast: 10.42.127.255        00001010.00101010.01 111111.11111111
Hosts/Net: 16382                 Class A, Private Internet
```

```console
$ ipcalc 10.42.128.0/18
Address:   10.42.128.0          00001010.00101010.10 000000.00000000
Netmask:   255.255.192.0 = 18   11111111.11111111.11 000000.00000000
Wildcard:  0.0.63.255           00000000.00000000.00 111111.11111111
=>
Network:   10.42.128.0/18       00001010.00101010.10 000000.00000000
HostMin:   10.42.128.1          00001010.00101010.10 000000.00000001
HostMax:   10.42.191.254        00001010.00101010.10 111111.11111110
Broadcast: 10.42.191.255        00001010.00101010.10 111111.11111111
Hosts/Net: 16382                 Class A, Private Internet
```

```console
$ ipcalc 10.42.192.0/18
Address:   10.42.192.0          00001010.00101010.11 000000.00000000
Netmask:   255.255.192.0 = 18   11111111.11111111.11 000000.00000000
Wildcard:  0.0.63.255           00000000.00000000.00 111111.11111111
=>
Network:   10.42.192.0/18       00001010.00101010.11 000000.00000000
HostMin:   10.42.192.1          00001010.00101010.11 000000.00000001
HostMax:   10.42.255.254        00001010.00101010.11 111111.11111110
Broadcast: 10.42.255.255        00001010.00101010.11 111111.11111111
Hosts/Net: 16382                 Class A, Private Internet
```

## 3. Conclusion

`ipcalc`'s output matches the manual calculation exactly for all 4
subnets: same network address, same broadcast address, same
`HostMin`/`HostMax` (usable range), and same host count (16382 per
subnet).

| Check | Manual | ipcalc | Match |
| - | - | - | - |
| Mask | `/18` = `255.255.192.0` | `/18` = `255.255.192.0` | Yes |
| Subnet 1 network/broadcast | `10.42.0.0` / `10.42.63.255` | `10.42.0.0` / `10.42.63.255` | Yes |
| Subnet 2 network/broadcast | `10.42.64.0` / `10.42.127.255` | `10.42.64.0` / `10.42.127.255` | Yes |
| Subnet 3 network/broadcast | `10.42.128.0` / `10.42.191.255` | `10.42.128.0` / `10.42.191.255` | Yes |
| Subnet 4 network/broadcast | `10.42.192.0` / `10.42.255.255` | `10.42.192.0` / `10.42.255.255` | Yes |
| Usable hosts/subnet | 16382 | 16382 | Yes |
