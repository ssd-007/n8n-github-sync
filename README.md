# n8n Workflow Backups

This repository is a self-maintaining backup of production n8n automation workflows. Files are pushed automatically from a live n8n instance on a daily schedule — no manual commits are required to keep this repo up to date.

---

## How Sync Works

A dedicated n8n workflow (`n8n-github-sync`) handles all synchronization. It runs at 4:00 AM daily and performs the following operations:

1. **Fetch** — Queries the n8n REST API to retrieve all active workflows from the live instance.
2. **Check** — For each workflow, calls the GitHub Contents API to determine whether a corresponding JSON file already exists in the repository.
3. **Create or Update** — If a file doesn't exist, it is created. If it already exists, it is updated with the latest version from the live instance.
4. **Notify** — After all workflows are processed, a single Telegram message is sent summarising what was created and what was updated in the run.

---

## Workflows in This Repository

This repository is powered by `n8n-github-sync`, the workflow responsible for all automated commits to this repo. Additional workflows are added as they reach production quality.

---

## Importing a Workflow

1. Open your n8n instance.
2. Navigate to **Workflows → Import from File**.
3. Select the desired `.json` file from this repository.
4. Review credentials and environment-specific settings before activating.
