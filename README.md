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

## ⚡ Phase 2: Splunk Alerting & Webhook Ingestion
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

## Part 2: Building the Detection Engineering Repository
### 🏗️ Step 1: Create Your GitHub Repository

To treat detections as code, you need a centralized repository where your rules can live, undergo peer review, and eventually trigger automated testing and deployment.
1. Create the Repository
   - Login to Github
   - Create a new private repository named something like: detection-engineering-lab
   - Clone the repository to your local workstation through Github Desktop

2. Establish Your Folder Structure


   <img width="741" height="216" alt="image" src="https://github.com/user-attachments/assets/734d1862-a892-41d4-9bad-fc630ad1c5a4" />

### 📝 Step 2: Stage Your First Sigma Rule

Inside rules/windows/, create a new file: win_edr_tampering_stopped.yml and enter in the following Sigma rule:

<img width="730" height="715" alt="image" src="https://github.com/user-attachments/assets/6211c23b-edb0-4c01-94cc-34be378a9f86" />


### 🚀 Step 3: Commit Your First Detection Rule

- Open GitHub Desktop. It will automatically detect that you added a new file and display the green changes on the screen.

- In the bottom-left corner, type a summary title like: Add EDR tampering rule.

- Click the blue Commit to main button.

- Click Push origin at the top to upload the file to your live GitHub repository.


### ⚙️ Step 4: Setting Up GitHub Actions CI/CD

Now that your Sigma rules live inside your private detection-engineering-lab repository, it’s time to introduce automation. Because your Splunk instance runs inside an internal Host‑Only virtual network, GitHub’s cloud runners cannot directly communicate with your VM. Instead, your pipeline will use sigma‑cli to validate rule syntax and compile your Sigma YAML into Splunk‑ready SPL artifacts.

This gives you a fully automated detection‑as‑code workflow without requiring external network access.
Inside your existing detection-engineering-lab/ repository, create the GitHub Actions workflow directory.
Make sure the .github folder begins with a dot: .github/workflows

Paste the following workflow definition: 

<img width="590" height="645" alt="image" src="https://github.com/user-attachments/assets/51434122-3214-4cf5-ae1b-9aa483a629dd" />


Then open GitHub Desktop (or your preferred Git client). You should see:
  - Your new workflow file
  - Any new Sigma rules you added

Then commit and push your changes:
 1. Commit to main
 2. Push origin

Then go to your GitHub repository in the browser, Click the Actions tab and watch your pipeline run automatically

### Step 5: 📥 Retrieve Your Compiled Splunk Queries

Once the github pipeline has compiled your human-readable YAML rules into machine-readable Splunk Search Processing Language (SPL) and saved them as a zip file asset, do the following:

  - Go to your repository on the GitHub website and click on the Actions tab.
  - Click on the most recent, successful workflow run at the top of the list and scroll down to the very bottom of the summary page to find the Artifacts section.
  - Click on compiled-splunk-alerts to download the zip file, then unzip it and open splunk_queries.txt. Inside, you will find your pre-formatted SPL queries mapping straight to your logs. For example, your account discovery rule will look something like this: index="mydfir-project" Image="*\\whoami.exe"


### Part 3 🔧Building the Tines Webhook Endpoint

### Step 1: ⚡Build the Real-Time Alerts inside Splunk

Now, you need to tell your Splunk instance to run these queries continuously in the background.

1. Log into your Splunk Web UI and go to the Search & Reporting app.
2. Take your first compiled query from the text file, paste it into the main search box, and click the search button to confirm it runs cleanly.
3. In the top right corner of the search panel, click Save As and select Alert.
4. Configure the alert engine with these critical parameters:
   - Title: SOC Alert - Local Account Discovery
   - Alert Type: Real-time (or Scheduled to run every 5 minutes if you want to conserve memory resources).
   - Trigger Conditions: Trigger when Number of Results is greater than 0.
  
### 🧲 Step 2: Grab Your Free Tines Webhook

Head over to Tines.com on your host machine and sign up for a free account. Once you're in the dashboard, click Create Story and choose a blank canvas and name it something like Detection Engineering Triage. Now look at the left-hand tool panel, Find the Webhook action icon, drag it, and drop it right into the middle of your canvas. 

The next thing you'll need to do is move your Tines story to your personal team to avoid the following error: AI actions cannot run in a personal team

