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
| Region | ticket description | Use the **Datacenter** field (e.g., `us1.prod`) — do NOT use `customfield_16565`, it is unreliable |
| Deletion due date | `customfield_10328` | Parse date with regex from text like `"Contractual due date for deletion: , YYYY-MM-DD"` |
| logs-admin URL | ticket description | Extract the hyperlink on the word "here" from a line like "Please follow the provided instructions **here**" |

After displaying the table, look for a **Child Orgs** section in the ticket description. Child orgs appear in the format:
`* [<ORG_ID> | <ORG_NAME>](<supportdog_url>)`

Extract all child org IDs and names and display them clearly, for example:
> **Child orgs found:** 401640 (Hero - to be deleted)

If no Child Orgs section exists or it is empty, note "No child orgs" and continue.

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

Once the user pastes the table, if child orgs were found in Step 1, for each child org construct and display its K9 activations URL:
```
https://logs-admin.<datacenter>.prod.dog/web/#/orgs/<CHILD_ORG_ID>/k9activations
```
Ask the user to open each URL, navigate to K9 Activations, and paste the table. Repeat until all child orgs' activations are collected.

Parse all activations. Present the summary **per org**:
- If all orgs have identical activations, say so and show one combined summary
- If any org differs, call out the differences explicitly so the user knows which actions apply to which org

Display a clear summary back to the user, for example:

**Activated products (require action):**
- `applicationSecurity` → [Manual] SQL deletion (you run kubectl + SQL; Claude provides the commands)
- `complianceMonitoring` → [Claude] Posts 3 Slack messages with cross-links
- `infraVulnerabilityManagement`, `containerVulnerabilityManagement`, `hostVulnerabilityManagement` → [Claude] Creates 1 K9VULN ticket (VM board) if any are active
- `codeSecurityScaStatic`, `codeSecurityScaRuntime` → [Claude] Creates 1 K9VULN ticket (SCA board) if any are active
- `codeSecurityIac` → [Claude] Creates 1 K9VULN ticket (IaC board) + posts link to `#k9-iac-secrets-backroom`
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

### `codeSecurityIac` — Create K9VULN ticket (IaC board) + Slack post

Create a ticket in the **K9VULN** project using the Atlassian MCP.

Fields:
- **Title:** The summary of the SCRS ticket (reuse verbatim)
- **Work Type:** Task
- **Components:** Support
- **Due Date:** deletion due date from Step 1, formatted as `MM/DD/YYYY`
- **Linked work item:** this ticket "Blocks" `<SCRS_TICKET_URL>`
- **Description:**
  > Orgs `<ORG_ID>` [and `<CHILD_ORG_ID>` ...] have requested that all their data be deleted. Please confirm all related data for Code Security IaC has been deleted.

Target board: `https://datadoghq.atlassian.net/jira/software/c/projects/K9VULN/boards/8574`

After creating the ticket, confirm the link back to the SCRS ticket was established.

Then post a single top-level message to `#k9-iac-secrets-backroom` with the K9VULN ticket link and the SCRS ticket link:

> Orgs `<ORG_ID>` [and `<CHILD_ORG_ID>` ...] have requested that all their data be deleted. Please confirm all related data for Code Security IaC has been deleted. Tracking ticket: `<K9VULN_TICKET_URL>`. Original deletion request: `<SCRS_TICKET_URL>`.

Retain the K9VULN ticket link — it will be included in the checklist comment posted in Step 4.

---

### `applicationSecurity` — Manual SQL deletion

1. Display the orgstore-prober URL and instruct the user. Use the datacenter from the ticket description (e.g., `us1.prod` → `us1`):
   ```
   https://orgstore-prober.<DATACENTER>.prod.dog/
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

   Wait for confirmation, then repeat steps 3–4 for each child org (same kubectl connection, different org ID in the script). Provide each child org's SQL script one at a time, waiting for confirmation between each.

---

### `complianceMonitoring` — 3 Slack messages with cross-links

If the Slack MCP is not authenticated, stop and tell the user to run `/mcp` and authenticate "claude.ai Slack" first.

Post the following message to all three channels and capture each message's permalink. Include all org IDs (parent + any child orgs) in the message:

> Orgs `<ORG_ID>` [and `<CHILD_ORG_ID>` ...] have requested that all their data be deleted. Please confirm all related data for Compliance Monitoring has been deleted.

**Channels:**
1. `#k9-ask-findings-platform`
2. `#k9-ask-security-graph-and-prioritization`
3. `#k9-ask-cspm`

If the Slack MCP did not return permalinks for all three messages, stop and warn the user:
> ⚠️ Could not retrieve permalink for one or more Slack messages. Retrieve them manually from Slack and paste them here before continuing.

Once all three permalinks are confirmed, post a thread reply to each message linking to the other two and to the original SCRS ticket:
> FYI, related deletion requests have also been posted in `#<channel-2>` (<permalink>) and `#<channel-3>` (<permalink>)
>
> Original deletion request is here: <SCRS_TICKET_URL>

