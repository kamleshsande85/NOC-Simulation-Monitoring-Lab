# 📁 Project 5: NOC Simulation & Monitoring Lab

---

## 📌 Project Overview
This project simulates a **Network Operations Center (NOC)** environment with centralized logging, monitoring, and packet analysis. It demonstrates how real-world NOC engineers monitor network health, troubleshoot issues, and maintain documentation.

---

## 🎯 Objectives

| Module | Technology | Purpose |
|--------|------------|---------|
| **NTP** | Network Time Protocol | Time sync across all devices |
| **Syslog** | Centralized Logging | Collect logs from all devices |
| **SNMP** | Simple Network Management Protocol | Monitor device health |
| **Wireshark** | Packet Analysis | Capture and analyze traffic |
| **Incident Response** | Documentation | Troubleshooting procedures |

---

## 🏗️ Lab Topology (GNS3)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         NOC SIMULATION LAB                                │
│                                                                            │
│    ┌──────────────────────────────────────────────────────────────┐       │
│    │            NOC Server (Ubuntu VM)                            │       │
│    │            - Syslog Server (rsyslog)                        │       │
│    │            - SNMP Monitoring (snmpwalk)                     │       │
│    │            - Wireshark (Packet Capture)                     │       │
│    │            IP: 172.16.10.100/24                             │       │
│    └──────────────────────────┬───────────────────────────────────┘       │
│                               │                                          │
│                               │ Syslog (UDP 514), SNMP (UDP 161)        │
│                               ▼                                          │
│    ┌──────────────────────────────────────────────────────────────┐       │
│    │              Cloud Node (GNS3 VM Bridge)                     │       │
│    └──────────────────────────┬───────────────────────────────────┘       │
│                               │                                          │
│       ┌───────────────────────┼───────────────────────┐                  │
│       │                       │                       │                  │
│       ▼                       ▼                       ▼                  │
│  ┌─────────┐            ┌─────────┐            ┌─────────┐              │
│  │ Router1 │            │ Router2 │            │ Switch1 │              │
│  │172.16.  │            │172.16.  │            │172.16.  │              │
│  │10.1/24  │            │10.2/24  │            │10.10/24 │              │
│  └─────────┘            └─────────┘            └─────────┘              │
│                                                                            │
│   🔵 = NOC Components                                                     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Part 1: NTP Configuration

### Router1 — NTP Server

```
! ========================================
! Router1 — NTP Server
! ========================================

hostname R1

! Enable NTP Server
ntp master 1
ntp source loopback 0

! NTP Authentication
ntp authentication-key 1 md5 Cisco123!
ntp trusted-key 1
ntp authenticate

! Loopback Interface
interface loopback 0
 ip address 172.16.10.1 255.255.255.255
 no shutdown
exit

! Management Interface
interface gigabitethernet 0/0
 ip address 172.16.10.1 255.255.255.0
 no shutdown
exit

! Verification
do show ntp status
do show ntp associations
```

### Router2 — NTP Client

```
! ========================================
! Router2 — NTP Client
! ========================================

hostname R2

! NTP Client Configuration
ntp server 172.16.10.1
ntp source loopback 0

! NTP Authentication
ntp authentication-key 1 md5 Cisco123!
ntp trusted-key 1
ntp authenticate

! Loopback
interface loopback 0
 ip address 172.16.10.2 255.255.255.255
 no shutdown
exit

! Management Interface
interface gigabitethernet 0/0
 ip address 172.16.10.2 255.255.255.0
 no shutdown
exit

! Verification
do show ntp status
do show ntp associations
```

### Switch1 — NTP Client

```
! ========================================
! Switch1 — NTP Client
! ========================================

hostname SW1

! NTP Client
ntp server 172.16.10.1

! Management
interface vlan 1
 ip address 172.16.10.10 255.255.255.0
 no shutdown
exit

ip default-gateway 172.16.10.1

! Verification
do show ntp status
```

---

## 📋 Part 2: Syslog Configuration

### Router1 — Syslog

```
! ========================================
! Router1 — Syslog Configuration
! ========================================

! Enable Syslog
logging on
logging host 172.16.10.100
logging trap debugging
logging source-interface loopback 0
logging facility local7

! Local Logging
logging console critical
logging monitor informational
logging buffered informational

! Verification
do show logging
```

### Router2 — Syslog

```
! ========================================
! Router2 — Syslog Configuration
! ========================================

logging on
logging host 172.16.10.100
logging trap debugging
logging source-interface loopback 0
logging facility local7

! Verification
do show logging
```

### Switch1 — Syslog

```
! ========================================
! Switch1 — Syslog Configuration
! ========================================

logging on
logging host 172.16.10.100
logging trap notifications
logging source-interface vlan 1
logging facility local7

! Verification
do show logging
```

---

## 📡 Part 3: SNMP Configuration

### Router1 — SNMP

