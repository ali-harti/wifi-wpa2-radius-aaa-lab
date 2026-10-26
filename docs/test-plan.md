# Test plan and evidence

Method: test each layer before moving to the next (cabling, addressing, routing, services, Wi-Fi, security). TTL values give a quick consistency check, since each router crossed removes one.

## Validation checks

| # | Test | Method | Observed result | Status |
|---|---|---|---|---|
| 1 | Router interfaces | `show ip interface brief` on R1 and R2 | Gi0/0 and Gi0/1 (R1), Gi0/0 and Gi0/2 (R2) up/up once both routers are configured | Pass |
| 2 | OSPF adjacency | `show ip ospf neighbor` | FULL on both routers (R2 is DR, R1 is BDR) | Pass |
| 3 | Routing through the core | PC0 pings 212.217.1.2, .3 and .4 | Replies with TTL 126. The first packet of each series times out because of ARP, which is normal | Pass |
| 4 | Remote server | PC0 pings 212.217.3.2 | 4 of 4 replies | Pass |
| 5 | DNS resolution | `ping intranet.com`, `ping internet.com` | Resolve to 212.217.1.4 and 212.217.3.2 | Pass |
| 6 | Web from PC0 | Web Browser | `intranet.com` shows "Intranet.com", `internet.com` shows "Internet.com" | Pass |
| 7 | DHCP on the WRTs | Desktop, IP Configuration | Smartphone0 192.168.1.100, Smartphone1 .101, Smartphone2 192.168.2.100, Smartphone3 .101. Correct gateways, DNS 212.217.1.2 | Pass |
| 8 | Smartphone to infrastructure | Ping series from one phone on each WRT | WRT gateway (TTL 255), 212.217.1.1 (254), 212.217.1.4 (127), 212.217.3.2 and .3.3 (125). Names resolve | Pass |
| 9 | Web from smartphones | Web Browser on phones of both WRTs | Both sites load in every scenario | Pass |
| 10 | Wrong credentials refused | Wrong PSK, then wrong AAA password | Wireless link dropped, restored after correction | Pass |
| 11 | Phones on the same WRT | Ping Smartphone0 to 1, then 2 to 3 | Works | Pass |
| 12 | Phones on different WRTs | Ping Smartphone0 to Smartphone2 | Fails (expected, NAT) | Pass |
| 13 | Wired hosts to smartphones | Ping PC0 to a smartphone | Fails (expected, NAT) | Pass |

Checks 12 and 13 are negative tests: the failure is the expected and desired outcome.

## Security tests per scenario

| Scenario | Test | Result |
|---|---|---|
| Open | Connectivity baseline from all four phones | Works, nothing to authenticate |
| WPA2-PSK | Wrong passphrase on Smartphone0 | Its link dropped, the other three phones stayed connected. After restoring the correct passphrase it reconnected and kept 192.168.1.100 |
| WPA2 + RADIUS | Valid credentials | Phones associated, received 192.168.x.x addresses, loaded both sites |
| WPA2 + RADIUS | Wrong password on one phone | Its link dropped, recovered once the correct password was restored |

## Traffic matrix (NAT)

| Traffic | Result | Reason |
|---|---|---|
| Smartphone to smartphone, same WRT | Works | Same private LAN |
| Smartphone to servers and PC0 | Works | The WRT translates the address, OSPF routes the packet |
| Servers or PC0 to a smartphone | Fails | No NAT entry for a connection started from outside |
| Smartphone to smartphone, different WRT | Fails | Each private network sits behind its own NAT, and 192.168.1.0/24 and 192.168.2.0/24 are not advertised in OSPF |

If full connectivity were required, the WRTs would have to become plain access points (connected by a LAN port, DHCP off) with the phones taking addresses from a DHCP pool on a router or server in the routed network. That is a different design, not needed here.

## Troubleshooting log

| Symptom | Cause | Fix |
|---|---|---|
| WRT GUI text truncated or overlapping | Windows display scaling above 100% in the embedded web page | Use the Config tab, or set Packet Tracer's high-DPI behaviour to Application |
| Phones attached to the wrong WRT after SSID changes | Phones still used the default SSID | Set the exact SSID on each phone and keep the two groups apart on the workspace |
| R1 Gi0/0 shows up/down | R2 not configured yet or its interface shut down | Configure R2 and run `no shutdown` on its Gi0/0 |
| `ping intranet.com` returns "could not find host" | DNS service was Off on Server1-DNS | Services, DNS, On |
| `intranet.com` opened the Internet.com page | The intranet.com record pointed to 212.217.3.2 | Correct the record to 212.217.1.4, clear the browser cache if needed |
| Phones on 169.254.x.x or "DHCP request failed" | DHCP start address of the WRT was the router address (x.x.x.1) | Set the last octet to 100 in GUI, Setup, Save Settings, renew DHCP |
| Phones have an address but no DNS | Static DNS 1 on the WRT was 0.0.0.0 | Enter 212.217.1.2 in the WRT's DHCP section |
| First ping of a series times out | ARP resolution | Normal, following packets succeed |
| Servers and PC0 cannot reach the phones | NAT on the WRTs | Expected behaviour |
| Pasting several commands in a phone prompt ran only some | The Packet Tracer prompt handles multi-line paste badly | Run commands one by one or in small groups |

## Performance notes from the report

| Factor | In this lab | Good practice in production |
|---|---|---|
| Wi-Fi channel planning | Both WRTs ended on channel 1 (no visible effect in the simulator) | Neighbouring access points on non-overlapping 2.4 GHz channels (1, 6, 11), or 5 GHz where possible |
| Encryption mode | AES in both secured scenarios | WPA2 with AES (CCMP) is robust and hardware-accelerated. Avoid WEP and TKIP |
| Authentication overhead | RADIUS only used when a phone connects | No per-packet cost, but the AAA server must be reachable and fast. Use two servers for resilience |
| Client density | 2 phones per WRT, 50-address pool | Size access points for concurrent clients, add access points or a controller as users grow |
| Routing control traffic | Passive interfaces on LAN ports, /30 on the core link | Limit OSPF messages to where routers exist |
| Name resolution | Local DNS in the server zone | A nearby caching DNS, at least two servers |
| Resilience | Single router path, single AAA and DNS | Redundant routers (HSRP or VRRP), redundant links, duplicated critical servers |
