---
name: pup-ddsql
description: Run DDSQL queries against Datadog's SQL engine using pup CLI, direct curl to the DDSQL API, or the Analysis Workspace API. Use when querying cloud resources (AWS, GCP, Azure), security findings, or infrastructure inventory via DDSQL, or when the user mentions pup ddsql, cloud resource queries, resource inventory SQL, or dd.security_findings().
---

# DDSQL Queries

## Overview

DDSQL is Datadog's SQL engine for querying cloud resources, security findings, and infrastructure inventory. Three methods are available:

| Method                     | Endpoint                                             | Supports `dd.security_findings()` | Auth               |
| -------------------------- | ---------------------------------------------------- | --------------------------------- | ------------------ |
| `pup ddsql query`          | `/api/v2/ddsql/user/table`                           | No                                | OAuth2 or API keys |
| Direct curl (DDSQL API)    | `/api/v2/ddsql/user/table`                           | No                                | API keys           |
| **Analysis Workspace API** | `/api/unstable/logs/analysis-workspace/query/scalar` | **Yes**                           | API keys           |

**Use the Analysis Workspace API (Method 3) when you need:**

- `dd.security_findings()` table functions
- `dd.containers` table
- Dot-notation tables (e.g., `aws.ecs_task`) -- these fail on pup/DDSQL resource API
- JOINs between security findings and container/infrastructure tables

## Method 1: pup CLI

