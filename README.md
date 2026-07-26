# 🛡️AI-assisted-Detection-Engineering-Incident-Response
AI-assisted Detection Engineering & Incident Response platform that integrates Detection-as-Code, Sigma, SIEM, SOAR, GitHub Actions, Jira, and an LLM Security Copilot to automate detection, investigation, response, reporting, and continuous detection tuning.

## 📖 Overview
This project demonstrates the design and implementation of an AI-assisted Detection Engineering and Incident Response platform that simulates a modern Security Operations Center (SOC). The platform combines Detection-as-Code principles with automated incident response, allowing security detections to be developed, tested, version-controlled, deployed, investigated, and continuously improved through a unified workflow.

The environment integrates Windows endpoints, Sysmon, Splunk, and Atomic Red Team to generate and collect security telemetry. Sigma rules are developed and managed through a Git-based CI/CD pipeline, automatically validated and converted into SIEM-specific detection logic. When malicious activity is detected, SOAR workflows orchestrate evidence collection, create Jira incident tickets, and initiate automated response actions.

An integrated LLM Security Copilot enhances analyst workflows by performing alert triage, ATT&CK mapping, timeline reconstruction, investigation assistance, playbook generation, incident reporting, and detection tuning recommendations. Rather than replacing analysts, the AI augments decision-making and reduces repetitive tasks, enabling faster and more consistent incident response.

The project is designed to mirror the workflows used by modern Security Operations Centers, demonstrating practical skills in detection engineering, incident response, security automation, threat detection, and AI-assisted security operations.

### Flow Breakdown
1. **Telemetry Ingestion:** A Windows Target VM logs process execution telemetry via Microsoft Sysmon and forwards it to Splunk Enterprise.
2. **Alert Trigger:** Splunk detects suspicious execution (e.g., `whoami.exe` or local account discovery) and posts a JSON payload to a Tines Webhook Listener.
3. **Automated Incident Creation:** Tines parses the alert telemetry, formats an AI Analyst Triage Report, and creates a Jira task issue using Jira's REST API.
4. **Analyst Approval Loop:** A SOC analyst reviews the Jira ticket. To execute containment, the analyst simply adds the `auto-isolate` label to the Jira ticket.
5. **Orchestrated Host Isolation:** Jira sends a webhook back to Tines. Tines validates the event via condition rules, executes a host network isolation action, and automatically posts a formatted Wiki Markdown containment audit comment back to the Jira ticket.

## 🧰 Tech Stack

* **Hypervisor:** Oracle VM VirtualBox
* **Target Endpoint:** Windows 10 VM
* **SIEM:** Splunk Enterprise & Splunk Universal Forwarder
* **Telemetry Agent:** Microsoft Sysmon (System Monitor)
* **SOAR Platform:** Tines (Cloud Edition)
* **Ticketing & Incident Management:** Jira Software Cloud

## 🛠️ Phase 1: Environment Setup & Virtual Machines
## Part 1: VM Setup

We're going to create 2 virtual machines: Windows 10 and Splunk(SIEM). 
## 🖥️ Virtual Machine Configuration

| VM Name            | Role             | OS             | CPUs | RAM   | Disk Size        |
|--------------------|------------------|----------------|------|-------|------------------|
| Windows-10         | Victim/Endpoint  | Windows 10     | 2    | 4 GB  | 60 GB (default)  |
| Splunk             | SIEM             | Ubuntu Server  | 2    | 8 GB  | 100 GB           |


## ⚙️ VM Creation Steps

For each virtual machine:

