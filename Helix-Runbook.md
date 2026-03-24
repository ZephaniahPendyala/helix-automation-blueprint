# PROJECT HELIX

**Governed Zero-Touch Infrastructure Automation**

**Implementation Runbook**

**Team RuntimeTerror**

IBM Auto-HackfesT | March 2026

---

## 1. What is Project HELIX?

HELIX is a zero-touch automation pipeline that connects five IBM tools into one closed-loop system. When a server has a problem, HELIX detects it, raises a ticket, checks whether it is allowed to act, scales the resource, generates the code to make the change permanent, and closes every ticket. All of this happens without a single human doing anything.

This document is a step-by-step guide to build this system from scratch. You do not need to be an expert. If you follow each section in order, you will have a working environment at the end.
This document provides a conceptual implementation blueprint designed for real world environments it outlines how existing IBM tools can be integrated into a closed loop automation system.


---

## 2. The Five Tools and What They Do

| Tool | Role in HELIX | In Plain English |
|---|---|---|
| IBM Instana | Observability | Watches your servers every second and fires an alert when something goes wrong. |
| ServiceNow | ITSM Ticketing | Creates and manages incident tickets and change requests automatically. |
| IBM Concert | Orchestration and Governance | Runs the workflow, checks the rules, and makes sure nothing happens without permission. |
| IBM Turbonomic | Resource Scaling | Decides exactly how much memory or CPU a server needs and resizes it automatically. |
| IBM Watsonx | Intelligence and Code Generation | Analyses telemetry to find the right fix, then writes the Ansible code to apply it. |

---

## 3. Before You Start - What You Need

Read through this entire section before touching any servers. Missing even one item here will cause a failure somewhere in the middle of the setup, and tracing it back is painful.

### 3.1 Server Requirements

| Tool | OS | CPU | RAM | Disk | Notes |
|---|---|---|---|---|---|
| Instana Backend | RHEL / Ubuntu / SUSE | 8 cores | 32 GB | 500 GB | DNS must be set up for the FQDN before you run the installer. |
| Turbonomic | VMware vCenter (OVA) | 16 cores | 160 GB | 1.80 TB | /var partition needs at least 340 GB on its own. This is not optional. |
| Concert | Linux x86_64 | 16 cores | 32 GB | 512 GB | Production environments need 32 cores, 64 GB RAM, and 1 TB disk. |
| ServiceNow | Cloud (SaaS) | N/A | N/A | N/A | A developer or sandbox instance is fine for testing. |
| Watsonx.ai | IBM Cloud (SaaS) | N/A | N/A | N/A | You need an IAM API key and a Project ID from your IBM Cloud account. |

### 3.2 Software Versions

| Software | Version Required | Why It Matters |
|---|---|---|
| Turbonomic | 8.19.x | Older versions do not support the REST API scheduling used in Step 5. |
| Concert | 2.3.x | The Delay block syntax and FaaS Ansible blocks only exist from version 2.3 onwards. |
| Instana Agent | 1.0.315 or later | This is the first release that lets you run the agent as a non-root user. |
| MariaDB (for Turbonomic) | 10.11.15 | Older versions will cause the Turbonomic installer to stop with an error. |
| MySQL (alternative) | 8.4 only | MySQL 8.0 is completely deprecated in the 2026 release and will not work. |
| ServiceNow | Xanadu release | The OAuth integration with Instana needs this release or newer. |
| Watsonx AI Model | IBM Granite foundation models | IBM's recommended enterprise reasoning models for 2026. |

### 3.3 Accounts and Access Tokens

Collect all of these before starting. You will be pasting them into config files throughout the setup process.

