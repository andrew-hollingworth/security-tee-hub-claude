You are helping a Security TEE at Datadog process a Security Data Deletion JIRA ticket. Work through the following steps carefully and in order, waiting for user input where required.

The JIRA ticket URL is: $ARGUMENTS

---

## Step 1 — Fetch ticket details

Extract the ticket key from the URL (e.g. `SCRS-2037`) and use the Atlassian MCP to fetch the full ticket.

If the Atlassian MCP is not authenticated, stop and tell the user to run `/mcp` and authenticate "claude.ai Atlassian" before continuing.

Extract and display the following:

| Field | JIRA Field | Value |
|---|---|---|
| Ticket key | — | e.g. SCRS-2037 |
| Org ID | `customfield_10236` | — |
| Org Name | `customfield_10237` | — |
| Region | `customfield_16565` | — |
| Deletion due date | `customfield_10328` | Parse date with regex from text like `"Contractual due date for deletion: , YYYY-MM-DD"` |
| logs-admin URL | ticket description | Extract the hyperlink on the word "here" from a line like "Please follow the provided instructions **here**" |

---

## Step 2 — Check K9 Activations

Display the logs-admin URL extracted in Step 1 and ask the user to:
1. Open it in their browser
2. Navigate to the **K9 Activations** tab
3. Copy and paste the full activations table here

The table will be tab-separated and look like this:
```
Product	Activation	Last updated
securityMonitoring	false	
applicationSecurity	true	3 year(s) ago
...
```

Once the user pastes the table, parse it and identify all rows where `Activation` is `true`.

Display a clear summary back to the user, for example:

**Activated products (require action):**
- `applicationSecurity` → [Manual] SQL deletion (you run kubectl + SQL; Claude provides the commands)
- `complianceMonitoring` → [Claude] Posts 3 Slack messages with cross-links
- `infraVulnerabilityManagement`, `containerVulnerabilityManagement`, `hostVulnerabilityManagement` → [Claude] Creates 1 K9VULN ticket (VM board) if any are active
- `codeSecurityScaStatic`, `codeSecurityScaRuntime` → [Claude] Creates 1 K9VULN ticket (SCA board) if any are active
- `codeSecurityIac` → [Manual] Action not yet documented in runbook
- `runtimeSecurity` → [Auto] No action needed
- `ciem` → [Auto] No action needed
- `securityMonitoring` → [Auto] No action needed (data auto-deletes after 90 days)
- `codeSecuritySast` → [Auto] No action needed (data auto-deletes 90 days after ticket closes)
- `codeSecurityIast` → [Auto] No action needed (data auto-deletes 90 days after ticket closes)
- `codeSecuritySecret` → [Auto] No action needed (data auto-deletes 90 days after ticket closes)

**Inactive products (no action needed):**
- All others — false

Ask the user: "Does this look correct? Confirm to proceed."

Wait for explicit confirmation before continuing to Step 3.

---

## Step 3 — Take action for each activated product

Work through each activated product one at a time.

---

### `ciem` — No action needed
Note this and move on.

---

### `securityMonitoring` — No action needed
Data auto-deletes after 90 days. Note this and move on.

---

### `runtimeSecurity` — No action needed
Note this and move on.

---

### `codeSecuritySast`, `codeSecurityIast`, `codeSecuritySecret` — No action needed
Data auto-deletes 90 days after ticket closes (closing happens within 24h of no activity). Note this and move on.

---

### `codeSecurityIac` — Action unknown
This product is not covered by the current runbook. Warn the user:
> ⚠️ `codeSecurityIac` is activated but the required action for this product is not yet documented in this skill. Please handle this manually and update the runbook.

---

### `applicationSecurity` — Manual SQL deletion

1. Display the orgstore-prober URL and instruct the user:
   ```
   https://orgstore-prober.<REGION>.prod.dog/
   ```
   > "Please open this URL, find the row for ASM, and share the value from the **k8s Cluster** column."

2. Wait for the user to provide the cluster name, then display the ready-to-copy kubectl login command with all values filled in:

   ```bash
   LOGICAL_NAME=asm
   K8S_CLUSTER=<VALUE_FROM_USER>
   kubectl exec \
     --context $K8S_CLUSTER \
     --namespace orgstore-$LOGICAL_NAME \
     -it \
     deployment/orgstore-$LOGICAL_NAME-toolbox -- pg-wrap -o $LOGICAL_NAME psql
   ```