Retain the three permalinks — they will be included in the checklist comment posted in Step 4.

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
  > Orgs `<ORG_ID>` [and `<CHILD_ORG_ID>` ...] have requested that all their data be deleted. Please confirm all related data for Vulnerability Management has been deleted.

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
  > Orgs `<ORG_ID>` [and `<CHILD_ORG_ID>` ...] have requested that all their data be deleted. Please confirm all related data for Code Security SCA has been deleted.

Target board: `https://datadoghq.atlassian.net/jira/software/c/projects/K9VULN/boards/8099`

After creating the ticket, confirm the link back to the SCRS ticket was established.

---

## Step 4 — Post checklist comment and update ticket status

### Post checklist comment

Post a single comprehensive comment to the SCRS ticket.

Use plain markdown checkbox syntax without bullet-point prefixes (no leading `-`). Jira renders each `[x]` or `[ ]` at the start of a line as a proper checkbox.

**If there are no child orgs**, use a flat format:

```
Deletion progress tracker for org `<ORG_ID>`:

[x] `securityMonitoring` — auto-deletes after 90 days, no action needed
[x] `applicationSecurity` — SQL deletion completed
[ ] `complianceMonitoring` — #k9-ask-findings-platform: <permalink>
[ ] `complianceMonitoring` — #k9-ask-security-graph-and-prioritization: <permalink>
[ ] `complianceMonitoring` — #k9-ask-cspm: <permalink>
[ ] `codeSecurityScaRuntime` — [K9VULN-XXXXX](<link>) resolved
[ ] `codeSecurityIac` — [K9VULN-XXXXX](<link>) resolved
```

**If there are child orgs**, use a two-section format:

```
Deletion progress tracker:

**Org <ORG_ID> (<ORG_NAME> — parent):**
[x] `securityMonitoring` — auto-deletes after 90 days, no action needed
[x] `applicationSecurity` — SQL deletion completed

**Org <CHILD_ORG_ID> (<CHILD_ORG_NAME> — child):**
[x] `securityMonitoring` — auto-deletes after 90 days, no action needed
[x] `applicationSecurity` — SQL deletion completed

**Shared actions (all orgs):**
[ ] `complianceMonitoring` — #k9-ask-findings-platform: <permalink>
[ ] `complianceMonitoring` — #k9-ask-security-graph-and-prioritization: <permalink>
[ ] `complianceMonitoring` — #k9-ask-cspm: <permalink>
[ ] `codeSecurityScaRuntime` — [K9VULN-XXXXX](<link>) resolved
[ ] `codeSecurityIac` — [K9VULN-XXXXX](<link>) resolved
```

Rules:
- Auto-delete products (`securityMonitoring`, `runtimeSecurity`, `ciem`, `codeSecuritySast`, `codeSecurityIast`, `codeSecuritySecret`) get `[x]` with a brief note — include in each org's per-org section
- `applicationSecurity` gets `[x]` if SQL deletion confirmed done, `[ ]` if still pending — per-org section
- `complianceMonitoring` Slack threads and K9VULN tickets go in the **Shared actions** section (they cover all orgs in one action)
- Only include rows for activated products
- Add one per-org section for each child org found

### Transition ticket status

Evaluate in priority order:

**Priority 1 — Engineering action needed → Engineering Triage**
If any K9VULN ticket(s) were created, OR `complianceMonitoring` Slack messages were sent (regardless of whether `applicationSecurity` SQL deletion was also needed — once confirmed done it no longer blocks transition):
Transition the SCRS ticket to **Engineering Triage** using the Atlassian MCP (`transitionJiraIssue`). Proceed to Step 5.

**Priority 2 — No action needed → Done**
Else (only auto-delete products were active, no tickets created and no Slack messages sent):
Transition the SCRS ticket to **Done** using the Atlassian MCP (`transitionJiraIssue`). Skip Step 5 and proceed directly to Step 6.

**In all cases**, also update the SCRS ticket using `editJiraIssue` to set the Escalation Reason field:
`customfield_19256: { "id": "28799" }` — `[ACCESS] - Access or Permissions Required`

---

## Step 5 — Wait for downstream tickets (if applicable)

If any K9VULN tickets were created, inform the user:
> "The following linked tickets have been created and must be resolved before closing this ticket: [list with links]. Return to this workflow once all are marked Done."

Wait for the user to confirm all downstream tickets are complete before proceeding to Step 6.

If no K9VULN tickets were created, proceed directly to Step 6.

---

## Step 6 — Document and close

1. Use the Atlassian MCP to post a comment on the SCRS ticket summarising all actions taken. The comment should include:
   - Which products were activated
   - Actions taken for each (Slack channels posted with links, K9VULN tickets created with links, SQL deletion confirmed, or no action required)

2. Transition the SCRS ticket to **Done** using the Atlassian MCP.

3. Confirm to the user that the ticket has been closed and the workflow is complete.