- IBM Cloud account with watsonx.ai enabled
- IBM Cloud IAM API Key - generate this from cloud.ibm.com under Manage > Access > API Keys
- Watsonx Project ID - found inside your watsonx.ai project settings
- Watson Machine Learning service instance provisioned and linked to your project
- ServiceNow instance URL (looks like https://mycompany.service-now.com)
- ServiceNow integration user with evt_mgmt_integration and itil roles
- ServiceNow OAuth Client ID and Client Secret - from System OAuth > Application Registry

> **Write these down somewhere safe.** You will need them at Steps 2, 3, 4, and 8. Losing them halfway through means starting over.

---

## 4. Setting Up the Environment

Follow each tool setup in the order listed here. Each one depends on the previous being ready.

### 4.1 Setting Up IBM Instana

Instana has two parts - the backend server where all the data lands, and agents that sit on the servers you want to monitor.

#### Step A - Install the Instana Backend

On your Instana server, run the installation script. It will ask for three things:

- core-base-domain: your company base domain, for example yourcompany.com
- unit-tenant-name: a name for your tenant, for example helix
- unit-unit-name: a name for this unit, for example prod

The installer joins all three together to build the full web address (FQDN) you will use to log in. For example: helix.prod.yourcompany.com

> **Sort DNS first.** Update your company DNS to point to the Instana server before running the installer. If DNS is not ready, the dashboard will be unreachable after installation and you will need to troubleshoot networking instead of moving forward.

#### Step B - Install the Host Agent on Each Server

For every server you want to monitor, install the Instana agent. On Red Hat or CentOS:

```bash
yum install instana-agent-dynamic
```

The dynamic agent updates itself when IBM releases new versions. This is the recommended choice for most environments.

After installation, open the config file and add your backend URL and agent key:

```
instanaAgentDir/etc/instana/com.instana.agent.main.sender.Backend.cfg
```

> **Security note:** From Instana version 1.0.315 onwards, you can run the agent as a non-root user on Linux. This is worth doing - if something ever compromises the monitoring agent, it cannot take over the host server.

---

### 4.2 Setting Up IBM Turbonomic

Turbonomic is deployed as a virtual machine image. Download the OVA file from IBM and import it into VMware vCenter.

#### Step A - Hardware Check

Before importing the OVA, run this on the physical host:

```bash
cat /proc/cpuinfo | grep sse4
```

If that command returns nothing, the server does not support SSE4.2 and Turbonomic will not install on it. You need different hardware.

#### Step B - Disk Partitioning

When setting up the VM storage, make sure the /var partition gets at least 340 GB by itself. Do not divide the total 1.80 TB evenly across partitions.

> **Why /var matters:** Turbonomic runs on containers internally. When it updates, it downloads new container images into /var. If this partition is too small, the update crashes and Kubernetes throws an ImagePullBackOff error. The only way out is to rebuild the VM with more space. Save yourself the trouble.

#### Step C - Database Setup

Turbonomic needs an external database. Install MariaDB 10.11.15 or MySQL 8.4. Do not use any older version. MySQL 8.0 specifically is not supported and the installer will check and stop if it finds it.

---

### 4.3 Setting Up IBM Concert

Concert is the engine that coordinates everything. Think of it as the traffic controller between all the other tools.

#### Step A - Create a Non-Root User

You must run Concert under a dedicated user account, not root. Create one first:

```bash
useradd -m concertuser
```

#### Step B - Fix the Podman Logout Problem

Concert uses Podman to run its containers. There is a known issue: when the terminal session ends, Podman kills background processes including Concert. Run this command before starting the installation:

```bash
loginctl enable-linger concertuser
```

This tells the operating system to keep the user's background processes alive after logout. If you skip it, Concert will stop every time you close your laptop.

#### Step C - Use Netavark Networking

Check that your container networking stack is using Netavark rather than the old CNI framework. CNI has been deprecated. On a fresh server with a recent Linux version this is already the default, so you may not need to do anything here.

#### Step D - Run the Concert Installer

Run the setup script as your concertuser. You will be asked for your IBM entitlement API key to pull the container images from IBM's registry.

> **SSL certificates:** For production environments, put your tls.crt and tls.key files into localstorage/volumes/infra/tls/external-tls/ before running the installer. If you add them after, you will need to restart services.

---

### 4.4 Connecting Watsonx.ai

Watsonx.ai is a cloud service. You do not install it locally - you connect Concert to it.

1. Log into cloud.ibm.com
2. Go to Manage > Access (IAM) > API Keys and create a new key. Copy it immediately - you will not see it again.
3. Open your watsonx.ai project and copy the Project ID from project settings.
4. Provision a Watson Machine Learning service instance and link it to the project.
5. In Concert, add these as environment variables: WATSONX_API_KEY and WATSONX_PROJECT_ID

> **Model used:** HELIX uses IBM Granite foundation models for AI analysis. These are IBM's enterprise reasoning models built for infrastructure use cases. For very complex analysis, the system can also call meta-llama/llama-3-3-70b-instruct as an alternative.

---

### 4.5 Setting Up ServiceNow

You need a ServiceNow instance and a dedicated robot user account for the HELIX integration to work.

#### Step A - Create the Integration User

1. In ServiceNow, go to User Management and create a new user. Something like helix-integration works fine as the name.
2. Assign two roles to this user: evt_mgmt_integration and itil.

> **The evt_mgmt_integration role is not optional.** Without it, ServiceNow will silently discard every alert coming from Instana. No error message appears - tickets just do not get created. This is the most common setup mistake people make.

#### Step B - Set Up OAuth

1. Go to System OAuth > Application Registry in ServiceNow.
2. Create a new OAuth application and save the Client ID and Client Secret.
3. You will use these when configuring the Instana alert channel in Step 2 of the workflow.

---

## 5. The 11-Step HELIX Workflow

Once all five tools are installed and connected, the automation runs on its own. Below is what happens during an incident, and what you configured to make each step work.

| Step | Tool | What Happens |
|---|---|---|
| 01 | Instana | Smart Alert fires on SLO breach. Anomaly detected in under 1 second. |
| 02 | Instana to ServiceNow | Alert channel fires - P1 incident auto-created in ServiceNow. |
| 03 | ServiceNow to Concert | ServiceNow webhook triggers Concert workflow automatically. |
| 04 | Concert | Guardrails checked - cost limits, policy boundaries, compliance rules. |
| 05A | Turbonomic | VM scaling executed automatically using analytics-driven decisions. |
| 05B | Concert | 24-hour lock enforced before any scale-down (governance path). |
| 06 | Concert to ServiceNow | Incident updated with diagnostic logs and team routing. |
| 07 | Concert to ServiceNow | Formal Change Request auto-generated with full audit trail. |
| 08 | Concert to Watsonx AI | Live Instana telemetry sent to AI for optimal scaling recommendation. |
| 09 | ServiceNow to Stakeholders | AI recommendation attached to CR - approval gate (optional). |
| 10 | Watsonx Code + Concert FaaS | Ansible playbook generated by AI and executed by Concert. |
| 11 | Concert + Instana + ServiceNow | VM verified stable - all tickets closed - stakeholders notified. |

---

### Step 01 - Anomaly Detection (Instana)

Instana monitors every server continuously. When something genuinely abnormal happens - say a server sitting at 94% CPU for several minutes - it fires a Smart Alert.

To configure this, go to Events & Alerts > Smart Alerts in Instana. Create a new Smart Alert based on a Service Level Objective. Set the thresholds for your environment - trigger only when the error budget is being consumed too fast, not on short spikes.

- Instana uses adaptive thresholds. It learns your baseline over time and only alerts on genuine problems.
- From the March 2026 release, you can set alert windows based on calendar months, which lines up with your cloud billing.

> **Keep the threshold sensible.** A single 30-second spike should not kick off the entire pipeline. Set it to sustained violations only - something like CPU above 85% for more than 5 minutes consistently.

---

### Step 02 - Auto-Create ServiceNow Incident (Instana Alert Channel)

Once the alert fires, Instana sends the details straight to ServiceNow through an Alert Channel.

In Instana, go to Settings > Events & Alerts > Alert Channels. Select ServiceNow as the channel type. Enter the instance URL, the integration user credentials, and enable OAuth. Every time an alert fires, a ticket is created automatically with:

- The exact time the problem started
- Severity level (5 for warning, 10 for critical - ServiceNow converts this to P1/P2/P3)
- A description of what went wrong
- Instana's initial read on the possible cause

> **ServiceNow webhook URL for Xanadu release:** https://yourinstance.service-now.com/api/sn_em_connector/em/inbound_event?source=instana

---

### Step 03 - Trigger Concert Workflow (ServiceNow to Concert)

ServiceNow has the incident. Now it needs to hand it off to Concert so the automation can begin.

In ServiceNow, open Flow Designer and create a new flow. When a new incident is created, the flow sends an HTTP POST to Concert's API gateway. This is an outbound REST message. The body contains the server name, domain, and severity score in JSON. The moment the ticket exists, Concert receives the data and wakes up.

---

### Step 04 - Validate Guardrails (Concert)

Before Concert touches anything, it checks the rules. This is the most important governance step in the pipeline.

Concert checks the Resilience Profile you set up during configuration. This contains rules like:

- Maximum memory limit for a given server group
- Cost ceiling - if the scaling action would push cloud costs above a set amount, stop
- Compliance boundaries - certain servers need extra approval before any changes

If everything checks out, Concert proceeds. If any check fails, it stops the workflow entirely. The ticket gets updated with the reason, and nothing is changed on any server.

> **Think of Concert as the person who checks the budget.** Just because a server is struggling does not mean spending money on it is approved. Concert enforces the financial and policy boundaries before any resource is touched.

---

### Step 05 - Execute Scaling (Two Options)

If guardrails pass, scaling begins. There are two approaches depending on how much control your organisation wants.

#### Option A - Intelligent Elasticity (Turbonomic)

In Turbonomic, create an automation policy called Auto-Resize-Policy. Set vCPU Resize Up and vMEM Resize Up actions to Automatic. Turbonomic scales the server immediately based on 30 days of usage history.

To stop it from scaling back down too fast, use the Turbonomic REST API to create a Policy Enforcement Schedule. This lets Turbonomic scale up now, but prevents any scale-down for 24 hours.

#### Option B - Strict Procedural Control (Concert)

If your organisation needs deterministic timing, use Concert to manage the reversion. Turbonomic does the scale-up, but Concert controls when things scale back down.

In the Concert workflow editor, add a Delay block after the scale-up step. Use this expression:

```
addDays(Entry_time, 1)
```

This pauses the workflow for exactly 24 hours. After that, Concert checks for any required approvals and executes the scale-down.

> **Why the 24-hour lock matters:** Without it, the system can end up in a loop - scale up, traffic drops, scale down, traffic spikes, scale up again. This destroys application stability. Holding resources for a full day gives the environment time to find its new normal.

---

### Step 06 - Update the Incident (Concert to ServiceNow)

While scaling is happening, Concert updates the original ServiceNow incident. It adds work notes confirming the guardrail checks passed and the server has been scaled. It also routes the ticket to the right engineering team queue so they have visibility without needing to do anything.

---

### Step 07 - Generate a Change Request (Concert to ServiceNow)

Changing a production server's CPU or memory is a formal infrastructure change. Enterprise practice requires a Change Request for every one. Historically this meant an engineer filling out a form at some point, often after the fact.

Concert handles this automatically. It fires another API request to the ServiceNow Change Management module and creates a fully filled-out Change Request with the server name, the reason, the resilience score, and a complete audit trail. Nobody types anything.

---

### Step 08 - AI Analysis (Concert to Watsonx AI)

The immediate problem has been handled. This step is about working out the right permanent fix, not just the emergency response.

Concert calls the Instana API to pull 30 days of historical telemetry for the affected server. It packages this into a prompt and sends it to Watsonx AI using the ask-wx service framework. The IBM Granite foundation model analyses the data and returns a recommendation in plain English - for example: scale this server to 8 vCPU and 16 GB RAM permanently.

> **Data privacy:** The telemetry sent to Watsonx AI is completely temporary. Once the AI returns its answer, the data is removed. Your infrastructure details are never stored in IBM's public training models.

---

### Step 09 - Stakeholder Approval Gate (Optional)

The AI recommendation lands directly on the ServiceNow Change Request. ServiceNow notifies the Change Advisory Board or the application owner.

For most environments this step runs automatically without needing anyone to click anything. If a customer specifically requires sign-off before permanent changes, the stakeholder opens ServiceNow, reads the recommendation in plain English, and either approves it or adjusts the numbers. When they approve, Concert gets the signal and carries on.

In Turbonomic, the pending action sits in a WAITING_FOR_CR_SCHEDULE or WAITING_FOR_EXEC state until approval comes through.

---

### Step 10 - Generate and Execute Ansible Code (Watsonx Code Assistant + Concert FaaS)

Approval is in. Now the actual infrastructure change needs to happen. Watsonx Code Assistant writes the Ansible playbook to do it.

Concert passes the approved recommendation to Watsonx Code Assistant. The AI generates a YAML Ansible playbook - no human writes a single line. Concert then runs this playbook using its built-in Function-as-a-Service (FaaS) engine.

| FaaS Block | What It Does | When to Use It |
|---|---|---|
| Ansible Playbook | Runs a YAML playbook directly against a server inventory. | Most HELIX scaling scenarios. Fastest option. |
| Ansible Pull | Fetches a playbook from a Git repository before running it. | When you want the generated code stored in version control. |
| Ansible Builder | Creates an isolated container to run the playbook. | Complex playbooks with special dependencies. |
| Terraform FaaS | Provisions new cloud infrastructure. | When scaling means creating new VMs rather than resizing existing ones. |

> **Istio limitation:** If your Concert deployment has Istio service mesh enabled, FaaS blocks are disabled by design. In that case, the playbook needs to be executed outside Concert.

---

### Step 11 - Close the Loop (Concert to Instana to ServiceNow)

The playbook has run. The last job is to verify it worked and clear all the open tickets.

1. Concert calls Instana and checks the server metrics. If CPU and memory are back to normal, it proceeds.
2. Concert closes the original incident ticket in ServiceNow and marks it Resolved.
3. Concert closes the Change Request and marks it Closed.
4. Concert sends a notification to the team's Slack or Microsoft Teams channel. The message is simple - problem detected, budget checked, server fixed, tickets closed.
5. If the verification fails, a high priority incident is raised and routed to the team for manual handling.

> **This is the moment the loop closes.** The server room is quiet. The on-call phone never rang. HELIX handled it from start to finish.

---

## 6. Common Problems and How to Fix Them

| Problem | Likely Cause | Fix |
|---|---|---|
| Instana dashboard not reachable after install | DNS was not configured before installation | Update DNS to point to the Instana backend FQDN, then restart the backend service. |
| Turbonomic install fails with SSE4.2 error | Server hardware is too old | Run: cat /proc/cpuinfo \| grep sse4. If blank, you need a newer server. |
| Turbonomic fails to update - ImagePullBackOff error | /var partition is too small | Rebuild the VM with at least 340 GB dedicated to /var. |
| Concert containers stop when you log out | loginctl enable-linger was not run | Run: loginctl enable-linger \<username\> then restart Concert. |
| ServiceNow tickets not created when Instana alerts fire | evt_mgmt_integration role is missing | Assign the evt_mgmt_integration role to the ServiceNow integration user. |
| Watsonx AI returns no recommendation | API key or Project ID is wrong | Check that WATSONX_API_KEY and WATSONX_PROJECT_ID are correctly set in Concert config. |
| Concert workflow stops at Step 4 | Guardrail policy is blocking the action | Review the Resilience Profile in Concert. Cost thresholds may be set too low for the requested change. |

---

## 7. Quick Reference

### 7.1 Full 11-Step Summary

| Step | Tool | Action | Output |
|---|---|---|---|
| 01 | Instana | Smart Alert fires on SLO breach | Alert triggered |
| 02 | Instana + ServiceNow | Alert channel routes to ServiceNow | P1 Incident created |
| 03 | ServiceNow + Concert | Webhook triggers Concert workflow | Workflow running |
| 04 | Concert | Guardrails validated | Green light to proceed |
| 05A | Turbonomic | VM scaling executed | Resources increased |
| 05B | Concert | 24-hour lock enforced | Scale-down prevented |
| 06 | Concert + ServiceNow | Incident updated with logs | Team notified |
| 07 | Concert + ServiceNow | Change Request auto-generated | Audit trail created |
| 08 | Concert + Watsonx AI | Telemetry analysed by Granite LLM | Recommendation generated |
| 09 | ServiceNow | Stakeholder approval gate | Change approved |
| 10 | Watsonx Code + Concert FaaS | Ansible playbook generated and executed | Infrastructure changed |
| 11 | Concert + Instana + ServiceNow | Verified and all tickets closed | Loop closed |

### 7.2 Key Configuration Values

| Item | Value or Location |
|---|---|
| Instana Agent Config File | instanaAgentDir/etc/instana/com.instana.agent.main.sender.Backend.cfg |
| ServiceNow Instana Webhook URL | https://\<instance\>.service-now.com/api/sn_em_connector/em/inbound_event?source=instana |
| Watsonx AI Model | IBM Granite foundation models |
| Concert 24hr Delay Expression | addDays(Entry_time, 1) |
| Concert Setup Script | ibm-concert-std/bin/setup |
| Turbonomic /var Minimum Size | 340 GB |
| Turbonomic Total Disk Minimum | 1.80 TB |
| Turbonomic RAM Minimum | 160 GB (96 GB acceptable for under 10,000 VMs) |
| MariaDB Version Required | 10.11.15 |
| MySQL Version Required | 8.4 (not 8.0) |

---

*Project HELIX | Team Runtime Terror | IBM Auto-HackfesT 2026*
