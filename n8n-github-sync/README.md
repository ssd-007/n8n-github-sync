# n8n GitHub Sync Workflow

**Workflow ID:** `bhWqdvOJPFtWzBiJ`
**Instance:** https://n8n.paracras.com
**Repo:** [ssd-007/n8n-workflows](https://github.com/ssd-007/n8n-workflows)
**Status:** Active — runs daily at 4am

---

## What It Does

Every day at 4am, this workflow:

1. Fetches all **active (published)** workflows from the n8n instance via the n8n API
2. For each workflow, checks whether a file already exists in this GitHub repo
3. If it exists — **updates** the file (Edit File)
4. If it doesn't — **creates** the file (Create File)
5. Sends a single **Telegram summary** listing what was updated, what was created, and how many workflows were processed

Each workflow is stored as `workflow.json` inside its own folder, named after the workflow (e.g., `github-sync-workflow/workflow.json`).

---

## Architecture

```
Schedule Trigger (4am daily)
          │
    Manual Trigger
          │
          ▼
   n8n Workflows node          ← fetches all active workflows from n8n API
          │
          ▼
     Set Variables             ← shapes each item: workflowName, filePath, fileContent
          │
          ▼
  Get File SHA (HTTP Request)  ← calls GitHub Contents API to get current SHA (if file exists)
          │
          ▼
         IF                    ← {{ !!$json.sha }} — does the file already exist?
        /    \
      Yes     No
       │       │
  Edit File  Create File       ← GitHub nodes — update or create workflow.json
       │       │
  Set Status  Set Status       ← tags each item as "updated" or "created"
        \     /
         Merge
          │
          ▼
      Summarise                ← JavaScript Code node — builds one summary string
          │
          ▼
       Telegram                ← sends the summary to a private chat
```

---

## Node-by-Node Breakdown

### Schedule Trigger
- **Why:** The workflow runs daily without any manual intervention. 4am was chosen to avoid overlap with business-hours usage of the n8n instance.
- **Config:** `cronExpression: 0 4 * * *`, timezone set to the instance default.

### Manual Trigger
- **Why:** Allows running the backup on demand without waiting for the next scheduled run — useful after making significant workflow changes.
- Both triggers feed into the same node downstream (the n8n Workflows node), so no duplication.

### n8n Workflows Node
- **Why:** The n8n node has a built-in "Get Many Workflows" operation that returns all workflow data including JSON definition, name, and active status. This is cleaner than a raw HTTP Request to the n8n API for this step.
- **Config:** `returnAll: true`, `activeOnly: true` — no name-based filtering needed since we want every active workflow backed up.
- A Filter node was considered here and later removed: once `activeOnly` was set, there was nothing left to filter.

### Set Variables
- **Why:** The data coming out of the n8n Workflows node is a full workflow object. This node explicitly shapes each item into the three values needed downstream:
  - `workflowName` — used to build the folder/file path
  - `filePath` — `workflows/{workflowName}/workflow.json`
  - `fileContent` — the full workflow JSON, base64-encoded for the GitHub API
- Shaping data early at a Set node makes every downstream node simpler and less error-prone.

### Get File SHA — HTTP Request node
- **Why this node, not the GitHub "Get File" node:**
  The n8n GitHub node's `Get File` operation returns file content as **binary output only**. The JSON output is unchanged from the upstream item — it does not include any file metadata. The `sha` field that GitHub requires to perform an update is simply not accessible.

  The GitHub Contents API (`GET /repos/{owner}/{repo}/contents/{path}`) returns `sha`, `name`, `path`, `size`, and more in JSON. An HTTP Request node calling this endpoint directly gives us the `sha` we need.

- **Config:**
  - Method: `GET`
  - URL: `https://api.github.com/repos/ssd-007/n8n-workflows/contents/{{ $json.filePath }}`
  - Authentication: GitHub credential (`aBvdYtLSAq2UoDhc`)
  - `continueOnFail: true` — a 404 (file doesn't exist yet) is expected and handled downstream by the IF node; without this the workflow would error on every first-ever backup of a workflow.

### IF Node
- **Condition:** `={{ !!$json.sha }}`
- **Why this expression, not `isNotEmpty`:**
  During testing, `isNotEmpty` with `typeValidation: strict` silently evaluated to `false` even when `sha` was clearly a non-empty string. The workflow always routed to the Create branch, overwriting files instead of updating them.

  Using `={{ !!$json.sha }}` with a boolean `true` operator bypasses type validation entirely and reliably returns `true` when `sha` is a non-empty string and `false` when it is `undefined` (file doesn't exist). This pattern — `!!value` with boolean operator — is now the preferred approach for field-presence checks in n8n IF nodes.

- **Outputs:** `true` → Edit File, `false` → Create File

### Edit File — GitHub node
- **Why:** The n8n GitHub node's `Edit File` operation wraps the GitHub Contents API `PUT` endpoint cleanly — handles base64 encoding, auth headers, and the required request body format automatically. No raw HTTP Request needed here.
- **Config:**
  - Owner: `ssd-007`
  - Repo: `n8n-workflows`
  - `filePath`: plain string expression `={{ $json.filePath }}` (not resource locator format — see note below)
  - `fileContent`: `={{ $json.fileContent }}`
  - `sha`: `={{ $json.sha }}` — required by the GitHub API to confirm you're updating the correct version
  - Commit message: `daily backup {{ $now.toFormat('yyyy-MM-dd') }}`

- **Note on `filePath` format:** The n8n validator suggested using a resource locator object (`__rl: true, mode: "expression"`) for `filePath`. This causes a runtime error: `url.endsWith is not a function`. The correct format is a plain string expression.

### Create File — GitHub node
- **Why:** Same as Edit File — the GitHub node handles the GitHub Contents API `PUT` cleanly. Creating and editing are both `PUT` requests to the same endpoint; the only difference is that creating omits the `sha` field.
- **Config:** Identical to Edit File except no `sha` property.

### Set Status (Updated) / Set Status (Created)
- **Why:** After the Edit/Create branch, each item needs a status label so the Merge node downstream and the Summarise node can tell which items were updated vs. created. A Set node on each branch adds `status: "updated"` or `status: "created"`.

### Merge Node
- **Why:** The IF node splits the stream into two branches. To send a single Telegram message covering all workflows (not one message per workflow), both branches must be rejoined before the Summarise node. The Merge node waits for all items from both branches.

### Summarise — JavaScript Code Node
- **Why this over native nodes:**
  The goal is to produce one summary string like:
  ```
  n8n Backup Complete — 2026-03-26
  Updated: Workflow A, Workflow B
  Created: Workflow C
  Total: 3 workflows
  ```
  Doing this with native n8n nodes would require an Aggregate node, a Split Out node, multiple Set nodes, and an Item Lists node — roughly 5-6 nodes chained together, each adding cognitive overhead. A single Code node with 10 lines of JavaScript is clearer, easier to maintain, and does the same job in one step.

- **Mode:** `Run Once for All Items` — receives the full merged array and produces one output item.

### Telegram Node
- **Why Telegram, not email or Slack:**
  Fast, free, no configuration overhead. A dedicated "n8n Bot" Telegram bot sends to a private chat (ID: `8382569331`). One message per run, not one per workflow.
- **Credential:** "Telegram n8n Bot"

---

## Credentials

| Service  | Credential Name       | Credential ID        | Used By                    |
|----------|-----------------------|----------------------|----------------------------|
| n8n API  | n8n account           | `OxDWNqPilqexXBWL`   | n8n Workflows node         |
| GitHub   | GitHub account        | `aBvdYtLSAq2UoDhc`   | HTTP Request, Edit File, Create File |
| Telegram | Telegram n8n Bot      | (name-referenced)    | Telegram node              |

GitHub owner: `ssd-007`

---

## Key Lessons From Building This

1. **The GitHub node does not expose `sha`.** Use an HTTP Request to the GitHub Contents API (`GET /repos/.../contents/path`) whenever you need file metadata. The GitHub node is fine for creating/editing but not for reading metadata.

2. **`isNotEmpty` with `typeValidation: strict` silently fails** in IF nodes. Use `={{ !!$json.fieldName }}` with a boolean operator instead for any field-existence check.

3. **`filePath` in GitHub nodes must be a plain string expression** — not the resource locator object format that the validator may suggest. The `__rl: true` format causes a runtime error.

4. **`continueOnFail: true` on the SHA request is not optional.** Without it, the first backup of any new workflow fails with a 404 error.

5. **One Code node beats five native nodes** when the transformation is a simple string aggregation. Reach for Code nodes when native nodes would require a long chain just to do something a few lines of JavaScript handles trivially.

---

## Why This Was Built

Portfolio piece demonstrating real-world n8n automation: scheduled triggers, API integration, conditional branching, error handling, and notification delivery. The repo is public so it also serves as a live example of automated workflow version control.
