# Multi-Site Enterprise Network Design & Implementation

## Project Overview
Designed and implemented a multi-site enterprise network for TechNova Pvt Ltd
(Head Office + Branch Office) using EVE-NG with real Cisco IOS.

## Platform
- EVE-NG Community Edition
- Cisco IOSv (Routers)
- Cisco IOSvL2 (Switches)
- Rocky Linux (Server)
- VPCS (Clients)
- Cloud0 NAT (Internet)

## Topology
- ISP-R: Internet simulation
- HO-R2: Head Office router
- BO-R2: Branch Office router
- HO-SW1, HO-SW2: Head Office switches
- BO-SW1: Branch Office switch
- PC1-PC3: HR, IT, FIN
- PC4-PC5: Sales, Support
- Linux: Server (VLAN 40)

## Features Implemented
- [x] VLANs (10, 20, 30, 40, 50, 60, 99)
- [x] 802.1Q Trunking
- [x] Inter-VLAN Routing (Router-on-a-Stick)
- [x] STP Root Bridge (Primary + Secondary)
- [x] EtherChannel (LACP)
- [x] Static Routing
- [x] OSPF Single Area
- [x] NAT (PAT + Static)
- [x] ACLs (Standard + Extended)
- [x] DHCP (Rocky Linux)
- [x] NTP (Rocky Linux)
- [x] GRE Tunnel
- [x] IPsec over GRE
- [x] SSH
- [x] Port Security

## VLAN Plan
| VLAN | Name | Subnet | Gateway |
|------|------|--------|---------|
| 10 | HR | 10.10.10.0/24 | 10.10.10.1 |
| 20 | IT | 10.10.20.0/24 | 10.10.20.1 |
| 30 | FIN | 10.10.30.0/24 | 10.10.30.1 |
| 40 | SERVERS | 10.10.40.0/24 | 10.10.40.1 |
| 50 | SALES | 10.10.50.0/24 | 10.10.50.1 |
| 60 | SUPPORT | 10.10.60.0/24 | 10.10.60.1 |
| 99 | MGMT | 10.10.99.0/24 | 10.10.99.1 |

## IP Schema
| Device | Interface | IP | Description |
|--------|-----------|-----|-------------|
| ISP-R | e0/0 | DHCP | Cloud |
| ISP-R | e0/1 | 200.1.1.1/30 | to HO-R2 |
| HO-R2 | e0/0 | 200.1.1.2/30 | to ISP-R |
| HO-R2 | e0/1 | 10.10.100.1/30 | to BO-R2 |
| HO-R2 | e0/2 | Trunk | to HO-SW1 |
| BO-R2 | e0/0 | 10.10.100.2/30 | to HO-R2 |
| BO-R2 | e0/1 | Trunk | to BO-SW1 |
| HO-R2 | Tunnel0 | 192.168.1.1/30 | GRE Tunnel |
| BO-R2 | Tunnel0 | 192.168.1.2/30 | GRE Tunnel |

## Verification
- ✅ Inter-VLAN ping successful
- ✅ Cross-site ping successful
- ✅ OSPF neighbor FULL
- ✅ STP Root Bridge elected
- ✅ EtherChannel LACP bundled
- ✅ NAT translations working
- ✅ GRE tunnel up
- ✅ IPsec SA ACTIVE
- ✅ SSH working
- ✅ Port Security configured

## Repository Structure