Install via Homebrew (or whatever the team's distribution channel is); once installed, `pup` should be on `$PATH` (e.g. `/opt/homebrew/bin/pup`).

```bash
# JSON output (default)
pup ddsql query "SELECT * FROM aws_bedrock_agent LIMIT 5"

# Table output
pup ddsql query "SELECT instance_id, instance_type FROM aws_ec2_instance LIMIT 10" --format table

# CSV output
pup ddsql query "SELECT * FROM aws_ec2_instance" --format csv

# Read query from file
pup ddsql query -f query.sql --format table
```

Auth: requires `pup auth login` (OAuth2) or `DD_API_KEY`/`DD_APP_KEY` env vars.

## Credential Management: dd-auth

`dd-auth` is an internal OAuth-based CLI that provisions `DD_API_KEY`, `DD_APP_KEY`, and `DD_SITE` on the fly via browser-based OAuth 2.0 PKCE flow. Use it when stored keys are expired, revoked, or return 403.

**Install (macOS):** `brew install datadog/tap/dd-auth`
**Upgrade:** `brew update && brew upgrade datadog/tap/dd-auth`
**Install (Linux):** `update-tool dd-auth`

### Usage

```bash
# Run a command with auto-provisioned credentials (simplest)
dd-auth -- python myscript.py

# For commands that need shell expansion of env vars, use bash -c:
dd-auth -- bash -c 'curl -H "DD-API-KEY: ${DD_API_KEY}" -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" https://api.${DD_SITE}/api/v1/validate'

# Or use heredoc for complex commands:
dd-auth << 'EOF'
curl -X POST "https://app.datadoghq.com/api/unstable/logs/analysis-workspace/query/scalar" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d '{"data":{"type":"analysis_workspace_query_request",...}}'
EOF

# Export credentials to current shell for subsequent commands
eval $(dd-auth --output)

# Target a specific org/domain
dd-auth --domain app.datadoghq.com -- <command>   # US1 prod (default)
dd-auth --domain app.datadoghq.eu -- <command>    # EU1
dd-auth --domain us3.datadoghq.com -- <command>   # US3
dd-auth --domain ap1.datadoghq.com -- <command>   # AP1

# Bypass credential cache (force fresh tokens)
dd-auth --no-cache -- <command>

# Set keys in a remote workspace
dd-auth --workspace <my-workspace>
```

### Key options

| Flag | Description |
|---|---|
| `--output, -o` | Print `DD_API_KEY=... DD_APP_KEY=... DD_SITE=...` to stdout |
| `--domain, -s` | Datadog org domain (default: `app.datadoghq.com`) |
| `--no-cache` | Skip cache, generate fresh keys |
| `--workspace, -w` | Set keys on a remote machine via SSH |
| `--verbose, -v` | Debug logging (WARNING: prints secrets) |
| `--api-key-env` | Custom env var name for API key (default: `DD_API_KEY`) |
| `--app-key-env` | Custom env var name for App key (default: `DD_APP_KEY`) |
| `--site-env` | Custom env var name for site (default: `DD_SITE`) |

### Fallback strategy for 403 errors

1. Try existing `DD_API_KEY`/`DD_APP_KEY` env vars
2. If 403, refresh via: `eval $(dd-auth --output)`
3. If dd-auth not installed: `brew install datadog/tap/dd-auth`
4. Last resort: re-export from macOS Keychain: `export DD_APP_KEY=$(security find-generic-password -a "$USER" -s "DD_APP_KEY" -w)`

### Notes

- Keys are cached in the macOS Keychain (click "Always Allow" on first use and after upgrades)
- Keys are ephemeral -- auto-generated and can be refreshed with `--no-cache`
- `dd-auth --output` also exports `DD_SITE` (e.g., `datadoghq.com`)
- Slack channel for support: `#core-observability`

## Method 2: Direct curl to DDSQL API

```bash
# Requires DD_API_KEY and DD_APP_KEY env vars
NOW_MS=$(($(date +%s) * 1000))
FROM_MS=$((NOW_MS - 86400000))

curl -s -X POST "https://app.datadoghq.com/api/v2/ddsql/user/table" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d "{
    \"data\": {
      \"type\": \"ddsql_table_request\",
      \"attributes\": {
        \"query\": \"SELECT agent_name, foundation_model FROM aws_bedrock_agent LIMIT 5\",
        \"default_start\": ${FROM_MS},
        \"default_end\": ${NOW_MS},
        \"default_interval\": 86400000
      }
    }
  }"
```

### Request format

```json
{
  "data": {
    "type": "ddsql_table_request",
    "attributes": {
      "query": "SELECT ...",
      "default_start": 1770800000000,
      "default_end": 1770900000000,
      "default_interval": 86400000
    }
  }
}
```

- `type`: must be `"ddsql_table_request"`
- `query`: SQL string
- `default_start` / `default_end`: epoch milliseconds (required, even for non-time-series tables)
- `default_interval`: milliseconds (required, must be > 0)

### Response format

```json
{
  "data": [
    {
      "id": "ddsql_response",
      "type": "scalar_response",
      "attributes": {
        "columns": [
          {
            "name": "agent_name",
            "type": "string",
            "values": ["city-assistant_dev", "holidays-agent"]
          },
          {
            "name": "foundation_model",
            "type": "string",
            "values": ["anthropic.claude-3-sonnet...", "..."]
          }
        ]
      }
    }
  ]
}
```

Columns are returned as arrays of values (columnar format), not row objects.

### Helper: curl to row-based JSON

```bash
ddsql() {
  local query="$1"
  local now_ms=$(($(date +%s) * 1000))
  local from_ms=$((now_ms - 86400000))

  curl -s -X POST "https://app.datadoghq.com/api/v2/ddsql/user/table" \
    -H "Content-Type: application/json" \
    -H "DD-API-KEY: ${DD_API_KEY}" \
    -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
    -d "{
      \"data\": {
        \"type\": \"ddsql_table_request\",
        \"attributes\": {
          \"query\": $(python3 -c "import json; print(json.dumps('$query'))"),
          \"default_start\": ${from_ms},
          \"default_end\": ${now_ms},
          \"default_interval\": 86400000
        }
      }
    }" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if 'errors' in resp:
    for e in resp['errors']:
        print('ERROR:', e.get('detail', e) if isinstance(e, dict) else e, file=sys.stderr)
    sys.exit(1)
cols = resp['data'][0]['attributes']['columns']
names = [c['name'] for c in cols]
rows = len(cols[0]['values']) if cols else 0
for i in range(rows):
    print(json.dumps({n: cols[j]['values'][i] for j, n in enumerate(names)}))
"
}

# Usage:
ddsql "SELECT agent_name, agent_status FROM aws_bedrock_agent LIMIT 5"
```

## Table Naming Convention

DDSQL tables follow the pattern: `{cloud_provider}_{service}_{resource_type}`

Examples:

- `aws_ec2_instance`, `aws_s3_bucket`, `aws_bedrock_agent`, `aws_lambda_function`
- `gcp_compute_instance`
- `azure_vm`

Some tables use dot notation (require Method 3 -- Analysis Workspace API):

- `aws.ec2_instance` (EC2 instances with `instance_id`, `instance_type`, `account_id`, `tags`)
- `aws.ecr_image` (ECR images with `repository_name`, `image_digest`, `account_id`)
- `aws.ecs_task` (ECS tasks with `containers` JSON, `desired_status`)
- `aws.lambda_function` (Lambda functions with `function_name`, `function_arn`, `package_type`, `runtime`, `tags` hstore)

**Important:** Dot-notation tables (e.g., `aws.ec2_instance`) are **different** from underscore tables (e.g., `aws_ec2_instance`). Use dot notation on Method 3.

Security/infra tables use the `dd.` prefix (require Method 3):

- `dd.hosts` (infrastructure hosts)
- `dd.containers` (running containers with image, tags hstore, orchestrator)
- `dd.security_findings(...)` (security findings -- **table function**)

## Discovering Tables

DDSQL does **not** support `SHOW TABLES` or `information_schema`. To find tables:

1. **From metrics:** Search for `rcapi.ng.queries.*` metrics in Datadog. Each metric maps to a table name (e.g., `rcapi.ng.queries.aws_bedrock_agent` -> table `aws_bedrock_agent`).
2. **From logs:** Search `service:retriever` logs for `BeagleSQL router` messages containing `FROM <table_name>`.
3. **Trial and error:** `SELECT * FROM {guessed_table_name} LIMIT 1`. HTTP 400 = doesn't exist; results = exists.

## AWS Bedrock Tables (Verified 2026-02-12)

| Table                                       | Populated | Description              |
| ------------------------------------------- | --------- | ------------------------ |
| `aws_bedrock_agent`                         | Yes (13)  | Bedrock agents           |
| `aws_bedrock_agent_action_group`            | Yes (3)   | Agent action groups      |
| `aws_bedrock_agent_alias`                   | Yes (50)  | Agent aliases            |
| `aws_bedrock_application_inference_profile` | Yes (1)   | Inference profiles       |
| `aws_bedrock_async_invoke`                  | No        | Async invocations        |
| `aws_bedrock_blueprint`                     | No        | Blueprints               |
| `aws_bedrock_custom_model`                  | Yes (2)   | Custom/fine-tuned models |
| `aws_bedrock_data_source`                   | Yes (3)   | KB data sources          |
| `aws_bedrock_evaluation_job`                | No        | Evaluation jobs          |
| `aws_bedrock_flow`                          | Yes (1)   | Flows                    |
| `aws_bedrock_flow_alias`                    | Yes (6)   | Flow aliases             |
| `aws_bedrock_guardrail`                     | Yes (6)   | Guardrails               |
| `aws_bedrock_imported_model`                | No        | Imported models          |
| `aws_bedrock_ingestion_job`                 | Yes (4)   | KB ingestion jobs        |
| `aws_bedrock_knowledge_base`                | Yes (3)   | Knowledge bases          |
| `aws_bedrock_marketplace_model_endpoint`    | No        | Marketplace endpoints    |
| `aws_bedrock_model_copy_job`                | No        | Model copy jobs          |
| `aws_bedrock_model_customization_job`       | Yes (4)   | Customization jobs       |
| `aws_bedrock_model_invocation_job`          | No        | Invocation jobs          |
| `aws_bedrock_prompt`                        | Yes (1)   | Managed prompts          |
| `aws_bedrock_prompt_router`                 | No        | Prompt routers           |
| `aws_bedrock_provisioned_model_throughput`  | No        | Provisioned throughput   |
| `aws_bedrock_settings`                      | Yes (2)   | Account settings         |

## Batch Checking Table Population

```bash
tables=("aws_bedrock_agent" "aws_bedrock_guardrail" "aws_bedrock_flow")
for t in "${tables[@]}"; do
  result=$(pup ddsql query "SELECT COUNT(*) AS cnt FROM $t" -o json 2>&1)
  if [ $? -eq 0 ]; then
    cnt=$(echo "$result" | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['cnt'])")
    echo "$t: $cnt rows"
  else
    echo "$t: ERROR"
  fi
done
```

## Common Patterns

```bash
# Count resources by account
pup ddsql query "SELECT account_id, COUNT(*) AS cnt FROM aws_bedrock_agent GROUP BY account_id ORDER BY cnt DESC" --format table

# Filter by tags
pup ddsql query "SELECT display_name, tags FROM aws_bedrock_guardrail WHERE tags LIKE '%prod%'" --format table

# Get column names for a table
pup ddsql query "SELECT * FROM aws_bedrock_agent LIMIT 1" -o json | python3 -c "import sys,json; print('\n'.join(json.load(sys.stdin)[0].keys()))"
```

## Method 3: Analysis Workspace API (supports `dd.security_findings()`)

**This is the only API method that supports `dd.security_findings()` table functions.**

**Endpoint:** `POST https://app.datadoghq.com/api/unstable/logs/analysis-workspace/query/scalar`

### Request format

```json
{
  "data": {
    "type": "analysis_workspace_query_request",
    "attributes": {
      "datasets": [
        {
          "data_source": "analysis_dataset",
          "name": "user_query",
          "query": {
            "sql_query": "SELECT ... FROM dd.security_findings(...) AS (...) ...",
            "type": "sql_analysis"
          }
        }
      ],
      "query": {
        "dataset": "user_query",
        "time_window": {
          "from": 1770973062468,
          "to": 1770976662468
        },
        "limit": 5000
      }
    }
  },
  "meta": {
    "client_id": "use_advanced_query_api_query",
    "user_query_id": "any-uuid-here",
    "use_async_querying": false
  }
}
```

Key fields:

- `data.type`: must be `"analysis_workspace_query_request"`
- `datasets[].data_source`: `"analysis_dataset"`
- `datasets[].name`: `"user_query"` (referenced by `query.dataset`)
- `datasets[].query.type`: `"sql_analysis"`
- `datasets[].query.sql_query`: the full SQL query string
- `query.dataset`: matches `datasets[].name`
- `query.time_window.from` / `to`: epoch milliseconds
- `query.limit`: max rows (default 5000)
- `meta.use_async_querying`: `false` for synchronous, `true` for async (returns `status: "running"`)

### Response format

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "query_result",
      "attributes": {
        "columns": [
          { "name": "col1", "type": "string", "values": ["val1", "val2"] },
          { "name": "col2", "type": "int64", "values": [1, 2] }
        ]
      }
    }
  ]
}
```

Columnar format: each column has a `values` array. Row N = `columns[*].values[N]`.

### Example: security_findings query via curl

```bash
NOW_MS=$(($(date +%s) * 1000))
FROM_MS=$((NOW_MS - 3600000))

