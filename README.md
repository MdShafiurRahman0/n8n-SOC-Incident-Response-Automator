
# Interactive SOC Automation: Wazuh to Slack to Jira via n8n

## 📌 Overview
This project implements an advanced Security Orchestration, Automation, and Response (SOAR) workflow with a "Human-in-the-loop" approval process. 
Security alerts triggered in **Wazuh** are sent to an **n8n** webhook. n8n then formats and sends an interactive message to **Slack** with "Approve" and "Decline" buttons. If a SOC engineer clicks "Approve", n8n automatically creates a High-Priority ticket in **Jira** for immediate incident response.

## 🚀 Prerequisites
* **Docker** installed.
* **Wazuh Manager** running.
* **Slack Workspace** with permissions to create apps and use interactivity.
* **Jira Software** account with an API token.

---

## 🛠️ Step-by-Step Implementation Guide

### Step 1: n8n Docker Setup
Run n8n locally using Docker. Execute the following command in your terminal:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -e N8N_ENFORCEMENT_AGREEMENT=true \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

### Step 2: Configure Slack App for Interactivity
To use buttons in Slack, you need to enable Interactivity in your Slack App.

1. Go to the [Slack API Dashboard](https://api.slack.com/apps) and create an app (e.g., `SOC-Alert-Bot`).
2. Add the `chat:write` scope under **OAuth & Permissions** and install the app to your workspace.
3. Copy the **Bot User OAuth Token** (`xoxb-...`).
4. Go to **Interactivity & Shortcuts** in the left menu and toggle it **ON**.
5. You will need to paste an **Interactivity Request URL** here (we will get this from n8n in Step 4).

### Step 3: Configure Jira API Access
1. Log into your Atlassian/Jira account.
2. Go to **Account Settings > Security > Create and manage API tokens**.
3. Create a new API token and copy it. Save your Jira email address and this token for n8n credentials.

### Step 4: Build the n8n Workflow
Open n8n at `http://localhost:5678` and create the following pipeline:

**1. Primary Webhook (From Wazuh):**
* Add a **Webhook** node. Method: `POST`. Copy the Test URL.

**2. Slack Node (Send Interactive Message):**
* Connect a **Slack** node. Authenticate using your Bot Token.
* Set Resource: `Message`, Operation: `Post`.
* In the **Blocks** (Block Kit) section, configure it to send the alert details and two interactive buttons: "Approve" (value: `approved`) and "Decline" (value: `declined`).

**3. Secondary Webhook (Slack Interactivity Callback):**
* Add a second **Webhook** node. Method: `POST`. 
* Copy this webhook's URL and paste it into the **Interactivity Request URL** field in your Slack App settings (from Step 2).

**4. IF Node (Decision Logic):**
* Connect an **IF** node after the Secondary Webhook.
* Condition: Check if the incoming payload from Slack's button click equals `approved`.

**5. Jira Node (Create Ticket):**
* Connect a **Jira Software** node to the `True` output of the IF node.
* Authenticate using your Jira email and API token.
* Resource: `Issue`, Operation: `Create`.
* Set Project Key, Issue Type (e.g., `Bug` or `Task`), Priority (`High`), and map the Wazuh alert details to the Summary and Description fields.

### Step 5: Configure Wazuh to Trigger n8n
Add the following integration block to your Wazuh Manager's `ossec.conf` file to forward alerts (Level 7 and above) to your Primary Webhook.

```xml
<integration>
  <name>custom-n8n</name>
  <hook_url>http://YOUR_N8N_IP:5678/webhook/YOUR_PRIMARY_WEBHOOK_ID</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```
*Restart Wazuh manager to apply changes.*

### Step 6: Test the Pipeline
Send a test cURL request to your primary n8n webhook to simulate a Wazuh alert:

```bash
curl -X POST http://localhost:5678/webhook-test/YOUR_PRIMARY_WEBHOOK_ID \
     -H "Content-Type: application/json" \
     -d '{
           "rule": {"description": "Multiple authentication failures", "level": 10},
           "agent": {"name": "Database-Server"}
         }'
```
You should receive a Slack message with buttons. Clicking "Approve" will trigger the secondary webhook and create a Jira ticket!

---

## 👨‍💻 Author
**Md. Shafiur Rahman**
* GitHub: [@MdShafiurRahman0](https://github.com/MdShafiurRahman0)
* Email: shafiur.rahman.shadat@gmail.com
```
