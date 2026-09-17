# Small Corporate Network — Full Build-Out

Builds on top of the VLAN/trunk rebuild already completed. This covers: IP
addressing, inter-VLAN routing, DHCP, WAN/router config with backup failover,
firewall (ASA 5506-X) rules, and wireless AP security.

---

## 1. IP Addressing Plan

Using 10.10.X.0/24 per VLAN — easy to remember, room to grow, private range.

| VLAN | Name       | Subnet          | Gateway (SVI) | Usable Hosts      |
|------|------------|-----------------|----------------|--------------------|
| 10   | IT         | 10.10.10.0/24   | 10.10.10.1     | .2 – .254          |
| 20   | Office     | 10.10.20.0/24   | 10.10.20.1     | .2 – .254          |
| 30   | Servers    | 10.10.30.0/24   | 10.10.30.1     | .2 – .254          |
| 40   | Corp_WiFi  | 10.10.40.0/24   | 10.10.40.1     | .2 – .254          |
| 50   | Guest_WiFi | 10.10.50.0/24   | 10.10.50.1     | .2 – .254          |
| 60   | Voice      | 10.10.60.0/24   | 10.10.60.1     | .2 – .254          |

**Server static IPs (VLAN 30):**
| Server       | IP           |
|--------------|--------------|
| DHCP/DNS     | 10.10.30.10  |
| File Server  | 10.10.30.11  |
| Monitoring   | 10.10.30.12  |
| Backup       | 10.10.30.13  |

**Transit / infrastructure links (point-to-point, /30):**
| Link                              | Subnet             |
|------------------------------------|--------------------|
| Core Switch ↔ Firewall (inside)    | 10.10.100.0/30     |
| Firewall (outside-primary) ↔ Router1 | 192.168.100.0/30 |
| Firewall (outside-backup) ↔ Backup Router | 192.168.101.0/30 |
| Router1 ↔ Internet cloud           | 203.0.113.0/30 *(placeholder — use whatever the cloud assigns in PT)* |
| Backup Router ↔ Internet cloud     | 203.0.114.0/30 *(placeholder)* |

---

## 2. Inter-VLAN Routing (Multilayer Switch0 — core)

You need a **routed port** from the core switch down to the firewall — use
Gig0/1 (unused so far).

```
enable
configure terminal
ip routing
!
interface Vlan10
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.10.30.10
!
interface Vlan20
 ip address 10.10.20.1 255.255.255.0
 ip helper-address 10.10.30.10
!
interface Vlan30
 ip address 10.10.30.1 255.255.255.0
!
interface Vlan40
 ip address 10.10.40.1 255.255.255.0
 ip helper-address 10.10.30.10
!
interface Vlan50
 ip address 10.10.50.1 255.255.255.0
 ip helper-address 10.10.30.10
!
interface Vlan60
 ip address 10.10.60.1 255.255.255.0
 ip helper-address 10.10.30.10
!
interface GigabitEthernet0/1
 no switchport
 ip address 10.10.100.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.10.100.2
!
end
write memory
```

`ip helper-address` forwards DHCP broadcasts from each VLAN to the DHCP
server in VLAN 30 (since DHCP requests are broadcast and normally can't
cross a router/L3 boundary).

---

## 3. DHCP Server (on the DHCP/DNS Server-PT, 10.10.30.10)

Packet Tracer's server DHCP service is GUI-based (Desktop → Services → DHCP),
not CLI. Create one pool per VLAN:

| Pool Name | Network       | Mask            | Default GW  | DNS Server  | Start IP    |
|-----------|---------------|-----------------|-------------|-------------|-------------|
| IT        | 10.10.10.0    | 255.255.255.0   | 10.10.10.1  | 10.10.30.10 | 10.10.10.10 |
| Office    | 10.10.20.0    | 255.255.255.0   | 10.10.20.1  | 10.10.30.10 | 10.10.20.10 |
| Corp_WiFi | 10.10.40.0    | 255.255.255.0   | 10.10.40.1  | 10.10.30.10 | 10.10.40.10 |
| Guest_WiFi| 10.10.50.0    | 255.255.255.0   | 10.10.50.1  | 10.10.30.10 | 10.10.50.10 |
| Voice     | 10.10.60.0    | 255.255.255.0   | 10.10.60.1  | 10.10.30.10 | 10.10.60.10 |