SQL_QUERY=$(python3 -c "
import json
q = '''SELECT resource_name, title, severity
FROM dd.security_findings(
    columns => ARRAY ['@resource_name', '@title', '@severity', '@status', '@resource_type'],
    finding_types => ARRAY ['host_and_container_vulnerability']
) AS (
    resource_name VARCHAR,
    title VARCHAR,
    severity VARCHAR,
    status VARCHAR,
    resource_type VARCHAR
)
WHERE status = 'open' AND resource_type = 'image'
LIMIT 20'''
print(json.dumps(q))
")

curl -s -X POST "https://app.datadoghq.com/api/unstable/logs/analysis-workspace/query/scalar" \
  -H "Content-Type: application/json" \
  -H "DD-API-KEY: ${DD_API_KEY}" \
  -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
  -d "{
    \"data\": {
      \"type\": \"analysis_workspace_query_request\",
      \"attributes\": {
        \"datasets\": [{
          \"data_source\": \"analysis_dataset\",
          \"name\": \"user_query\",
          \"query\": {
            \"sql_query\": ${SQL_QUERY},
            \"type\": \"sql_analysis\"
          }
        }],
        \"query\": {
          \"dataset\": \"user_query\",
          \"time_window\": {
            \"from\": ${FROM_MS},
            \"to\": ${NOW_MS}
          },
          \"limit\": 5000
        }
      }
    },
    \"meta\": {
      \"client_id\": \"use_advanced_query_api_query\",
      \"user_query_id\": \"$(python3 -c 'import uuid; print(uuid.uuid4())')\",
      \"use_async_querying\": false
    }
  }"