3. Display the ready-to-copy SQL deletion script with the org ID pre-filled:

   ```sql
   DO $$
   DECLARE
       target_org_id BIGINT := <ORG_ID>;
   BEGIN
       DELETE FROM actor_notes                       WHERE org_id = target_org_id;
       DELETE FROM asm_cluster                       WHERE org_id = target_org_id;
       DELETE FROM asm_comments                      WHERE org_id = target_org_id;
       DELETE FROM asm_threat_intel_tables           WHERE org_id = target_org_id;
       DELETE FROM flagged_ips                       WHERE org_id = target_org_id;
       DELETE FROM integration_activation_overrides  WHERE org_id = target_org_id;
       DELETE FROM route_inference                   WHERE org_id = target_org_id;
       DELETE FROM signal_actions                    WHERE org_id = target_org_id;
       DELETE FROM vm_activation                     WHERE org_id = target_org_id;
       DELETE FROM ai_guard_blocking                 WHERE org_id = target_org_id;
   END $$;
   ```

4. Instruct:
   > "Run the kubectl command to connect to the cluster, then run the SQL script. Take a screenshot of the completed output for your records. Confirm here when done."

   Wait for confirmation before continuing.

---

### `complianceMonitoring` — 3 Slack messages with cross-links

If the Slack MCP is not authenticated, stop and tell the user to run `/mcp` and authenticate "claude.ai Slack" first.

Post the following message to all three channels and capture each message's permalink:

> Org `<ORG_ID>` has requested that all their data be deleted. Please confirm all related data for Compliance Monitoring has been deleted.

**Channels:**
1. `#k9-ask-findings-platform`
2. `#k9-ask-security-graph-and-prioritization`
3. `#k9-ask-cspm`

Once all three are sent, post a thread reply to each message linking to the other two:
> FYI, related deletion requests have also been posted in `#<channel-2>` (<permalink>) and `#<channel-3>` (<permalink>)

If the Slack MCP does not return message permalinks, post the thread replies mentioning the channel names only (without deep links).

---

### `infraVulnerabilityManagement`, `containerVulnerabilityManagement`, `hostVulnerabilityManagement` — Create K9VULN ticket (VM board)

If **any** of these three products are activated, create a **single** ticket in the **K9VULN** project using the Atlassian MCP.

Fields:
- **Title:** The summary of the SCRS ticket (reuse verbatim)
- **Work Type:** Task
- **Components:** Support
- **Due Date:** deletion due date from Step 1, formatted as `MM/DD/YYYY`
- **Linked work item:** this ticket "Blocks" `<SCRS_TICKET_URL>`
- **Description:**
  > Org `<ORG_ID>` has requested that all their data be deleted. Please confirm all related data for Vulnerability Management has been deleted.

Target board: `https://datadoghq.atlassian.net/jira/software/c/projects/K9VULN/boards/6854/backlog`

After creating the ticket, confirm the link back to the SCRS ticket was established.

---

### `codeSecurityScaStatic`, `codeSecurityScaRuntime` — Create K9VULN ticket (SCA board)

If **any** of these two products are activated, create a **single** ticket in the **K9VULN** project using the Atlassian MCP.

Fields:
- **Title:** The summary of the SCRS ticket (reuse verbatim)
- **Work Type:** Task
- **Components:** Support
- **Due Date:** deletion due date from Step 1, formatted as `MM/DD/YYYY`
- **Linked work item:** this ticket "Blocks" `<SCRS_TICKET_URL>`
- **Description:**
  > Org `<ORG_ID>` has requested that all their data be deleted. Please confirm all related data for Code Security SCA has been deleted.

Target board: `https://datadoghq.atlassian.net/jira/software/c/projects/K9VULN/boards/8099`

After creating the ticket, confirm the link back to the SCRS ticket was established.

---

## Step 4 — Wait for downstream tickets (if applicable)

If any K9VULN tickets were created, inform the user:
> "The following linked tickets have been created and must be resolved before closing this ticket: [list with links]. Return to this workflow once all are marked Done."

Wait for the user to confirm all downstream tickets are complete before proceeding to Step 5.

If no K9VULN tickets were created, proceed directly to Step 5.

---

## Step 5 — Document and close

1. Use the Atlassian MCP to post a comment on the SCRS ticket summarising all actions taken. The comment should include:
   - Which products were activated
   - Actions taken for each (Slack channels posted with links, K9VULN tickets created with links, SQL deletion confirmed, or no action required)

2. Transition the SCRS ticket to **Done** using the Atlassian MCP.

3. Confirm to the user that the ticket has been closed and the workflow is complete.