```
! ========================================
! Router1 — SNMP Configuration
! ========================================

! SNMPv2c (Read-Only)
snmp-server community RO_Community RO
snmp-server community RW_Community RW

! SNMPv3 (Secure)
snmp-server group SNMPv3_GROUP v3 priv
snmp-server user SNMPv3_USER SNMPv3_GROUP v3 auth sha Cisco123! priv aes 128 Cisco123!
snmp-server view ALL iso included

! SNMP Traps
snmp-server enable traps
snmp-server host 172.16.10.100 version 2c RO_Community
snmp-server host 172.16.10.100 version 3 priv SNMPv3_USER

! Verification
do show snmp
do show snmp group
do show snmp user
```

### Router2 — SNMP

```
! ========================================
! Router2 — SNMP Configuration
! ========================================

snmp-server community RO_Community RO
snmp-server enable traps
snmp-server host 172.16.10.100 version 2c RO_Community

! Verification
do show snmp
```

### Switch1 — SNMP

```
! ========================================
! Switch1 — SNMP Configuration
! ========================================

snmp-server community RO_Community RO
snmp-server enable traps
snmp-server host 172.16.10.100 version 2c RO_Community

! Verification
do show snmp
```

---

## 🖥️ Part 4: NOC Server Setup (Ubuntu VM)

### Step 1: Syslog Server Setup

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Install rsyslog
sudo apt install rsyslog -y

# Edit config
sudo nano /etc/rsyslog.conf

# Add these lines
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")

# Create remote log template
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs

# Create directory
sudo mkdir -p /var/log/remote

# Restart rsyslog
sudo systemctl restart rsyslog
sudo systemctl enable rsyslog

# Check status
sudo systemctl status rsyslog
```

### Step 2: SNMP Tools Setup

```bash
# Install SNMP tools
sudo apt install snmp snmpd snmp-mibs-downloader -y

# Test SNMP walk
snmpwalk -v 2c -c RO_Community 172.16.10.1 system

# Test SNMPv3
snmpwalk -v 3 -u SNMPv3_USER -l priv -a sha -A Cisco123! -x aes -X Cisco123! 172.16.10.1 system

# Get system info
snmpwalk -v 2c -c RO_Community 172.16.10.1 sysDescr
snmpwalk -v 2c -c RO_Community 172.16.10.1 sysName
snmpwalk -v 2c -c RO_Community 172.16.10.1 sysLocation
snmpwalk -v 2c -c RO_Community 172.16.10.1 sysContact

# Get interface info
snmpwalk -v 2c -c RO_Community 172.16.10.1 ifDescr
snmpwalk -v 2c -c RO_Community 172.16.10.1 ifOperStatus
snmpwalk -v 2c -c RO_Community 172.16.10.1 ifSpeed
```

### Step 3: Wireshark Setup

```bash
# Install Wireshark
sudo apt install wireshark -y

# Add user to wireshark group
sudo usermod -a -G wireshark $USER

# Launch Wireshark
wireshark &

# Filter: ip.addr == 172.16.10.0/24
# Capture filter: udp port 514 or udp port 161/162
```

---

## ✅ Part 5: Verification

### NTP Verification

```
R1# show ntp status
Clock is synchronized, stratum 1, reference is 127.127.1.1
✅

R2# show ntp associations
  address         ref clock     st   when   poll reach  delay  offset   disp
*~172.16.10.1   127.127.1.1     1      2     64    377  0.000   0.00  0.000
✅
```

### Syslog Verification

```bash
# Ubuntu VM
sudo tail -f /var/log/remote/R1/R1.log
Aug 4 15:30:01 R1 %SYS-5-CONFIG_I: Configured from console by console
Aug 4 15:31:00 R1 %LINEPROTO-5-UPDOWN: Line protocol on Interface Gig0/0, changed state to up
✅
```

### SNMP Verification

```bash
# Ubuntu VM
snmpwalk -v 2c -c RO_Community 172.16.10.1 system
SNMPv2-MIB::sysDescr.0 = STRING: Cisco IOS Software, Version 15.1...
SNMPv2-MIB::sysName.0 = STRING: R1
✅
```

### Wireshark Verification

```
Filter: ip.addr == 172.16.10.1 or ip.addr == 172.16.10.100
Expected packets:
- Syslog: UDP 514
- SNMP: UDP 161/162
- ICMP: ping
✅
```

---

## 📸 Screenshot List (GitHub Ke Liye)

| # | Screenshot | Command |
|---|------------|---------|
| 1 | **GNS3 Topology** | GNS3 window |
| 2 | **NTP Status (R1)** | `show ntp status` |
| 3 | **NTP Associations (R2)** | `show ntp associations` |
| 4 | **Syslog Logs (R1)** | `sudo tail -f /var/log/remote/R1/R1.log` |
| 5 | **Syslog Logs (R2)** | `sudo tail -f /var/log/remote/R2/R2.log` |
| 6 | **SNMP Walk (R1)** | `snmpwalk -v 2c -c RO_Community 172.16.10.1 system` |
| 7 | **SNMP Interface (R1)** | `snmpwalk -v 2c -c RO_Community 172.16.10.1 ifDescr` |
| 8 | **Wireshark Capture** | Wireshark GUI |
| 9 | **SNMPv3 Walk** | `snmpwalk -v 3 -u SNMPv3_USER -l priv -a sha -A Cisco123! -x aes -X Cisco123! 172.16.10.1` |
| 10 | **Incident Response Doc** | Documentation screenshot |

---

## 📝 Incident Response Template

### Incident Report Template

```
========================================
NETWORK INCIDENT REPORT
========================================