```

### Helper: `ddsql_findings()` shell function

```bash
ddsql_findings() {
  local query="$1"
  local limit="${2:-5000}"
  local now_ms=$(($(date +%s) * 1000))
  local from_ms=$((now_ms - 3600000))

  local sql_json=$(python3 -c "import json,sys; print(json.dumps(sys.argv[1]))" "$query")

  curl -s -X POST "https://app.datadoghq.com/api/unstable/logs/analysis-workspace/query/scalar" \
    -H "Content-Type: application/json" \
    -H "DD-API-KEY: ${DD_API_KEY}" \
    -H "DD-APPLICATION-KEY: ${DD_APP_KEY}" \
    -d "{
      \"data\": {
        \"type\": \"analysis_workspace_query_request\",
        \"attributes\": {
          \"datasets\": [{
            \"data_source\": \"analysis_dataset\",
            \"name\": \"user_query\",
            \"query\": {
              \"sql_query\": ${sql_json},
              \"type\": \"sql_analysis\"
            }
          }],
          \"query\": {
            \"dataset\": \"user_query\",
            \"time_window\": {
              \"from\": ${from_ms},
              \"to\": ${now_ms}
            },
            \"limit\": ${limit}
          }
        }
      },
      \"meta\": {
        \"client_id\": \"use_advanced_query_api_query\",
        \"user_query_id\": \"$(python3 -c 'import uuid; print(uuid.uuid4())')\",
        \"use_async_querying\": false
      }
    }" | python3 -c "
