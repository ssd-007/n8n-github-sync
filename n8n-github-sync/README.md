# n8n GitHub Sync Workflow

Backs up all active n8n workflows to this repository daily.

---

## What It Does

Every day at 4am, this workflow fetches all active workflows from the n8n instance and saves each one as `workflow.json` in its own folder. If the file already exists it updates it; if not, it creates it. A Telegram message summarises the run at the end.

---

## Flow

```
Schedule Trigger (4am) + Manual Trigger
          ↓
   n8n Workflows node       ← get all active workflows
          ↓
     Set Variables          ← build filePath and fileContent for each workflow
          ↓
  HTTP Request              ← call GitHub Contents API to get file SHA
          ↓
         IF                 ← does the file already exist?
        /    \
  Edit File  Create File
        \    /
         Merge
          ↓
      Summarise (Code)      ← build one summary string across all items
          ↓
       Telegram             ← send summary to private chat
```

---

## Nodes

- **Schedule Trigger** — runs at 4am daily; Manual Trigger allows on-demand runs
- **n8n Workflows node** — fetches all active workflows (`returnAll: true`, `activeOnly: true`)
- **Set Variables** — extracts `workflowName`, builds `filePath` and base64 `fileContent`
- **HTTP Request** — calls the GitHub Contents API to retrieve the file's current `sha` (`continueOnFail: true` so 404s on new files don't stop the workflow)
- **IF node** — condition `={{ !!$json.sha }}` routes to Edit or Create
- **Edit File / Create File** — GitHub nodes that commit the workflow JSON; Edit passes `sha`, Create does not
- **Set Status** — tags each item as `"updated"` or `"created"` before merging
- **Merge** — rejoins both branches so one Telegram message covers all workflows
- **Summarise** — JavaScript Code node that builds the summary (updated list, created list, total count)
- **Telegram** — sends the summary to a private chat via a dedicated bot

---

## Credentials

All credentials are stored in the n8n credential store — no keys are in the workflow JSON.

| Service  | Used By                              |
|----------|--------------------------------------|
| n8n API  | n8n Workflows node                   |
| GitHub   | HTTP Request, Edit File, Create File |
| Telegram | Telegram node                        |

---

## Why This Was Built

Portfolio piece demonstrating scheduled triggers, API integration, conditional branching, and notification delivery. The repo is public and serves as a live example of automated workflow version control.
