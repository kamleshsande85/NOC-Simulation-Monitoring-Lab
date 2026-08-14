# 📁 Project 5: NOC Simulation & Monitoring Lab

---

## 📌 Project Overview

This project simulates a **Network Operations Center (NOC)** environment with centralized logging, monitoring, and packet analysis. It demonstrates how real-world NOC engineers monitor network health and investigate incidents using NTP, Syslog, SNMP and packet captures.

---

## 🏗️ Lab Topology (GNS3)

Below is the topology image used for this lab. (Replaced the previous text/ASCII topology with the actual image from the repository.)

![NOC Topology — GNS3 project](Screenshort/01_topology.png)

---

## 🎯 Objectives

| Module | Technology | Purpose |
| --- | --- | --- |
| **NTP** | Network Time Protocol | Time sync across all devices |
| **Syslog** | Centralized Logging | Collect logs from all devices |
| **SNMP** | Simple Network Management Protocol | Monitor device health |
| **Wireshark** | Packet Analysis | Capture and analyze traffic |
| **Incident Response** | Documentation | Troubleshooting procedures |

---

## 🌐 IP Addressing Scheme

| Device | Interface | IP Address | Subnet Mask | Role |
| --- | --- | --- | --- | --- |
| NOC Server | `gns3tap0` | 172.16.10.100 | 255.255.255.0 | Monitoring Host (rsyslog, SNMP, Wireshark) |
| Router1 (R1) | Gi0/0 | 172.16.10.1 | 255.255.255.0 | Primary Gateway / NTP Master / SNMP Agent |
| Router2 (R2) | Gi0/0 | 172.16.10.2 | 255.255.255.0 | Secondary Gateway / NTP Client / SNMP Agent |

---

## ⚙️ Module-by-Module Configuration & Execution

---

### 🔹 Module 1: Network Time Protocol (NTP)

**Objective:** Sync system clock across all routers.

#### Configuration (Router 1 - NTP Server)

```ios
R1# configure terminal
! Basic IP Setup
interface GigabitEthernet0/0
 ip address 172.16.10.1 255.255.255.0
 no shutdown
exit

! NTP Master Setup
ntp master 2
clock set 12:00:00 14 Aug 2026
end
write memory
```

#### Configuration (Router 2 - NTP Client)

```ios
R2# configure terminal
! Basic IP Setup
interface GigabitEthernet0/0
 ip address 172.16.10.2 255.255.255.0
 no shutdown
exit

! NTP Client Setup
ntp server 172.16.10.1
end
write memory
```

#### Verification Commands

```ios
! On Router 1
R1# show ntp status
R1# show ntp associations

! On Router 2
R2# show ntp status
R2# show ntp associations
```

NTP verification screenshots:

![NTP status — Router1](Screenshort/02_ntp_status-Router1.png)

![NTP status — Router2](Screenshort/02_ntp_status-Router2.png)

![NTP associations — Router1](Screenshort/03_ntp_associations-Router1.png)

![NTP associations — Router2](Screenshort/03_ntp_associations-Router2.png)

---

### 🔹 Module 2: Centralized Syslog Server

**Objective:** Send all router logs to Kali NOC Server.

#### Configuration (Routers R1 & R2)

```ios
configure terminal
logging host 172.16.10.100
logging trap informational
logging facility local7
logging source-interface GigabitEthernet0/0
end
write memory
```

#### NOC Server Configuration (Kali Linux)

1. Edit `/etc/rsyslog.conf`:

```bash
sudo nano /etc/rsyslog.conf
```

2. Enable UDP module (uncomment these lines):

```text
module(load="imudp")
input(type="imudp" port="514")
```

3. Restart rsyslog service:

```bash
sudo systemctl restart rsyslog
```

#### Verification Command (On NOC Server)

```bash
sudo tail -f /var/log/syslog
```

Syslog capture screenshot:

![Syslog logs](Screenshort/04_syslog_logs.png)

---

### 🔹 Module 3: SNMP Monitoring

**Objective:** Monitor network device status remotely via SNMP.

#### Configuration (Routers R1 & R2)

```ios
configure terminal
snmp-server community public RO
snmp-server location Enterprise_HQ_DataCenter
snmp-server contact Admin_NOC@company.com
snmp-server enable traps
end
write memory
```

#### Verification Commands (On NOC Server)

```bash
! Poll System Info (R1)
snmpwalk -v2c -c public 172.16.10.1 system

! Poll System Info (R2)
snmpwalk -v2c -c public 172.16.10.2 system
```

SNMP verification screenshots:

![SNMP walk — Router1](Screenshort/05_snmp_walk-Router1.png)

![SNMP walk — Router2](Screenshort/05_snmp_walk-Router2.png)

---

### 🔹 Module 4: Wireshark Packet Analysis

**Objective:** Capture and analyze SNMP, NTP, and Syslog traffic.

#### Capture Setup

1. Interface: `gns3tap0`
2. Filter: `udp port 161 or udp port 123 or udp port 514` (or GUI filter: `snmp || ntp || syslog`)

#### Packet Capture Command (CLI / TShark)