import sys, json
resp = json.load(sys.stdin)
if 'errors' in resp:
    for e in resp.get('errors', []):
        print('ERROR:', e.get('detail', e) if isinstance(e, dict) else e, file=sys.stderr)
    sys.exit(1)
if not resp.get('data'):
    print('No data returned', file=sys.stderr); sys.exit(1)
cols = resp['data'][0]['attributes']['columns']
names = [c['name'] for c in cols]
rows = len(cols[0]['values']) if cols else 0
for i in range(rows):
    print(json.dumps({n: cols[j]['values'][i] for j, n in enumerate(names)}))
"
}

# Usage:
ddsql_findings "SELECT resource_name, severity FROM dd.security_findings(
    columns => ARRAY ['@resource_name', '@severity', '@status'],
    finding_types => ARRAY ['host_and_container_vulnerability']
) AS (resource_name VARCHAR, severity VARCHAR, status VARCHAR)
WHERE status = 'open' LIMIT 10"
```

## dd.containers Table (Method 3 only)

| Column         | Type      | Description                                                                                |
| -------------- | --------- | ------------------------------------------------------------------------------------------ |
| `image`        | string    | Full registry image path (e.g., `464622532012.dkr.ecr.us-east-1.amazonaws.com/dogweb-gbi`) |
| `name`         | string    | Container name                                                                             |
| `hostname`     | string    | Host ID                                                                                    |
| `id`           | string    | Container ID                                                                               |
| `orchestrator` | string    | `k8s`, `containerd`, etc.                                                                  |
| `type`         | string    | Container runtime type                                                                     |
| `tags`         | hstore    | Key-value tags (use `->` operator)                                                         |
| `cpu_limit`    | string    | CPU limit                                                                                  |
| `memory_limit` | int64     | Memory limit in bytes                                                                      |
| `thread_limit` | int64     | Thread limit                                                                               |
| `created_at`   | timestamp | Creation time                                                                              |
| `started_at`   | timestamp | Start time                                                                                 |

### Useful tag keys (access via `tags->'key'`)

- `kube_namespace` -- Kubernetes namespace
- `kube_cluster_name` -- Cluster name
- `kube_deployment` -- Deployment name
- `kube_service` -- Service name
- `short_image` -- Short image name (e.g., `dogweb-gbi`)
- `image_name` -- Full image path (same as `image` column)
- `image_tag` -- Image tag
- `env` -- Environment
- `team` -- Owning team
- `service` -- Datadog service name
- `pod_name` -- Pod name
- `kube_container_name` -- Container name within pod

### Example: containers by namespace

```sql
SELECT
    tags->'kube_namespace' AS namespace,
    tags->'kube_cluster_name' AS cluster,
    tags->'short_image' AS short_image,
    image
