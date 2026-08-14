# NOC Simulation & Monitoring Lab

This repository contains a lab for simulating a Network Operations Center (NOC) environment using GNS3 and a small device topology. The lab demonstrates centralized logging (rsyslog), SNMP monitoring, NTP time synchronization, and packet analysis with Wireshark/TShark.

## Project Overview

The NOC Simulation & Monitoring Lab recreates a simple management network and core fabric to teach and test common NOC tasks: collecting logs, polling device telemetry, keeping devices time-synchronized, and capturing/analyzing network traffic. It is suitable for training, demos, and validation of monitoring configurations in a controlled environment.

## Objectives

- NTP: Keep device clocks synchronized across the topology.
- Syslog: Aggregate router logs centrally on a Kali Linux NOC server running rsyslog.
- SNMP: Poll device health and configuration using SNMPv2c.
- Packet Analysis: Capture and inspect NTP, SNMP, and Syslog traffic with Wireshark / TShark.
- Incident Response: Document and practice handling network incidents.

## Lab Topology (GNS3)

The lab uses a small topology with a management zone and an OSPF core fabric.

- NOC Management Zone (172.16.10.0/24)
  - Kali Linux / NOC Server (172.16.10.100): rsyslog, SNMP tools, Wireshark/TShark
  - Management switch (MGMT_SW1)
- OSPF Area 0 Core Fabric
  - Router R1 (NTP master / SNMP agent) — 172.16.10.1
  - Router R2 (NTP client / SNMP agent) — 172.16.10.2

(See `Project 5 NOC Simulation.md` for a full ASCII topology diagram.)

## IP Addressing Scheme

| Device       | Interface  | IP Address      | Subnet Mask      | Role                                      |
|--------------|------------|-----------------|------------------|-------------------------------------------|
| NOC Server   | gns3tap0   | 172.16.10.100   | 255.255.255.0    | Monitoring host (rsyslog, snmpwalk, wireshark) |
| Router R1    | Gi0/0      | 172.16.10.1     | 255.255.255.0    | Primary gateway / NTP master / SNMP agent |
| Router R2    | Gi0/0      | 172.16.10.2     | 255.255.255.0    | Secondary gateway / NTP client / SNMP agent |

## Module-by-Module Configuration

### Module 1 — NTP (Time Sync)

Router 1 (NTP server / master):

```ios
configure terminal
interface GigabitEthernet0/0
 ip address 172.16.10.1 255.255.255.0
 no shutdown
exit

ntp master 2
clock set 12:00:00 14 Aug 2026
end
write memory
```

Router 2 (NTP client):

```ios
configure terminal
interface GigabitEthernet0/0
 ip address 172.16.10.2 255.255.255.0
 no shutdown
exit

ntp server 172.16.10.1
end
write memory
```

Verification:
- On routers: `show ntp status`, `show ntp associations`

### Module 2 — Centralized Syslog (rsyslog)

On routers (R1 & R2):

```ios
configure terminal
logging host 172.16.10.100
logging trap informational
logging facility local7
logging source-interface GigabitEthernet0/0
end
write memory
```

On Kali NOC Server (rsyslog):
1. Edit `/etc/rsyslog.conf` and enable UDP:

```
module(load="imudp")
input(type="imudp" port="514")
```

2. Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

Check logs:

```bash
sudo tail -f /var/log/syslog
```

### Module 3 — SNMP Monitoring

On routers:

```ios
configure terminal
snmp-server community public RO
snmp-server location Enterprise_HQ_DataCenter
snmp-server contact Admin_NOC@company.com
snmp-server enable traps
end
write memory
```

On the NOC server, verify with:

```bash
snmpwalk -v2c -c public 172.16.10.1 system
snmpwalk -v2c -c public 172.16.10.2 system
```

### Module 4 — Packet Capture (Wireshark / TShark)

Capture interface: `gns3tap0`

Capture filter (BPF): `udp port 161 or udp port 123 or udp port 514`

TShark (CLI):

```bash
sudo tshark -i gns3tap0 -f "udp port 161 or udp port 123 or udp port 514"
```

Wireshark GUI:

```bash
sudo wireshark -i gns3tap0 -k
```

## Incident Response Example

An example incident report is included to demonstrate documenting detection, impact, root cause, resolution and preventive measures. See `Project 5 NOC Simulation.md` for a full incident timeline and remediation steps.

## Repository Structure

```
Project5-NOC-Simulation/
├── README.md
├── Project 5 NOC Simulation.md
├── GNS3 data/
├── Screenshort/ (screenshots)
├── configs/
│   ├── Router1.txt
│   ├── Router2.txt
│   └── Switch1.txt
├── scripts/
│   ├── syslog_setup.sh
│   ├── snmp_setup.sh
│   └── ntp_setup.sh
├── docs/
│   └── incident-response-template.md
└── logs/
    └── (generated syslog files)
```

> Note: Filenames and subfolders may vary — this README summarizes the content and points to the main guidance file `Project 5 NOC Simulation.md` which contains detailed configs, commands and screenshots.

## Verification Checklist

- NTP Server: `show ntp status` on R1
- NTP Client: `show ntp associations` on R2
- Syslog Server: `sudo systemctl status rsyslog` on NOC host
- Syslog Logs: `sudo tail -f /var/log/syslog`
- SNMP Monitoring: `snmpwalk -v2c -c public 172.16.10.1 system`
- Wireshark: `sudo tshark -i gns3tap0`

## Prerequisites

- GNS3 (recommended recent release)
- Kali Linux (or other Linux host) for rsyslog, snmp tools and Wireshark
- Cisco IOS images or compatible router images for GNS3

## How to use this repo

1. Import the provided GNS3 project or recreate the topology in GNS3 using the topology diagram in `Project 5 NOC Simulation.md`.
2. Provision the Kali host and bind a TAP interface (`gns3tap0`) to the Cloud node.
3. Apply router configs from `configs/` or via the CLI snippets in `Project 5 NOC Simulation.md`.
4. Start monitoring: rsyslog, snmpwalk and Wireshark/TShark.

## License & Credits

This repository is provided for learning and lab purposes. Use at your own risk. Credits to the author/maintainer.

---

(README updated using `Project 5 NOC Simulation.md` as reference.)
