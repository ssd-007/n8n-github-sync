# n8n Workflow Backups

This repository is a self-maintaining backup of production n8n automation workflows. Files are pushed automatically from a live n8n instance on a daily schedule — no manual commits are required to keep this repo up to date.

---

## How Sync Works

A dedicated n8n workflow (`n8n-github-sync`) handles all synchronization. It runs at 4:00 AM daily and performs the following operations:

1. **Fetch** — Queries the n8n REST API to retrieve all active workflows from the live instance.
2. **Check** — For each workflow, calls the GitHub Contents API to determine if a corresponding JSON file already exists in the repository, comparing SHA hashes to detect changes.
3. **Create or Update** — If a file doesn't exist, it is created. If a file exists but the SHA differs (i.e., the workflow has changed), the file is updated. Unchanged files are skipped.
4. **Notify** — After all workflows are processed, a single Telegram message is sent summarizing what was created and what was updated in the run.

This approach is idempotent — running the workflow multiple times will not produce duplicate commits or false updates.

---

## Workflows in This Repository

| File | Description |
|------|-------------|
| `n8n-github-sync.json` | The sync workflow itself. Fetches all active workflows from the n8n instance and pushes changes to this repository daily. |

> Additional workflows will be added as they reach production quality.

---

## Tech Stack

| Component | Role |
|-----------|------|
| **n8n** (self-hosted) | Automation platform running all workflows |
| **n8n REST API** | Source of truth for fetching live workflow definitions |
| **GitHub Contents API** | File creation and updates with SHA-based conflict detection |
| **Telegram Bot API** | Post-run summary notifications |
| **JavaScript** (n8n Code node) | Builds the notification payload from processed workflow data |

---

## Repository Structure
```
/
├── n8n-github-sync.json   # GitHub sync workflow
└── README.md
```

Each workflow is exported as a raw JSON file using n8n's native export format and can be imported directly into any n8n instance.

---

## Importing a Workflow

1. Open your n8n instance.
2. Navigate to **Workflows → Import from File**.
3. Select the desired `.json` file from this repository.
4. Review credentials and environment-specific settings before activating.