FROM dd.containers
WHERE orchestrator = 'k8s'
  AND tags->'kube_namespace' IS NOT NULL
LIMIT 20
```

## Joining Vulnerabilities with Container Context (Method 3 only)

**Join key:** `dd.security_findings.resource_name = dd.containers.image` (both use full registry path)

```sql
WITH image_vulns AS (
    SELECT DISTINCT
        resource_name, title, severity, advisory_aliases
    FROM dd.security_findings(
        columns => ARRAY [
            '@resource_name', '@resource_type', '@status',
            '@title', '@severity', '@advisory.aliases'
        ],
        finding_types => ARRAY ['host_and_container_vulnerability']
    ) AS (
        resource_name VARCHAR, resource_type VARCHAR, status VARCHAR,
        title VARCHAR, severity VARCHAR, advisory_aliases VARCHAR
    )
    WHERE resource_type = 'image' AND status = 'open'
),
container_context AS (
    SELECT DISTINCT
        image,
        tags->'kube_namespace' AS namespace,
        tags->'kube_cluster_name' AS cluster_name,
        tags->'short_image' AS short_image,
        tags->'kube_deployment' AS deployment
    FROM dd.containers
    WHERE orchestrator = 'k8s'
      AND tags->'kube_namespace' IS NOT NULL
)
SELECT
    v.severity, v.advisory_aliases AS cve, v.title,
    c.short_image, c.namespace, c.cluster_name, c.deployment,
    v.resource_name AS full_image
FROM image_vulns v
INNER JOIN container_context c ON v.resource_name = c.image
ORDER BY
    CASE v.severity
        WHEN 'critical' THEN 1 WHEN 'high' THEN 2
        WHEN 'medium' THEN 3 WHEN 'low' THEN 4 ELSE 5
    END,
    c.namespace, c.cluster_name
```

## aws.ecs_task Table (Method 3 only, dot notation)

Dot-notation tables like `aws.ecs_task` only work via the Analysis Workspace API. `pup ddsql` returns "missing table" errors for these.

| Column           | Type          | Description                     |
| ---------------- | ------------- | ------------------------------- |
| `containers`     | string (JSON) | JSON array of container objects |
| `desired_status` | string        | `RUNNING` or `STOPPED`          |

### Extracting images from ECS tasks

The `containers` column is a JSON string. Use `json_extract_path_text` with CAST to extract fields:

```sql
SELECT
    json_extract_path_text(CAST(containers AS JSON), '0', 'image') AS image,
    json_extract_path_text(CAST(containers AS JSON), '0', 'image_digest') AS digest,
    desired_status
