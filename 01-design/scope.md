# Project Scope: Segmented Enterprise Network + Monitoring Lab

**Project name:** Segmented Network Lab
**Owner:** Cyrus Kivuva Musau
**Start date:** 29 Oct 2026
**Target finish:** 8 weeks from start
**Version:** 1.0

---

## 1. Goal

Design and build a segmented network for a fictional company inside a virtual lab, protect it with a default-deny firewall, and add centralized monitoring, alerting and logging so that failures and suspicious activity are detected quickly.

**In plain words:** split the network into locked zones, put a guard on every door, and install cameras and alarms.

## 2. Objectives

1. Build six separate network segments, all routed through one firewall.
2. Enforce least-privilege firewall rules (default deny between segments).
3. Provide core services: DNS, DHCP, file sharing and a public-facing web server.
4. Monitor every device with dashboards and alerts.
5. Collect logs centrally and detect basic security events.
6. Prove everything works with documented tests.

## 3. Network Segments

| VLAN / Zone | Name | Subnet | Gateway (firewall) | Purpose |
|---|---|---|---|---|
| 10 | Management | 10.10.10.0/24 | 10.10.10.1 | Admin access to firewall and servers |
| 20 | Servers | 10.10.20.0/24 | 10.10.20.1 | Internal DNS and file share |
| 30 | Users | 10.10.30.0/24 | 10.10.30.1 | Employee workstations |
| 40 | DMZ | 10.10.40.0/24 | 10.10.40.1 | Public-facing web server |
| 50 | Monitoring | 10.10.50.0/24 | 10.10.50.1 | Zabbix, central syslog |
| 99 | Guest | 10.10.99.0/24 | 10.10.99.1 | Internet-only visitors |

*Each zone is a separate virtual network in KVM, with one firewall network card in each zone. This behaves like VLANs for routing and firewall purposes.*

## 4. Lab Environment

**Host:** Ubuntu PC, 8 GB RAM, KVM + virt-manager
**Budget for lab VMs:** about 5.5 GB RAM

| VM | Zone | IP | RAM | Role |
|---|---|---|---|---|
| fw-opnsense | All zones + WAN | .1 in every zone | 1 GB | Firewall, router, NAT, DHCP |
| core-srv | Servers | 10.10.20.10 | 1 GB | DNS (acme.lab), file share, NTP |
| web-dmz | DMZ | 10.10.40.10 | 512 MB | nginx with HTTPS |
| monitor-srv | Monitoring | 10.10.50.10 | 2 GB | Zabbix, Grafana (optional), rsyslog |
| client-user | Users | DHCP | 512 MB | Employee test machine |
| client-guest | Guest | DHCP | 512 MB | Visitor test machine |

The Ubuntu host itself acts as the **admin workstation** on the Management network (10.10.10.2).

## 5. In Scope

- Segmentation and inter-zone routing through OPNsense
- Firewall policy with logging of denied traffic
- DNS, DHCP, file sharing, web server with HTTPS, time sync (NTP)
- Monitoring: up/down, CPU, RAM, disk, interface traffic, service checks
- Alerting by email or Telegram
- Central logging via syslog
- Security tests: port scans, failed logins, outages
- Backups and restore test of firewall config and one VM
- Full documentation and a short demo

## 6. Out of Scope (future improvements)

- Windows Server and Active Directory (needs more RAM)
- Wazuh SIEM (needs about 4 GB; may be added later)
- Physical switches, real VLAN tagging, 802.1X
- VPN, high availability, IDS/IPS (Suricata)
- Any connection to real production networks

## 7. Success Criteria

Each criterion is pass/fail and must be backed by evidence (screenshot or log) in `04-test-results/`.

| # | Criterion | How it will be tested |
|---|---|---|
| 1 | Six segments exist and all traffic between them passes through the firewall | Traceroute and ping tests between zones |
| 2 | Guest cannot reach any internal segment | Ping and connection attempts from client-guest |
| 3 | DMZ cannot reach the Servers segment | Connection attempts from web-dmz to core-srv |
| 4 | Users can reach only DNS, file share and the web server over HTTPS | Port tests from client-user |
| 5 | Management is reachable only from the admin workstation | Access attempts from other zones |
| 6 | Every device appears in the monitoring dashboard | Zabbix host list shows all VMs as available |
| 7 | Shutting down a device triggers an alert within 5 minutes | Timed shutdown test |
| 8 | Port scans and failed logins are logged and detected | nmap scan and repeated bad SSH logins |
| 9 | Firewall config and one VM can be restored from backup | Restore test |

## 8. Assumptions and Constraints

- Only 8 GB RAM, so VMs are command-line only and the two clients run only during testing.
- Free software only.
- Internet access is available for downloads and updates.
- The lab is isolated; nothing is exposed to the real internet.

## 9. Deliverables

| Folder | Content |
|---|---|
| 01-design | Scope, topology diagram, IP plan, firewall matrix |
| 02-configs | Exported firewall and server configs |
| 03-screenshots | Dashboards, alerts, rule sets |
| 04-test-results | Test table with pass/fail and evidence |
| 05-report | Final report, notes, lessons learned |
| issue-log.md | Problems, causes, fixes |

## 10. Timeline

| Week | Milestone |
|---|---|
| 1 | Design approved, tools installed, ISOs ready |
| 2 | KVM networks and OPNsense running |
| 3-4 | Segmentation and firewall rules tested |
| 4-5 | DNS, DHCP, file and web servers live |
| 5-7 | Monitoring, alerts and logging live |
| 7 | Security and failure tests completed |
| 8 | Documentation and demo ready |

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Running out of RAM | VMs crash or lab is slow | Run clients only when testing; add swap; light VMs |
| Misconfigured firewall locks me out | Lost access to OPNsense | Keep the Management zone rule first; snapshot before changes |
| Large downloads fail | Delays | Use `wget -c`, mirrors or torrents |
| Time overrun | Incomplete project | Cut optional items (Grafana, Wazuh) first |