To do this:

  1. Look at the very top-left corner of your Tines screen. You will see your current team name (it likely says something like Your Name's Personal Team).
  2. Click on that team name dropdown to see your available spaces.
  3. Select a standard team from the list
  4. If your story isn't visible there, pivot back to your Personal Team, open your Story, click the three dots (...) next to the Story name at the top, and select Move to Team.
  5. Choose your standard/shared team space as the destination.

In Splunk, go to Settings > Searches, reports, and alerts. Find your SOC Alert - Local Account Discovery via Whoami alert and click Edit > Edit Alert. Scroll down to your Webhook trigger action and paste your live Tines Webhook URL directly into the box.

Note: make sure you've copied the entire 32-character URL

### 🔨 Part 4: Building the Tines Story 

### Step 1: Drag out a Condition Action

  1. Grab a Condition Action from your left-hand menu and drop it onto the canvas.
  2. Connect your Webhook Action to this new Condition Action.
  3. Rename the node to Condition: Is Whoami Alert.

### Step 2: Configure the Matching Rule

  1. Click on the Condition Action to open the panel on the right.
  2. In the rule configuration, set it up to look for your Splunk alert title:
       - Field to check: webhook.body.search_name
       - Operator: contains
       - Value: Whoami

Repeat these steps for each of your Sigma rules by simply coping the initial node

### Step 3: Add the AI Agent Action

  1. Look at your Tines left-hand menu, find the AI Agent Action, and drag it onto your canvas.
  2. Draw a connecting arrow from your Condition node directly into this new AI Agent Action.
  3. Rename the AI action node
  4. Click on your new AI Triage node and in the right-hand panel, locate the Prompt text area and Paste the following analysis playbook directly into the prompt box:

     <img width="786" height="173" alt="image" src="https://github.com/user-attachments/assets/06aa5630-82a8-43ed-aded-1ce0ffa56347" />

You'll also need to set the temperature to 0.1 

If done correctly, your test should have the following result


<img width="731" height="211" alt="image" src="https://github.com/user-attachments/assets/a152a8d7-4b3e-4388-9e8c-4e534c0fc459" />


### Step 4: Linking Jira

For this part, you'll want to head over to Jira and set up an account. Once done, you'll want to select the Kanban project template.  

 1. Generate the Atlassian API Token in Jira Portal:
      - Log in to your Atlassian account management page at id.atlassian.com/manage-profile/security/api-tokens.
      - Click the Create API token button, give it a clear label like Tines-SOC-Automation, and hit Create.

 2. Copy the generated token string to your clipboard right away. Atlassian will only show this string to you once. If you close the window before copying it, the token is obscured forever and you will have to delete it and start over.
 3. Open Tines and ensure you are working inside your standard team space, Click your team name or settings drop-down menu in the top-left corner and select Credentials, then click New Credential.

In Tines, if you went with the Kanban project template, you'll want to select Jira Software-Create an issue with Jira software 

Your Jira domain, Project Key, and Username fields should look like this:

<img width="796" height="240" alt="image" src="https://github.com/user-attachments/assets/27d75b81-50b0-48ce-b3bb-6012dd94f066" />

Now because we're doing 3 separate AI nodes into jira, you'll need to create 3 separate jira nodes. Towards the bottom of your jira node, you should a field called Summary. Enter the following into the summary field


<img width="533" height="75" alt="image" src="https://github.com/user-attachments/assets/c47e8430-6daa-491a-8281-5dc12055d7c1" />

and repeat for the other 3 nodes, changing the summary based on the Sigma rule condition

Your description field will have the following:

h2. 🤖 AI Analyst Triage Report
*Verdict:* <<ai_agent.output.threat_verdict>>
*Risk Assessment:* <<ai_agent.output.risk_assessment>>
*Remediation Step:* <<ai_agent.output.remediation_step>>

---

h2. 🔍 Raw Telemetry Details
* *Timestamp:* <<webhook.body.result._time>>
* *Affected System:* {code}<<webhook.body.result.host>>{code}
* *User Account:* <<webhook.body.result.User>>
* *Process Image:* {code}<<webhook.body.result.Image>>{code}

h3. 💻 Executed Command Line
{code:bash}
<<webhook.body.result.CommandLine>>
{code}

If it successful, you should see something like this


<img width="975" height="86" alt="image" src="https://github.com/user-attachments/assets/a638c689-213d-4eff-992b-b59abdfcd24d" />

in your jira dashboard with the description looking like this


<img width="975" height="608" alt="image" src="https://github.com/user-attachments/assets/bc177d79-58df-40d3-abf9-0bc1effe834c" />


### Step 5: Building out the Incident Response 

This part is where we take our simple jira ticketing system and transform it into a Incident Response platform

  1. Create the Webhook Listener in Tines
       - On your Tines storyboard, drag a new Webhook Action onto the canvas and name it Jira Incident Response Listener
       - Click on the action to open the right sidebar and copy its unique Webhook URL

  2. Configure Jira to Send the Webhook
       - In Jira, go to Project Settings (or System Settings) $\rightarrow$ Webhooks.
       - Click Create a Webhook and set the URL to the Tines Webhook URL you copied earlier
       - Under Events, choose Issue related events for when Jira should notify Tines 

Note: Right above the event checkboxes in Jira, you'll see a field called JQL (Jira Query Language). If you leave it blank, Jira will ping Tines every single time any ticket in your Jira instance is modified. To prevent spamming your Tines workflow, put this JQL filter            in that box: project = "KAN" AND labels = "auto-isolate"

  3. Build the Containment Filter in Tines
       - Add a Trigger Action to filter out unrelated Jira events and connect it to your Jira Incident Response Listener node
           - Name: Is Containment Requested?

           - <img width="444" height="194" alt="image" src="https://github.com/user-attachments/assets/0bc8feb1-8ed9-437c-81a7-47b5b4b3be53" />

           You'll also need to add the auto-isolate label in Jira 

  4. Execute the Containment Action
       - use an HTTP Request Action in Tines that simulates the EDR API request body
       - In the Payload section, input the following:
           {
  "action": "network_isolation",
  "target_ticket": "=webhook.body.issue.key",
  "initiated_by": "=webhook.body.user.displayName",
  "status": "isolation_requested",
  "timestamp": "=NOW()"
}

  5. Post the Confirmation Back to Jira
       - add a Jira Software-Add issue comment with Jira markdown

       - <img width="385" height="242" alt="image" src="https://github.com/user-attachments/assets/5aa44033-1a9e-4d72-89c6-2212c16d37dd" />

       - For the comment section, add the following formula:

         <img width="478" height="292" alt="image" src="https://github.com/user-attachments/assets/af70c334-2a8a-4849-a29a-e38fc8639ff0" />


If done correctly, you should have the see the following:


<img width="975" height="436" alt="image" src="https://github.com/user-attachments/assets/07bf7e83-d7b1-49a6-8dc4-36b689578f1b" />

and your Tines story should look like this:


<img width="975" height="792" alt="image" src="https://github.com/user-attachments/assets/c9c308a9-765e-4e2f-ae66-856d02898cb1" />