INCIDENT ID: INC-2026-08-001
DATE/TIME: 2026-08-04 15:30:00
SEVERITY: P2 (High)
REPORTED BY: NOC Engineer

----------------------------------------
DESCRIPTION:
[What happened]
----------------------------------------
IMPACT:
[Which devices/services affected]
----------------------------------------
ROOT CAUSE:
[Why it happened]
----------------------------------------
RESOLUTION:
[How it was fixed]
----------------------------------------
TIMELINE:
- 15:30:00 - Incident detected
- 15:32:00 - Investigation started
- 15:45:00 - Root cause identified
- 16:00:00 - Resolution implemented
- 16:15:00 - Service restored
----------------------------------------
PREVENTIVE MEASURES:
[How to prevent in future]
----------------------------------------
```

### Sample Incident Report

```
========================================
NETWORK INCIDENT REPORT
========================================

INCIDENT ID: INC-2026-08-001
DATE/TIME: 2026-08-04 15:30:00
SEVERITY: P2 (High)
REPORTED BY: NOC Engineer

----------------------------------------
DESCRIPTION:
Router1 became unreachable. All network traffic to Branch1 affected.
----------------------------------------
IMPACT:
- Router1 unreachable
- 50 users in Branch1 lost connectivity
- 3 critical servers unreachable
----------------------------------------
ROOT CAUSE:
Interface Gig0/0 went down due to cable issue.
----------------------------------------
RESOLUTION:
- Verified physical cable connection
- No shutdown interface
- Interface came back up
----------------------------------------
TIMELINE:
- 15:30:00 - Monitoring alert
- 15:32:00 - Investigation started
- 15:35:00 - Interface found down
- 15:40:00 - Cable replaced
- 15:45:00 - Interface up
- 15:46:00 - Connectivity restored
- 15:50:00 - Incident closed
----------------------------------------
PREVENTIVE MEASURES:
- Redundant link to be configured
- Cable replacement SOP created
- Monitoring alerts to be improved
----------------------------------------
```

---

## 🎯 LinkedIn Post Template

> **"🚀 Project 5: NOC Simulation & Monitoring Lab Completed!**
>
> **✅ NTP synchronized across all network devices**
> **✅ Centralized Syslog server collecting logs**
> **✅ SNMPv2c/v3 monitoring active**
> **✅ Wireshark packet capture and analysis**
> **✅ Incident response documentation**
>
> **#NOC #NetworkMonitoring #Syslog #SNMP #CCNA #GNS3"**

---

## 📂 GitHub Repository Structure

```
Project5-NOC-Simulation/
├── README.md
├── configs/
│   ├── Router1.txt
│   ├── Router2.txt
│   └── Switch1.txt
├── screenshots/
│   ├── topology.png
│   ├── ntp-status.png
│   ├── ntp-associations.png
│   ├── syslog-logs.png
│   ├── snmp-walk.png
│   ├── snmp-interfaces.png
│   ├── wireshark-capture.png
│   └── incident-response.png
├── scripts/
│   ├── syslog_setup.sh
│   ├── snmp_setup.sh
│   └── ntp_setup.sh
├── docs/
│   └── incident-response-template.md
└── logs/
    └── (generated syslog files)
```

---

## 🔧 Verification Checklist

| Feature | Command | Status |
|---------|---------|--------|
| NTP Server | `show ntp status` on R1 | [ ] |
| NTP Client | `show ntp associations` on R2 | [ ] |
| Syslog Server | `sudo systemctl status rsyslog` | [ ] |
| Syslog Logs | `tail -f /var/log/remote/R1/R1.log` | [ ] |
| SNMPv2c | `snmpwalk -v 2c -c RO_Community 172.16.10.1 system` | [ ] |
| SNMPv3 | `snmpwalk -v 3 -u SNMPv3_USER -l priv -a sha -A Cisco123! -x aes -X Cisco123! 172.16.10.1` | [ ] |
| Wireshark | Capture packets on UDP 514 | [ ] |

---

**Bhai, Project 5 documentation complete hai!** 🎯

**Ab job hunting start karo!** 🚀

**All the best! 💪**
