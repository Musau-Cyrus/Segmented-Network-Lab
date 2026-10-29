# Firewall Policy Matrix

**Principle:** default deny. Nothing crosses between zones unless a rule below allows it.
Replies to allowed connections are permitted automatically (the firewall is stateful).

## 1. Hosts and Aliases

| Alias name | Value | Meaning |
|---|---|---|
| ADMIN_HOST | 10.10.10.2 | The Ubuntu host (admin workstation) |
| CORE_SRV | 10.10.20.10 | DNS, file share, NTP |
| WEB_DMZ | 10.10.40.10 | Public web server |
| MON_SRV | 10.10.50.10 | Zabbix and syslog |
| RFC1918 | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | All private ranges (used to block "internal" traffic) |

## 2. Summary Matrix (who can reach whom)

Read as: **row = source, column = destination.**

| From \ To | Management | Servers | Users | DMZ | Monitoring | Guest | Internet |
|---|---|---|---|---|---|---|---|
| **Management** | n/a | Allow (admin ports) | Allow (ping, SSH) | Allow (SSH) | Allow (SSH, web UI) | Deny | Allow |
| **Servers** | Deny | n/a | Deny | Deny | Allow (agent, syslog) | Deny | Allow (updates) |
| **Users** | Deny | Allow (DNS, file share, NTP) | n/a | Allow (HTTPS only) | Deny | Deny | Allow (web) |
| **DMZ** | Deny | **Deny** | Deny | n/a | Allow (agent, syslog) | Deny | Allow (updates) |
| **Monitoring** | Allow (SNMP to firewall) | Allow (agent) | Allow (agent) | Allow (agent) | n/a | Deny | Allow (updates, alerts) |
| **Guest** | Deny | Deny | Deny | Deny | Deny | n/a | Allow (web only) |
| **Internet** | Deny | Deny | Deny | Allow (HTTPS to web server only) | Deny | Deny | n/a |

## 3. Detailed Rules

Rules are applied on the interface where the traffic **enters** the firewall. Order matters: the first match wins.

### 3.1 MANAGEMENT interface (10.10.10.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | ADMIN_HOST | This firewall | TCP 443, 22 | Admin the firewall (set this first) |
| 2 | Allow | ADMIN_HOST | Servers net | TCP 22, ICMP | SSH and ping to servers |
| 3 | Allow | ADMIN_HOST | DMZ net | TCP 22, ICMP | SSH and ping to web server |
| 4 | Allow | ADMIN_HOST | Monitoring net | TCP 22, 80, 443, ICMP | SSH and Zabbix web UI |
| 5 | Allow | ADMIN_HOST | Users net | ICMP | Ping only |
| 6 | Block | Any | Guest net | Any | Never reach guests |
| 7 | Allow | ADMIN_HOST | Not RFC1918 | TCP 80, 443; UDP 53 | Internet and updates |
| 8 | Block (log) | Any | Any | Any | Default deny |

### 3.2 SERVERS interface (10.10.20.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | Servers net | MON_SRV | TCP 10051, UDP/TCP 514 | Zabbix agent data and syslog out |
| 2 | Allow | CORE_SRV | Not RFC1918 | TCP 80, 443; UDP 53, 123 | Updates, DNS forwarding, time |
| 3 | Block (log) | Any | RFC1918 | Any | No access to other internal zones |
| 4 | Block (log) | Any | Any | Any | Default deny |

### 3.3 USERS interface (10.10.30.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | Users net | CORE_SRV | UDP/TCP 53 | DNS |
| 2 | Allow | Users net | CORE_SRV | TCP 445 | File shares (SMB) |
| 3 | Allow | Users net | CORE_SRV | UDP 123 | Time sync |
| 4 | Allow | Users net | WEB_DMZ | TCP 443 | Company website |
| 5 | Allow | Users net | This firewall | UDP 67/68 | DHCP |
| 6 | Block (log) | Any | RFC1918 | Any | Everything else internal is blocked |
| 7 | Allow | Users net | Any | TCP 80, 443 | Internet browsing |
| 8 | Block (log) | Any | Any | Any | Default deny |

### 3.4 DMZ interface (10.10.40.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | WEB_DMZ | MON_SRV | TCP 10051, UDP/TCP 514 | Agent data and logs out |
| 2 | Allow | WEB_DMZ | This firewall | UDP 53, 123 | DNS and time (DMZ never talks to the Servers zone) |
| 3 | **Block (log)** | Any | Servers net | Any | **DMZ must never reach Servers** |
| 4 | Block (log) | Any | RFC1918 | Any | No other internal access |
| 5 | Allow | WEB_DMZ | Any | TCP 80, 443 | Updates |
| 6 | Block (log) | Any | Any | Any | Default deny |

### 3.5 MONITORING interface (10.10.50.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | MON_SRV | Firewall (all interfaces) | UDP 161 | SNMP polling of the firewall |
| 2 | Allow | MON_SRV | Servers, Users, DMZ nets | TCP 10050, ICMP | Zabbix agent checks and ping |
| 3 | Allow | MON_SRV | This firewall | UDP 53, 123 | DNS and time |
| 4 | Block (log) | Any | Guest net | Any | Isolation |
| 5 | Allow | MON_SRV | Not RFC1918 | TCP 80, 443 | Updates and Telegram/email alerts |
| 6 | Block (log) | Any | Any | Any | Default deny |

### 3.6 GUEST interface (10.10.99.0/24)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | Guest net | This firewall | UDP 67/68, UDP 53 | DHCP and DNS |
| 2 | Block (log) | Any | RFC1918 | Any | **No internal access at all** |
| 3 | Allow | Guest net | Any | TCP 80, 443 | Web only |
| 4 | Block (log) | Any | Any | Any | Default deny |

### 3.7 WAN interface (Internet side)

| # | Action | Source | Destination | Port / Protocol | Purpose |
|---|---|---|---|---|---|
| 1 | Allow | Any | WEB_DMZ (port forward) | TCP 443 | Public website only |
| 2 | Block | Any | Any | Any | Default deny |

*In the lab the WAN sits behind your host's NAT, so test inbound access from the Ubuntu host.*

## 4. Design Decisions

| Decision | Reason |
|---|---|
| Firewall provides DHCP | Saves RAM and keeps the lab simple; servers use static IPs |
| DMZ uses the firewall for DNS, not core-srv | DMZ to Servers must stay fully blocked |
| Deny rules are logged | Blocked traffic feeds the monitoring and security detections |
| Monitoring is allowed in only on agent/SNMP ports | Monitoring servers are high-value targets |
| Block RFC1918 before allowing the internet | Stops "internet" rules from leaking into internal zones |

## 5. Test Table (fill in as you test)

| # | Test | Expected | Result | Evidence |
|---|---|---|---|---|
| 1 | Guest pings 10.10.20.10 | Blocked | | |
| 2 | Guest opens a website | Works | | |
| 3 | DMZ connects to 10.10.20.10 on any port | Blocked | | |
| 4 | Users resolve DNS via core-srv | Works | | |
| 5 | Users open https://web-dmz | Works | | |
| 6 | Users SSH to 10.10.20.10 | Blocked | | |
| 7 | Users reach monitor-srv | Blocked | | |
| 8 | Host (admin) SSH to core-srv | Works | | |
| 9 | Core-srv sends agent data to monitor-srv | Works | | |
| 10 | Host reaches https://web-dmz through WAN forward | Works | | |