```bash
sudo tshark -i gns3tap0 -f "udp port 161 or udp port 123 or udp port 514"
```

#### Wireshark GUI Command

```bash
sudo wireshark -i gns3tap0 -k
```

Packet capture screenshots:

![TShark capture](Screenshort/06_tshark_capture.png)

![Wireshark capture](Screenshort/07_wireshark_capture.png)

---

### 5. Post-Incident Status (Example)

```text
========================================
NETWORK INCIDENT REPORT
========================================

INCIDENT ID: INC-2026-08-001
DATE/TIME: 2026-08-04 15:30:00
SEVERITY: P2 (High)
REPORTED BY: NOC Engineer

----------------------------------------
DESCRIPTION:
Unplanned interface state flap detected on core gateway Router R1 (GigabitEthernet0/0). The interface transitioned to DOWN state unexpectedly, triggering OSPF neighbor adjacency drop and temporary service interruptions.
----------------------------------------
IMPACT:
- Primary Core Gateway R1 (172.16.10.1) interface Gi0/0 became unreachable.
- Temporary loss of Syslog logging and SNMP telemetry polling on NOC Server (172.16.10.100).
- NTP Client synchronization on Router R2 (172.16.10.2) briefly degraded due to master sync loss.
----------------------------------------
ROOT CAUSE:
Virtual TAP bridge adapter (gns3tap0) binding mismatch on Cloud Node caused a physical/datalink layer disconnection between MGMT_SW1 and core gateway R1.
----------------------------------------
RESOLUTION:
1. Re-mapped and bound the TAP interface 'gns3tap0' cleanly to the GNS3 Cloud Node.
2. Executed interface reset sequence on R1: 'shutdown' followed by 'no shutdown' on interface GigabitEthernet0/0.
3. Verified Layer 2/Layer 3 link status, restored SNMP walk queries, and confirmed NTP peer synchronization.
----------------------------------------
TIMELINE:
- 15:30:00 - Incident detected via Syslog alert and SNMP polling timeout
- 15:32:00 - Investigation started on NOC Server (gns3tap0 interface)
- 15:45:00 - Root cause identified as TAP adapter binding drop on Cloud Node
- 16:00:00 - Resolution implemented (TAP interface re-bound and Gi0/0 bounced)
- 16:15:00 - Service restored (SNMP, Syslog, and NTP verified operational)
----------------------------------------
PREVENTIVE MEASURES:
1. Automate TAP interface startup and persistent IP configuration via systemd service script on NOC Host.
2. Configure SNMP Traps for immediate interface status change notifications to reduce detection time.
3. Implement secondary redundant management path for out-of-band monitoring.
----------------------------------------
```

---

## 📂 Repository structure (actual)

Repository root (current):

- README.md
- Project 5 NOC Simulation.md
- GNS3 data/
  - Project 5: NOC Simulation.gns3
- Screenshort/
  - 01_topology.png
  - 02_ntp_status-Router1.png
  - 02_ntp_status-Router2.png
  - 03_ntp_associations-Router1.png
  - 03_ntp_associations-Router2.png
  - 04_syslog_logs.png
  - 05_snmp_walk-Router1.png
  - 05_snmp_walk-Router2.png
  - 06_tshark_capture.png
  - 07_wireshark_capture.png

Notes:
- Directories/files referenced elsewhere in older drafts (for example `configs/`, `scripts/`, `docs/`, `logs/`) are not present in the repository right now. If you want those added, tell me and I will add them.
- The GNS3 project file is in `GNS3 data/Project 5: NOC Simulation.gns3`. Use that to import the topology into GNS3.

---

## 🔧 Verification Checklist

| Feature | Command | Status |
| --- | --- | --- |
| NTP Server | `show ntp status` on R1 | [x] |
| NTP Client | `show ntp associations` on R2 | [x] |
| Syslog Server | `sudo systemctl status rsyslog` | [x] |
| Syslog Logs | `sudo tail -f /var/log/syslog` | [x] |
| SNMP Monitoring | `snmpwalk -v2c -c public 172.16.10.1 system` | [x] |
| Wireshark Traffic | `sudo tshark -i gns3tap0` | [x] |

---

## 🔧 Useful files (where to find configs & scripts)

- `GNS3 data/Project 5: NOC Simulation.gns3` — import this into GNS3 to recreate the topology.
- Screenshots are in `Screenshort/`.

---

## 🔧 Prerequisites

- GNS3 (recommended recent release)
- Kali Linux (or other Linux host) for rsyslog, snmp tools and Wireshark
- Cisco IOS images or compatible router images for GNS3

## 🔧 How to use this repo

1. Import the provided GNS3 project or recreate the topology in GNS3 using the topology diagram in this document.
2. Provision the Kali host and bind a TAP interface (`gns3tap0`) to the Cloud node.
3. Apply router configs from the configuration snippets above.
4. Start monitoring: rsyslog, snmpwalk and Wireshark/TShark.

## 🔧 License & Credits

This repository is provided for learning and lab purposes. Use at your own risk. Credits to the author/maintainer.

---

(Project file updated to include inline screenshots.)