1. Follow your hypervisor's **"New Virtual Machine" wizard**.  
2. Point the installer to the corresponding ISO (Windows or Ubuntu).  
3. Assign the resources as specified in the [VM Configuration Table](#🖥️-virtual-machine-configuration).  
4. Set the **Network Adapters** to **NAT** and **Host-Only Adapter** for all VMs.  
5. Power on the VMs to begin OS installation.

### A. Windows 10 (Windows-10)
 * **OS:** Windows 10 (64-bit)
   * **RAM:** 4096 MB
   * **Processors:** 2 vCPUs

1. Follow the Windows setup wizard.  
2. When prompted, select:
   - **"I don't have a product key."**
   - **Windows 10 Pro**
   - **Custom: Install Windows only**
3. During the initial setup (OOBE):
   - Set up for personal use → Offline account → Limited experience.  
   - Create a local user (e.g., `mydfir`) and set a password.  
   - Decline all privacy and telemetry options.  
4. Once at the desktop, enable Remote Desktop:
   - Start → *Remote desktop settings* → Toggle **Enable Remote Desktop** to **On**.
   - disable Windows Defender real-time protection (for lab testing purposes)
5. Open `cmd` and get the VM’s IP address: ipconfig

### B. Ubuntu Servers (Splunk)
   * **OS:** Ubuntu (64-bit)
   * **RAM:** 8192 MB
   * **Processors:** 4 vCPUs

1. 	Follow the Ubuntu Server setup wizard.
2. 	Keep defaults for language, keyboard, and network (DHCP).
3. 	For storage, select Use an entire disk.
4. 	Set up your user (e.g., Name: , Server Name: , Username: ).
5. 	CRITICAL: When prompted, check the box to Install OpenSSH server.
6. 	Wait for installation to finish and reboot.
7. 	Log in to the console, get the IP address: ip a, and run system updates: sudo apt-get update && sudo apt-get upgrade -y

Note: Keep a record of all VM IP addresses.

## 🔬 Part 2: Splunk Installation & Configuration

### 1. Download and Install Splunk
1. 	On your host machine, go to the Splunk Enterprise download page and copy the  link for the  file.
2. 	In your SSH session for the mydfir-splunk VM, download the file: wget -O splunk.deb "https://download.splunk.com/..." (Remember your link will be different)
3. 	Install the package: sudo dpkg -i splunk.deb
4. 	Start Splunk and accept the license: sudo /opt/splunk/bin/splunk start
5.  Enable Splunk to start on boot: sudo /opt/splunk/bin/splunk enable boot-start -user splunk

### 2. Configure Splunk GUI
On your host browser, go to http://your-splunk-vm-ip:8000 and Log in with the credentials you just created.

### A. Enable Data Receiving
1. Go to Settings > Forwarding & receiving.
2. Under "Receive data," click Configure receiving.
3. Click New Receiving Port, enter 9997, and click Save.

<img width="653" height="639" alt="514913755-84b5ae48-dad3-4dcd-9614-9ae7d2dc7ff8" src="https://github.com/user-attachments/assets/e9df1858-7204-475f-98b8-7b49b22b11e0" />

## Part 3: Install Sysmon & Splunk Universal Forwarder
### Step 1. Install Microsoft Sysmon
On your Windows VM, install Sysmon by navigating to https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon. Next, navigate to https://github.com/olafhartong/sysmon-modular scroll down and select sysmonconfig.xml. Click "raw", and save the file. Extract the sysmon file, copy URL of the extracted directory, take the sysmonconfig.xml file and place it in the extracted Sysmon file.and open Powershell as administrator, and navigate to that directory. Run .\Sysmon64.exe -i ..\sysmonconfig.xml, then click agree. 

### 2. Install Splunk Universal Forwarder
1. Download and install the Splunk Universal Forwarder on the Windows VM.
   - Follow the setup wizard:
      - Accept the license agreement.
      - Select Splunk Enterprise (on-premise).
      - Username: (local user for the forwarder).
      - Set a password.
      - Deployment Server: Leave blank.
      - Receiving Indexer:
          Host: Your splunk VM IP
          Port: 9997
      - Click Next and Install.
2. Create a New Index
   - Go to Settings > Indexes.
   - Click New Index.
   - Set Index Name to [project-name] and click Save.
3. Install Windows Add-on
   - Go to Apps > Find More Apps.
   - Search for Windows event and install the Splunk Add-on for Microsoft Windows.
   - You will need to enter your splunk.com credentials (not your server credentials) to install.
Finally, take a snapshot of the Splunk VM and name it Splunk-installed.

### ⚡ Phase 2: Splunk Alerting & Webhook Ingestion
## Part 1: Send Windows Telemetry to Splunk
1. On the Windows 10 VM, navigate to C:\Program Files\SplunkUniversalForwarder\etc\system\local
2. Open notepad and create a new file
3. Head over to [inputs.conf](https://drive.google.com/file/d/1-qYp4oCrT1BqhG1oaprQfkFhgFIHiWEm/view) and copy the entire text to paste it into the notepad file, then save the file as inputs.conf under C:\Program Files\SplunkUniversalForwarder\etc\system\local
4. Run the Services app as admin (services.msc).
5. Find the SplunkForwarder service:
   - Right-click > Properties > Log On tab.
   - Select Local System account and click Apply.
   - Right-click the service and Restart.

To verify:
1. Go back to your Splunk web interface and click on Search & Reporting.
2. In the search bar, type in index="mydfir-project" and set the time range to Last 24 hours

You should be able to see events from your Windows 10 VM 

