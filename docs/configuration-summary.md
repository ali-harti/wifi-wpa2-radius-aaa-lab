# Configuration summary

Everything below comes from the report (sections 4 to 7 and Annex C). The two routers have text configs in [`../configs/`](../configs/). The WRT300N routers, servers and smartphones are configured through the Packet Tracer interface, so their settings are listed here as parameters. Lab credentials are shown as placeholders on purpose.

Packet Tracer interface labels (tabs, fields, buttons) are kept in English, as in the report, to make them easy to find on screen.

## Cabling

All cables are straight-through except the router link (crossover). The WRTs connect through their **Internet** port, not a LAN port.

| From | To | Cable |
|---|---|---|
| WRT1, Internet | Switch1, Fa0/5 | Straight |
| WRT2, Internet | Switch1, Fa0/6 | Straight |
| Server AAA, Fa0 | Switch1, Fa0/2 | Straight |
| Server1-DNS, Fa0 | Switch1, Fa0/3 | Straight |
| Server Intranet.com, Fa0 | Switch1, Fa0/4 | Straight |
| Switch1, Fa0/1 | Router1, Gi0/1 | Straight |
| Router1, Gi0/0 | Router2, Gi0/0 | Crossover |
| Router2, Gi0/2 | Switch2, Fa0/1 | Straight |
| PC0, Fa0 | Switch2, Fa0/2 | Straight |
| Server 2 Internet.com, Fa0 | Switch2, Fa0/3 | Straight |

Port numbers were read from the Packet Tracer workspace. If a rebuild uses other ports, adapt the interface names in the router commands.

## Servers and PC0

For each device: Desktop, IP Configuration, Static, then the values from the addressing table in the README. The two remote-LAN devices use gateway 212.217.3.1. All devices use DNS 212.217.1.2.

**Server1-DNS** (Services, DNS): service **On** (records only answer when it is on), then two A records, clicking Add after each.

| Name | Type | Address |
|---|---|---|
| intranet.com | A record | 212.217.1.4 |
| internet.com | A record | 212.217.3.2 |

**Web servers** (Services, HTTP): keep HTTP enabled and edit `index.html` so each site is identifiable: text `Intranet.com` on Server Intranet.com, `Internet.com` on Server 2 Internet.com. Save, then Yes to overwrite the default page.

## Wireless routers (WRT1 and WRT2)

Use the **Config** tab where possible, and **GUI** only for the DHCP section (the GUI page can look truncated when Windows display scaling is above 100%).

| Parameter | WRT1 | WRT2 | Where |
|---|---|---|---|
| Internet connection type | Static | Static | Config, Internet |
| Internet IP / mask | 212.217.1.10 / 255.255.255.0 | 212.217.1.11 / 255.255.255.0 | Config, Internet |
| Default gateway | 212.217.1.1 | 212.217.1.1 | Config, Internet |
| DNS server | 212.217.1.2 | 212.217.1.2 | Config, Internet |
| LAN IP / mask | 192.168.1.1 / 255.255.255.0 | 192.168.2.1 / 255.255.255.0 | Config, LAN |
| DHCP server | Enabled | Enabled | GUI, Setup |
| DHCP start address | 192.168.1.100 | 192.168.2.100 | GUI, Setup |
| Maximum users | 50 (range .100 to .149) | 50 (range .100 to .149) | GUI, Setup |
| Static DNS 1 (given to clients) | 212.217.1.2 | 212.217.1.2 | GUI, Setup |
| SSID | wifi1 | wifi2 | Config, Wireless |

Two points that decide whether it works:
- **The DHCP start address must not be the router's own address.** The default can end up as 192.168.x.1, which makes DHCP fail and sends phones to 169.254.x.x. Set the last octet to 100 and click Save Settings in the GUI.
- **The DNS given to clients comes from the DHCP section.** If Static DNS 1 stays at 0.0.0.0, phones get an address but no DNS and names do not resolve.

## Smartphones

Config, Wireless0: set the SSID (`wifi1` for Smartphone0 and 1, `wifi2` for Smartphone2 and 3) and the authentication settings of the scenario under test. Keep IP configuration on DHCP. After any change, renew the lease: Desktop, IP Configuration, Static, then DHCP.

## The three Wi-Fi scenarios

The scenarios are applied one after another on the same WRTs and phones, and each replaces the previous settings. Save a separate copy of the file before each change.

### Scenario 1: open Wi-Fi

| Device | Setting |
|---|---|
| WRT and smartphones | Authentication **Disabled** |

Anyone in range can connect and the radio traffic is not encrypted. Role in the lab: reference point.

### Scenario 2: WPA2-PSK

| Device | Setting |
|---|---|
| WRT (Config, Wireless) | Authentication WPA2-PSK, PSK Pass Phrase `<PSK>`, Encryption AES, SSID wifi1 or wifi2 |
| Smartphone (Config, Wireless0) | Same SSID, WPA2-PSK, same passphrase, AES. No username |

Control happens on the WRT itself. No server is involved.

### Scenario 3: WPA2 with RADIUS (AAA)

**Server AAA** (Services, AAA):
1. Service **On** (RADIUS port shown by Packet Tracer: 1645)
2. Network Configuration: add two RADIUS clients, clicking Add after each. WRT1 with client IP 212.217.1.10 and WRT2 with client IP 212.217.1.11, server type Radius, shared secret `<RADIUS_SECRET>` for both
3. User Setup: add user `user1` with password `<USER_PASSWORD>`, then Add

| Device | Setting |
|---|---|
| WRT (Config, Wireless) | Authentication **WPA2** (the enterprise option, not WPA2-PSK), RADIUS server IP 212.217.1.3, shared secret `<RADIUS_SECRET>`, Encryption AES |
| Smartphone | SSID, Authentication WPA2, User ID `user1`, Password `<USER_PASSWORD>`, Encryption AES |

The shared secret and the client IPs must match on both sides. The client IP in the AAA table is the WRT's **Internet** address, because that is where the RADIUS requests come from.

## Verification commands

Routers: `show ip interface brief`, `show ip ospf neighbor`, `show ip route`, `show running-config`, `write memory`.

Smartphone command prompt (Smartphone0 and 1; on Smartphone2 and 3 start with 192.168.2.1): `ping 192.168.1.1`, `ping 212.217.1.1`, `ping 212.217.1.4`, `ping 212.217.3.2`, `ping 212.217.3.3`, `ping intranet.com`, `ping internet.com`. Run them one by one or in small groups: the Packet Tracer prompt handles multi-line pasting badly.

Expected: `show ip ospf neighbor` lists the other router as FULL, `show ip route` on R1 contains an `O` route to 212.217.3.0/24, and every ping gets a reply (the very first packet may time out because of ARP).