Steps in PT: click the server → **Services** tab → **DHCP** → set "Service"
to **On** → add each pool above → **Save**. Leave VLAN 30 (Servers) on
static IPs only, no pool needed.

---

## 4. WAN Routers (Router1 = primary, Backup Router = failover)

### Router1 (primary path)
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 203.0.113.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 ip address 192.168.100.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 203.0.113.2
!
end
write memory
```

### Backup Router
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 203.0.114.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 ip address 192.168.101.1 255.255.255.252
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 203.0.114.2
!
end
write memory
```

> Replace the `203.0.113.x` / `203.0.114.x` addresses with whatever your
> Internet cloud object in Packet Tracer actually assigns/expects — those
> are placeholders representing "ISP-facing" addressing since PT's cloud
> doesn't simulate a real ISP.

---

## 5. Firewall (ASA 5506-X) — dual WAN + NAT + guest isolation

```
enable
configure terminal
!
interface GigabitEthernet1/1
 nameif outside-primary
 security-level 0
 ip address 192.168.100.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/2
 nameif outside-backup
 security-level 0
 ip address 192.168.101.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet1/3
 nameif inside
 security-level 100
 ip address 10.10.100.2 255.255.255.252
 no shutdown
!
! Primary route preferred (AD 1), backup route as floating (AD 200)
route outside-primary 0.0.0.0 0.0.0.0 192.168.100.1 1
route outside-backup 0.0.0.0 0.0.0.0 192.168.101.1 200
!
! NAT internal networks out to the Internet
object network INSIDE-NET
 subnet 10.10.0.0 255.255.0.0
nat (inside,outside-primary) dynamic interface
nat (inside,outside-backup) dynamic interface
!
! Guest Wi-Fi (VLAN 50) can reach the Internet but not any internal VLAN
access-list GUEST-ISOLATION extended deny ip 10.10.50.0 255.255.255.0 10.10.0.0 255.255.0.0
access-list GUEST-ISOLATION extended permit ip 10.10.50.0 255.255.255.0 any
access-group GUEST-ISOLATION in interface inside
!
end
write memory
```

This gives automatic failover: if the primary route's next hop stops
responding, the ASA falls back to the backup route (basic static-route
failover — for true SLA-monitored failover you'd add a `sla monitor` +
`track` object, which is a nice-to-have but not required for a working lab).

---

## 6. Wireless AP Security

**Corporate Wi-Fi AP (VLAN 40):**
- Security mode: **WPA2-PSK** (or WPA2-Enterprise if you want to show off
  RADIUS integration — more complex, optional)
- SSID: `Corp-WiFi` (or similar, not overly descriptive)
- Strong passphrase, not shared outside IT
- SSID broadcast: on (employees need to find it) — or off + manually
  configured on devices for slightly more obscurity

**Guest Wi-Fi AP (VLAN 50):**
- Security mode: **WPA2-PSK** with a separate, simpler passphrase you can
  rotate/share easily with visitors
- Client isolation: **enabled** if the AP model supports it (prevents guest
  devices from seeing each other)
- Internal network access: blocked at the firewall via the
  `GUEST-ISOLATION` ACL above — Internet-only

In Packet Tracer, click the Access Point → **Config** tab → **Port 1
(802.11)** → set SSID, Authentication (WPA2-PSK), and Pass Phrase there.

---

## 7. Suggested Verification Checklist

- [ ] `show ip route` on core switch — all 6 VLAN subnets + default route present
- [ ] PC in each VLAN gets a correct DHCP lease (`ipconfig /all` on PC-PT)
- [ ] PC in VLAN 10 can ping a PC in VLAN 20 (inter-VLAN routing works)
- [ ] PC in VLAN 50 (Guest) **cannot** ping a PC in VLAN 10/20/30/40/60
- [ ] PC in VLAN 50 (Guest) **can** reach the Internet cloud
- [ ] IP Phone on VLAN 60 registers and gets its own IP separate from the PC behind it
- [ ] Failing Router1's link causes traffic to fail over to Backup Router (test by shutting Router1's outside interface)
