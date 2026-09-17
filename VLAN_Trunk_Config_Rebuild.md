# Small Corporate Network — VLAN & Trunk Rebuild (Clean Slate)

IP addressing (Layer 3 / SVI addresses, DHCP scopes, routing) is **untouched** by this
rebuild — only VLAN database and switchport (trunk/access/voice) configuration is
being reset and reapplied. Paste each block into the matching device's CLI.

---

## VLAN Plan

| VLAN | Name       | Purpose                                             |
|------|------------|------------------------------------------------------|
| 10   | IT         | IT Department hosts                                   |
| 20   | Office     | General corporate data (office PCs, printer, laptops) |
| 30   | Servers    | Server Room                                            |
| 40   | Corp_WiFi  | Corporate Wi-Fi SSID                                   |
| 50   | Guest_WiFi | Guest Wi-Fi SSID                                       |
| 60   | Voice      | IP phones                                               |

Native VLAN on every trunk = **VLAN 1** (matches existing default config on the
access switches — this is what clears the `%CDP-4-NATIVE_VLAN_MISMATCH` errors).

---

## 1. Multilayer Switch0 (3560-24PS) — Core

```
enable
configure terminal
!
! --- wipe old VLANs (ignore any "not found" errors) ---
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
!
! --- recreate VLANs ---
vlan 10
 name IT
vlan 20
 name Office
vlan 30
 name Servers
vlan 40
 name Corp_WiFi
vlan 50
 name Guest_WiFi
vlan 60
 name Voice
exit
!
! --- Fa0/1: trunk to Switch0 (Server Room) ---
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 30
!
! --- Fa0/2: trunk to Switch1 (Guest Network) ---
interface FastEthernet0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 50
!
! --- Fa0/3: trunk to Switch2 (IT Department) ---
interface FastEthernet0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 10
!
! --- Fa0/4: trunk to Switch3 (Corporate Network) ---
interface FastEthernet0/4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 20,40
!
! --- Fa0/5: trunk to Switch4 (Office Users) ---
interface FastEthernet0/5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 20,60
!
end
write memory
```

> Note: on the 3560 the ports are switchports by default, but if any of these were
> previously set with `no switchport` (routed port) you'll need `switchport` first
> before the trunk commands will take.

---

## 2. Switch0 — Server Room

```
enable
configure terminal
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
vlan 30
 name Servers
exit
!
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 30
!
interface range FastEthernet0/2 - 5
 switchport mode access
 switchport access vlan 30
!
end
write memory
```
(Fa0/2–0/5 → DHCP/DNS server, File server, Monitoring server, Backup server)

---

## 3. Switch1 — Guest Network

```
enable
configure terminal
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
vlan 50
 name Guest_WiFi
exit
!
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 50
!
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 50
!
end
write memory
```
(Fa0/2 → Guest Wi-Fi Access Point)

---

## 4. Switch2 — IT Department

```
enable
configure terminal
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
vlan 10
 name IT
exit
!
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 10
!
interface range FastEthernet0/2 - 4
 switchport mode access
 switchport access vlan 10
!
end
write memory
```
(Fa0/2 IT Admin PC · Fa0/3 Developer laptop · Fa0/4 IT Printer)

---

## 5. Switch3 — Corporate Network

```
enable
configure terminal
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
vlan 20
 name Office
vlan 40
 name Corp_WiFi
exit
!
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 20,40
!
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 40
!
interface range FastEthernet0/3 - 4
 switchport mode access
 switchport access vlan 20
!
end
write memory
```
(Fa0/2 → Corporate Wi-Fi AP · Fa0/3 Employee Laptop 1 · Fa0/4 Employee Laptop 2)

---

## 6. Switch4 — Office Users

```
enable
configure terminal
no vlan 10
no vlan 20
no vlan 30
no vlan 40
no vlan 50
no vlan 60
vlan 20
 name Office
vlan 60
 name Voice
exit
!
interface FastEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 1
 switchport trunk allowed vlan 20,60
!
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 switchport voice vlan 60
!
interface range FastEthernet0/3 - 4
 switchport mode access
 switchport access vlan 20
!
interface FastEthernet0/5
 switchport mode access
 switchport access vlan 20
!
end
write memory
```
(Fa0/2 → IP Phone0 + PC behind phone, data VLAN 20 / voice VLAN 60 · Fa0/3 PC User1 ·
Fa0/4 PC User2 · Fa0/5 Office Printer)

---

## After applying

On the core switch, confirm everything's clean:
```
show vlan brief
show interfaces trunk
show cdp neighbors
```
The `NATIVE_VLAN_MISMATCH` messages should stop appearing once every trunk's native
VLAN matches on both ends.
