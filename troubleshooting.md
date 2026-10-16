# Troubleshooting Guide

Real issues I hit while building this EVE-NG lab (HO + Branch, router-on-a-stick, Rocky Linux as DHCP/HTTP/NTP server, PAT + static NAT, GRE over IPsec).

Sections marked **Real issue** happened in this lab. Sections marked **General checks** are standard verification steps I use, not problems I hit here.

## Quick method

Work bottom-up and change one thing at a time:

1. L1/L2: `show ip interface brief`, `show interfaces status`, `show vlan brief`, `show interfaces trunk`
2. L3: `show ip route`, ping the gateway, then the next hop
3. Services: DHCP, NAT, ACL hit counters
4. Compare a working device with a failing one and look for what is different (often just the source IP, MAC or VLAN)
5. Save (`wr`) only after testing from a clean state

## Topology reference

| Link | Network |
|---|---|
| HO-R2 ↔ ISP-R | 200.1.1.0/28 (ISP-R .1, HO-R2 .2) |
| HO-R2 ↔ BO-R2 | 10.10.100.0/30 |
| GRE tunnel | 192.168.1.0/30 |
| HO VLANs | 10 HR, 20 IT, 30 FIN, 40 Servers (10.10.x.0/24) |
| Branch VLANs | 50 Sales, 60 Support |
| Linux server | 10.10.40.10 (DHCP, HTTP, NTP) |

---

## 1. VLANs + Trunk, Inter-VLAN (General checks)

```
show vlan brief
show interfaces trunk
show ip interface brief
show run interface e0/2.20
```

- Access port in the right VLAN and VLAN exists on the switch
- Trunk allows the VLAN; subinterface has the correct `encapsulation dot1Q <id>` and IP
- Ping order: PC → gateway → another VLAN's gateway → host in another VLAN

## 2. STP, EtherChannel (General checks)

```
show spanning-tree
show spanning-tree vlan 10
show etherchannel summary
```

- Confirm the expected root bridge and which ports are blocking
- EtherChannel members must match (speed, duplex, VLANs, mode); look for `(P)` flags in the summary

**Duplex mismatch log** (seen in this lab): `%CDP-4-DUPLEX_MISMATCH` between HO-R2 e0/2 and HO-SW1 e0/0. On real hardware set the same duplex on both ends.

## 3. OSPF (General checks)

```
show ip ospf neighbor
show ip route ospf
show ip ospf interface brief
```

- Neighbors must be `FULL`; check network statements, area, and passive interfaces

---

## 4. ACL (Real issue)

**Goal:** HR can reach only the web server (10.10.40.10:80).

**Symptom:** HR could not get DHCP or ping its own gateway. Interfaces and VLANs were fine. It worked after I removed the ACL.

**Causes found**

| Cause | Detail |
|---|---|
| `deny ip any any` blocks everything not permitted | DHCP, gateway ping and other traffic were dropped |
| DHCP source is 0.0.0.0 | A rule matching only `host 10.10.10.100` never matches a DHCP Discover |
| Numbered ACL appends at the bottom | Old `deny ip any any` stayed on line 20 and new permits landed below it (0 matches) |

**Fix**

```
no access-list 100
access-list 100 permit udp any eq 68 any eq 67
access-list 100 permit icmp 10.10.10.0 0.0.0.255 host 10.10.10.1
access-list 100 permit tcp 10.10.10.0 0.0.0.255 host 10.10.40.10 eq 80
access-list 100 deny ip any any
!
interface e0/2.10
 ip access-group 100 in
```

**Verify:** `show access-lists 100` (order and hit counters), `show ip interface e0/2.10 | include access list`

**Direction (in/out) is from the router's point of view**

- `in` = packet entering the router through that interface
- `out` = packet leaving the router through that interface
- Blocking HR → IT with standard ACL 10 (`deny host 10.10.10.100`, `permit any`): on `e0/2.20` **in** had no effect (HR traffic never arrives from that VLAN); **out** worked.
- Standard ACL: place near the destination. Extended ACL: place near the source.

**Lessons**

- Test what you block, not only what you allow (DHCP, gateway ping, fresh client)
- Check `show access-lists` after every change
- Keep a rollback command ready (`no ip access-group 100 in`)

## 5. DHCP (Real issue, related to ACL)

Linux (10.10.40.10) is the DHCP server for all VLANs, so every client subinterface needs a relay.

```
interface e0/2.10
 ip helper-address 10.10.40.10
```

Checks:

```
show run interface e0/2.20           ! helper-address present?
sudo systemctl status dhcpd          ! on Linux
sudo tail -20 /var/log/messages      ! look for DHCPDISCOVER
```

- Scope for each VLAN subnet must exist in `dhcpd.conf` with `option routers`
- If an ACL is on the client interface, permit `udp any eq 68 any eq 67`
- VPCS: `ip dhcp -r` then `ip dhcp`

## 6. NAT: PAT + Static (Real issue)

**Setup:** PAT on HO-R2 (`ip nat inside source list 2 interface e0/0 overload`) plus static NAT for Linux (`10.10.40.10 → 200.1.1.10`).

**Symptom:** With PAT only, every device (including Linux) could ping 8.8.8.8. After adding static NAT, **only Linux** lost internet and could not ping 200.1.1.1.

**Cause:** The HO-R2/ISP-R link was a `/30`, so only .1 and .2 were in the subnet. PAT traffic used 200.1.1.2 (in the subnet). Static NAT used 200.1.1.10 (outside the subnet), so ISP-R had no return path for it. Nothing was configured on ISP-R.

**Fix:** change the mask on both ends of the link to `/28`:

```
! ISP-R
interface e0/1
 ip address 200.1.1.1 255.255.255.240
! HO-R2
interface e0/0
 ip address 200.1.1.2 255.255.255.240
```

Block size = 256 − 240 = 16, so .1 to .14 are usable and .1, .2, .10 share one subnet. The mask must match on both ends.

**Verify**

```
show ip nat translations
clear ip nat translation *
ISP-R# telnet 200.1.1.10 80      ! "Open" = inbound static NAT + web server + return path OK
```

Other NAT checks: `ip nat inside` on the VLAN subinterfaces, `ip nat outside` on e0/0, the NAT ACL covers 10.10.0.0/16, and the default route points to 200.1.1.1.

**Lesson:** when only one device breaks, compare what is different about its traffic. Here it was only the source IP.

## 7. NTP (General checks)

```
show ntp status
show ntp associations
show clock
```

- Clients point to 10.10.40.10; check ACLs allow UDP 123 and the server firewall allows NTP
- On Rocky: `sudo systemctl status chronyd`, `chronyc sources`

---

## 8. GRE + IPsec (Real issue)

### 8.1 Tunnel "up/up" but nothing works

- GRE had no keepalive (`Keepalive not set`), so the tunnel shows up/up regardless of the far end
- Real test: `ping 192.168.1.2 source tunnel0`, not interface status
- `Last input never, output never` on `show interfaces tunnel0` means no traffic has crossed

### 8.2 IPsec not forming

**Symptom:** `show crypto isakmp sa` empty, `#pkts encaps: 0`, `current outbound spi: 0x0`.

Checked and found correct: underlay ping, key, peer, transform-set, mirrored ACL 101, crypto map applied to the physical interface.

**What changed when it worked:** I rebuilt the config with `group 5` instead of `group 12`. I never captured the debug output, so `group 12` is the most likely cause but **not proven**.

Working config pattern (HO-R2 shown, mirror on BO-R2):

```
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key 1234 address 10.10.100.2
crypto ipsec transform-set TS esp-aes 256 esp-sha-hmac
access-list 101 permit gre host 10.10.100.1 host 10.10.100.2
crypto map CMAP 10 ipsec-isakmp
 set peer 10.10.100.2
 set transform-set TS
 match address 101
interface e0/1
 crypto map CMAP
```

Notes:

- Crypto map goes on the **physical** interface, not the tunnel
- ACL matches **GRE** (protocol 47) between the tunnel endpoints
- Lab values (`group 5`, `sha`, key `1234`) are for lab only; production should use stronger settings (e.g. group 14+, sha256, strong key)

**Verify**

```
show crypto isakmp sa            ! QM_IDLE / ACTIVE
show crypto ipsec sa             ! encaps/decaps increasing, send/recv errors 0
clear crypto sa counters         ! then ping and re-check
debug crypto isakmp              ! lab only; undebug all afterwards
```

A small encaps/decaps difference while pinging is normal (packets in flight). Alarm if decaps stays 0 or the gap keeps growing.

### 8.3 LAN traffic not using the tunnel

**Symptom:** `trace 10.10.50.100` showed hop 2 = `10.10.100.2` (direct link) and IPsec counters did not move with LAN pings.

**Cause:** the direct link 10.10.100.0/30 was already in OSPF, and 192.168.1.0/30 was not routed anywhere, so OSPF kept using the direct link.

**Fix used (static routes, AD 1 beats OSPF AD 110):**

