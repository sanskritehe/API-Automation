# AI-Assisted API Automation Pipeline 

An end-to-end pipeline that automatically transforms Jira tickets into complete, verified GitHub Pull Requests.

Given a ticket, the orchestrator fetches the API specification from Confluence, constructs a tailored system prompt, runs a GitHub Copilot (via GitHub Models API) Generate-and-Judge loop, writes the implementation files directly, commits them to a feature branch, opens a PR, and updates the Jira ticket.

---

## The Workflow

```
       Jira Ticket Assigned
                │
                ▼
  Fetch Ticket & Confluence API Spec
                │
                ▼
       Generate Prompt & Code
                │
                ▼
       ┌────────────────┐
  ┌───►│  Copilot Judge │
  │    └────────┬───────┘
  │             │
  │ Rejected    │ Approved
  └─ Feedback   ▼
          Create Branch
                │
                ▼
       Commit Code & Open PR
                │
                ▼
   Post PR Links to Jira Ticket
```

---

## Features

- **Automated Spec Fetching**: Integrates with Jira and Confluence to extract issue descriptions and API designs automatically.
- **Generate & Judge Loop**: Leverages GitHub Copilot (`gpt-4o`) to write code, which is then evaluated by a secondary judge loop (up to 3 iterations) for safety and specification compliance.
- **Multi-Repo Operations**: Scans keyword triggers to target and commit code to multiple repositories simultaneously (e.g., API gateway, database service).
- **Webhook Integration**: Includes a lightweight FastAPI webhook server to trigger the pipeline instantly when tickets are assigned.

---

## Quick Start

### 1. Prerequisites
- **Python**: 3.10+
- **GitHub Account**: Access to GitHub Copilot / Models API (using `GITHUB_TOKEN` or `COPILOT_GITHUB_TOKEN`).
- **Jira & Confluence**: Atlassian account with API Token access.

### 2. Installation
```bash
git clone https://github.com/your-org/api-automation.git
cd api-automation/orchestrator

# Set up virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### 3. Configuration (`.env`)
Create a `.env` file in the `orchestrator/` folder using `.env.example` as a template:

```env
# Atlassian (Jira/Confluence) Credentials
JIRA_DOMAIN=https://your-org.atlassian.net
JIRA_EMAIL=your-email@example.com
JIRA_API_TOKEN=your-atlassian-api-token

CONFLUENCE_DOMAIN=https://your-org.atlassian.net
CONFLUENCE_EMAIL=your-email@example.com
CONFLUENCE_API_TOKEN=your-atlassian-api-token

# Confluence Space and Default Page for Webhook Mode
CONFLUENCE_SPACE=your-space-key
CONFLUENCE_PAGE=Your API Spec Page Title

# GitHub Credentials
GITHUB_TOKEN=your-github-pat
BASE_BRANCH=main

# GitHub Copilot & Models Configuration
COPILOT_GITHUB_TOKEN=your-copilot-pat  # Leave blank to fallback to GITHUB_TOKEN
GENERATOR_MODEL=gpt-4o
JUDGE_MODEL=gpt-4o

# Webhook Assignee Filter (Optional)
COPILOT_ASSIGNEE=automation-bot@company.com
```

### 4. Service Routing (`service_groups.json`)
Configure `service_groups.json` to map Jira issue keywords (e.g., `appointment`) to target GitHub repositories:

```json
{
  "appointment": [
    {
      "repo": "your-org/Appointment-Service",
      "role": "api",
      "operations": ["GET", "POST", "PUT", "PATCH", "DELETE"]
    }
  ]
}
```

---

## Running the Pipeline

### Manual Execution (Full Pipeline)
Run the pipeline directly from the command line by targeting a specific Jira issue:
```bash
python pipeline.py \
  --ticket KAN-25 \
  --confluence-space "DEV" \
  --confluence-page "Appointment Service API Spec"
```

### Run the Webhook Server
Start the FastAPI server to process webhooks from Jira automatically upon assignment:
```bash
uvicorn webhook_server:app --host 0.0.0.0 --port 8000
```
> **Tip**: Expose the server to the internet during development using `ngrok http 8000` and register the webhook URL (`https://<ngrok-subdomain>.ngrok-free.app/webhook`) in **Jira Settings → System → Webhooks**.

### Offline Test (Copilot Generation Loop Only)
If `prompt.md` already exists locally in the `orchestrator/` directory, you can test the Copilot generation loop in isolation:
```bash
python orchestrator.py
```

---

## Production Deployment
To run the webhook server persistently, choose one of the following setups:

- **Option A: systemd Service**: Run uvicorn as a system service in Linux and reverse-proxy it using **Nginx** or **Caddy** to handle TLS/SSL certificates automatically.
- **Option B: Docker Compose**: Build and run using the provided `docker-compose.yml` configuration:
  ```bash
  docker compose up -d
  ```

---

## Troubleshooting
- **`ignored: no assignee set`**: Check that the Jira ticket is assigned.
- **`ignored: assignee is not the copilot agent`**: Check that the ticket assignee matches `COPILOT_ASSIGNEE` in `.env`.
- **`ignored: no matching service keyword found`**: Ensure a keyword from `service_groups.json` is present in the Jira issue's title or description.
- **Rate Limits (HTTP 429)**: Ensure your GitHub Copilot subscription token is valid and limit requests to match the GitHub Models API quota.