FROM aws.ecs_task
WHERE json_extract_path_text(CAST(containers AS JSON), '0', 'image') IS NOT NULL
```

For tasks with multiple containers, UNION ALL across indices `'0'`, `'1'`, `'2'`, `'3'`, etc.

## EC2 Host Vulnerability Coverage (Method 3)

**Join key:** `dd.security_findings.resource_name = aws.ec2_instance.instance_id`

```sql
WITH host_vulns AS (
    SELECT
        resource_name, status,
        MAX(CASE WHEN status IN ('open', 'in-progress') THEN 1 ELSE 0 END) AS has_open_vuln
    FROM dd.security_findings(
        columns => ARRAY ['@resource_name', '@resource_type', '@status'],
        finding_types => ARRAY ['host_and_container_vulnerability']
    ) AS (resource_name VARCHAR, resource_type VARCHAR, status VARCHAR)
    WHERE resource_type = 'host'
    GROUP BY resource_name, status
),
scanned_hosts AS (
    SELECT resource_name, MAX(has_open_vuln) AS has_open_vuln
    FROM host_vulns
    GROUP BY resource_name
),
ec2_instances AS (
    SELECT instance_id, account_id FROM aws.ec2_instance
),
total_counts AS (
    SELECT
        COUNT(DISTINCT e.instance_id) AS total_ec2,
        COUNT(DISTINCT CASE WHEN s.resource_name IS NOT NULL THEN e.instance_id END) AS ec2_scanned,
        COUNT(DISTINCT CASE WHEN s.has_open_vuln = 1 THEN e.instance_id END) AS ec2_with_vulnerabilities,
        COUNT(DISTINCT CASE WHEN s.resource_name IS NULL THEN e.instance_id END) AS ec2_not_scanned
    FROM ec2_instances e
    LEFT JOIN scanned_hosts s ON s.resource_name = e.instance_id
)
SELECT
    CASE WHEN total_ec2 = 0 THEN 0
         ELSE (ec2_scanned * 100.0 / total_ec2)
    END AS "EC2 Hosts scanned %"
FROM total_counts
```

## Lambda Library Vulnerability Coverage (Method 3)

**Join key:** `dd.security_findings.resource_name = aws.lambda_function.function_name`

The agentless scanner uses the Lambda **function name** (from `sbomEntity.getId()`) as the `resource_name` on library vulnerability findings. This means:
- The join key is `function_name`, **NOT** `tags->'service'`
- No Datadog APM service tag is required for Lambda vulns to appear in Code Security
- Lambda findings always have `finding_type = library_vulnerability`, `resource_type = service`, `origin = agentless-scanner`
- The `enrolledEnvironments` gate (SCA activation) is the only backend filter — not a service tag

```sql
WITH lambda_lib_vulns AS (
    SELECT resource_name, status,
        MAX(CASE WHEN status IN ('open', 'in-progress') THEN 1 ELSE 0 END) AS has_open_vuln
    FROM dd.security_findings(
        columns => ARRAY ['@resource_name', '@resource_type', '@status'],
        finding_types => ARRAY ['library_vulnerability']
    ) AS (resource_name VARCHAR, resource_type VARCHAR, status VARCHAR)
    WHERE resource_type = 'service'
    GROUP BY resource_name, status
),
scanned_services AS (
    SELECT resource_name, MAX(has_open_vuln) AS has_open_vuln
    FROM lambda_lib_vulns GROUP BY resource_name
),
all_lambdas AS (
    SELECT function_name, function_arn, package_type, runtime
    FROM aws.lambda_function
),
total_counts AS (
    SELECT
        COUNT(DISTINCT l.function_arn) AS total_lambdas,
        COUNT(DISTINCT CASE WHEN s.resource_name IS NOT NULL THEN l.function_arn END) AS lambdas_scanned,
        COUNT(DISTINCT CASE WHEN s.has_open_vuln = 1 THEN l.function_arn END) AS lambdas_with_vulnerabilities,
        COUNT(DISTINCT CASE WHEN s.resource_name IS NULL THEN l.function_arn END) AS lambdas_not_scanned
    FROM all_lambdas l
    LEFT JOIN scanned_services s ON s.resource_name = l.function_name
)
SELECT
    total_lambdas,
    lambdas_scanned,
    lambdas_with_vulnerabilities,
    lambdas_not_scanned,
    CASE WHEN total_lambdas = 0 THEN 0
         ELSE (lambdas_scanned * 100.0 / total_lambdas)
    END AS "Lambda scanned %"