```
! HO-R2
ip route 10.10.50.0 255.255.255.0 192.168.1.2
ip route 10.10.60.0 255.255.255.0 192.168.1.2
! BO-R2
ip route 10.10.10.0 255.255.255.0 192.168.1.1
ip route 10.10.20.0 255.255.255.0 192.168.1.1
ip route 10.10.30.0 255.255.255.0 192.168.1.1
ip route 10.10.40.0 255.255.255.0 192.168.1.1
```

**Proof it works:** trace hop 2 became `192.168.1.2`, and `encaps`/`decaps` increased with every ping.

**Caveat:** static routes plus no tunnel keepalive means traffic is black-holed if the tunnel dies.

**Lab vs production:** here the underlay is a direct link that is also in OSPF. In production the tunnel rides over the internet and the underlay is not in OSPF.

### 8.4 Removing the VPN cleanly

Order: routes → crypto → GRE.

```
no ip route 10.10.50.0 255.255.255.0 192.168.1.2          ! and the others
interface e0/1
 no crypto map CMAP
no crypto map CMAP 10
no access-list 101
no crypto ipsec transform-set TS
no crypto isakmp key 1234 address 10.10.100.2
no crypto isakmp policy 10
clear crypto sa
clear crypto isakmp
no interface tunnel 0
```

Also remove any `ip ospf cost` you set on tunnel0 / e0/1. Verify with `trace`, then `wr`.

---

## 9. SSH (Real issue)

- **VPCS has no SSH client.** Use a Linux node, a router (`ssh -l admin <ip>`), or PuTTY from a real PC.
- Enable SSH on the router:

```
ip domain-name lab.local
crypto key generate rsa modulus 2048
username admin privilege 15 secret <password>
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
```

- **HR VLAN ACL blocks SSH.** The HR ACL only permits DHCP, gateway ping and HTTP, so TCP 22 needs its own permit, or SSH from a management VLAN instead.
- Restrict VTY with `access-class` to a management subnet.

## 10. Port Security (Real issue)

**Config used on HO-SW1 e0/3 (IT, VLAN 20):**

```
interface e0/3
 switchport mode access
 switchport access vlan 20
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
```

**Issues and fixes**

| Issue | Cause / fix |
|---|---|
| Port went `err-disabled` after violation | `shutdown` then `no shutdown` (both are needed). Remove the cause first or it re-triggers |
| `clear port-security` not available | Not present in this IOL image (`clear p?` showed only pagp, parser, policy-map). Use config commands instead |
| `no ... sticky` did not remove the old MAC | The learned MAC stays as `switchport port-security mac-address sticky <MAC>` in the config. Remove it explicitly with that MAC |
| `clear` rejected in config mode | `clear` is an EXEC command: use `end` first or `do clear ...` |
| New device also pinged fine | After the sticky MAC was deleted, the new device became the first learned MAC. Violation only happens when a *second, different* MAC appears after one is learned |
| Wrong port configured | Always run `show run interface e0/3` after entering the interface |
| New device had no connectivity while old sticky MAC existed | Expected behavior: only the learned MAC is allowed |

**Clean reset**

```
interface e0/3
 shutdown
 no switchport port-security mac-address sticky <MAC>
 no switchport port-security mac-address sticky
 no switchport port-security violation
 no switchport port-security maximum
 no switchport port-security
 no shutdown
```

**Test sequence**

1. Connect the correct PC, apply port security, ping the gateway so the MAC is learned
2. `show port-security address` shows the correct MAC as `SecureSticky`
3. Replace with a device with a different MAC and ping
4. `show interface status err-disabled` and `show port-security interface e0/3`: expect `Secure-shutdown`, violation count 1
5. Recover: reconnect the correct PC, `shutdown`, `no shutdown`
6. Save only after the port is healthy

**Verify**

```
show port-security
show port-security interface e0/3
show port-security address
show mac address-table interface e0/3
show interface e0/3                  ! "0 packets input" = frames not reaching the switch
```

If the PC still gets no DHCP after the port is up, check the VPCS MAC against the sticky MAC, then `show interface e0/3`, VLAN membership, and the relay.

---

## Lessons learned

1. `up/up` does not mean working. Verify with ping, counters and traces.
2. Check ACL order and hit counters after every change.
3. "Test, then save." Save only after testing from a clean state, and reload to confirm it survives.
4. Compare a working device with a failing one to find the difference.
5. Confirm you are in the right interface before typing commands.
6. A lab with a direct underlay is not production design; note the differences honestly.
