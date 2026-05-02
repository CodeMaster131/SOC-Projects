# 🛡️ SIEM Home Lab Walkthrough – Splunk, Sysmon & Windows Security Logging

This document outlines the process of building a functional home SIEM lab using Splunk and Windows logging tools. It is written as technical lab notes, focusing on setup, troubleshooting, and security monitoring workflows.

The goal of this lab is to simulate how SOC analysts ingest logs, detect suspicious activity, and investigate potential attacks such as authentication abuse and network scanning.

---

## 🧱 Lab Environment

### Operating Systems
- Windows 10 Virtual Machine (Target / Log Source)
- Kali Linux (Attacker Machine)

### Tools Used
- Splunk Enterprise (SIEM Platform)
- Sysmon (Windows Event Logging Enhancement)
- Nmap (Network Reconnaissance)
- Metasploit Framework (Exploitation / Simulation)

### Objective
Simulate attacker behavior from Kali Linux and detect malicious activity on a Windows endpoint using Splunk and Sysmon logs.


---

## 🚀 Step 1 – Installing Splunk Enterprise

Splunk Enterprise was installed on a Windows 10 virtual machine to function as the SIEM platform. It is responsible for ingesting, indexing, and analyzing log data from endpoint sources.

After installation, Splunk Web was verified locally:

`http://127.0.0.1:8000`

This confirmed that the Splunk service was running correctly.
![image](https://hackmd.io/_uploads/SycdIii6-e.png)
---

## ⚠️ Step 2 – Install Sysmon and install Olaf Configuration

After setting up Splunk, Sysmon was installed on the Windows machine to enhance system logging capabilities. Sysmon provides detailed event-level visibility such as:

- Process creation events
- Network connections
- File creation timestamps
- Parent-child process relationships

Sysmon download:
https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon
![image](https://hackmd.io/_uploads/HJXoDsjaZe.png)

---

### 🔧 Sysmon Configuration (Olaf Hartong Config)

To improve logging quality and reduce noise, the Sysmon configuration file from Olaf Hartong was used. This configuration is widely used in SOC environments for structured telemetry collection.

GitHub Repository:
https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml

The configuration helps normalize logs and improves detection of suspicious behavior such as:
- Process injection
- Credential dumping attempts
- Suspicious network connections





---