FROM total_counts
```

### Lambda SBOM Pipeline (from logs-backend)

Two ingestion paths exist:

1. **Agentless Scanner** (primary): CycloneDX SBOMs on the `SBOM` EvP track with `SbomSource.AGENTLESS_SCANNER`. The `AgentlessSbomDeserializer` extracts `serviceName = sbomEntity.getId()` (= Lambda function name) and `runtimeId` (= Lambda ARN from `runtime_id` tag). Sets `isServerless = true`, `ResourceType.RESOURCE_TYPE_SERVICE`.

2. **APM Telemetry** (instrumented Lambdas): `ApmTelemetryDeserializer` detects `function_arn` or `cloud_resource_type == "AWSLambda"` headers. Also sets `isServerless = true`.

Both paths produce `library_vulnerability` findings with `asset_type = SERVICE`, `origin = agentless-scanner`.

## DDSQL String Functions Reference

| Function | Works? | Notes |
|---|---|---|
| `SUBSTRING(s FROM pos FOR len)` | Yes | Extract portion of text |
| `STRPOS(s, needle)` | Yes | Find position of substring |
| `SPLIT_PART(s, delim, n)` | Yes | Split and take nth part |
| `TRIM(s)` | Yes | Remove whitespace |
| `REPLACE(s, old, new)` | Yes | String replacement |
| `LIKE '%pattern%'` | Yes | Pattern matching |
| `U&'\000a'` | Yes | Newline character |
| `json_extract_path_text(CAST(col AS JSON), ...)` | Yes | JSON field extraction |
| `E'\n'` | **No** | Use `U&'\000a'` instead |
| `CHR(10)` | **No** | Use `U&'\000a'` instead |
| `SUBSTR(...)` | **No** | Use `SUBSTRING` instead |
| `REGEXP_EXTRACT(...)` | **No** | Not available |

## Limitations

- **`dd.security_findings()`, `dd.containers`, and dot-notation tables (e.g., `aws.ecs_task`) only work via the Analysis Workspace API (Method 3).** Both pup and the `/api/v2/ddsql/user/table` endpoint return parse errors or "missing table" for these. Use Method 3 or the Datadog MCP `search_datadog_security_findings` tool for findings.
- No `SHOW TABLES`, `DESCRIBE TABLE`, or `information_schema`
- No `INSERT`, `UPDATE`, `DELETE` -- read-only queries only
- Query results scoped to the authenticated user's org visibility
- `<nil>` values in pup output represent NULL fields
- Analysis Workspace API is on an unstable path (`/api/unstable/...`) -- may change without notice

## API Details

### DDSQL Resource API (Methods 1 & 2)

- **Endpoint:** `POST https://app.datadoghq.com/api/v2/ddsql/user/table`
- **Also works:** `POST https://app.datadoghq.com/api/v2/ddsql/table` (no `user/` prefix)
- **Auth:** `DD-API-KEY` + `DD-APPLICATION-KEY` headers, or OAuth2 bearer token
- **Base URL:** `app.datadoghq.com` (NOT `api.datadoghq.com`)
- **Supports:** Cloud resource tables (`aws_*`, `gcp_*`, `azure_*`), `dd.hosts`
- **Does NOT support:** `dd.security_findings()` table functions

### Analysis Workspace API (Method 3)

- **Endpoint:** `POST https://app.datadoghq.com/api/unstable/logs/analysis-workspace/query/scalar`
- **Auth:** `DD-API-KEY` + `DD-APPLICATION-KEY` headers
- **Base URL:** `app.datadoghq.com`
- **Supports:** Everything from DDSQL resource API **plus** `dd.security_findings()`, `dd.containers`, dot-notation tables (`aws.ec2_instance`, `aws.ecr_image`, `aws.ecs_task`, `aws.lambda_function`), and cross-table JOINs
- **Note:** Unstable endpoint, format may change. Verified working 2026-02-13. As of 2026-03-02, this endpoint returns 401 with dd-auth provisioned keys and some Keychain-stored keys. If blocked, fall back to `pup security findings search` (max 100 results per call) + DDSQL resource API for client-side JOINs.
