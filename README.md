# grafana-iac

Infrastructure as Code for AWS Managed Grafana, managed via GitHub Actions CI/CD workflows.

This repository provisions and manages an AWS Managed Grafana Workspace using CloudFormation, and keeps dashboards, alerts, and other Grafana resources version-controlled and automatically deployed.

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── cfn-deploy-grafana.yml      # Provisions the Grafana Workspace
│       ├── cfn-deploy-config.yml       # Configures the workspace (plugins, data sources)
│       ├── cfn-deploy-alerts.yml       # Deploys alert rules, contact points & notification policies
│       └── cfn-load-dashboards.yml     # Syncs dashboard JSON files and loads them into Grafana
├── alerts/
│   ├── contact-points/                 # Contact point JSON definitions
│   ├── rules/                          # Alert rule group JSON definitions
│   └── notification-policies/         # Notification policy tree JSON
├── cloudformation/
│   ├── grafana-workspace.yml           # CFN template — Grafana Workspace
│   ├── grafana-config.yml              # CFN template — workspace configuration
│   ├── grafana-alerts.yml              # CFN template — alerts stack
│   └── grafana-dashboards.yml          # CFN template — dashboard loader stack
├── dashboards/                         # Grafana dashboard JSON definitions
└── README.md
```

---

## Workflows

### `cfn-deploy-grafana.yml` — Deploy Grafana Workspace

Provisions or updates the core AWS Managed Grafana Workspace via CloudFormation.

| Trigger | Details |
|---|---|
| Push to `main` | When `cloudformation/grafana-workspace.yml` or the workflow file changes |
| Manual (`workflow_dispatch`) | Choose environment (`development` / `production`) and action (`deploy` / `delete`) |

**Jobs:**
1. **Validate** — runs `aws cloudformation validate-template` against the workspace template.
2. **Deploy Grafana Workspace** — deploys the `grafana-<env>` CloudFormation stack with parameters like `WorkspaceName`, `GrafanaVersion` (12.4), and `OrganizationRoleName`. Outputs the workspace URL, ID, role ARN, and SSM path for the API key.

---

### `cfn-deploy-config.yml` — Deploy Grafana Configuration

Configures an existing workspace — installs plugins and sets data source parameters.

| Trigger | Details |
|---|---|
| Push to `main` | When `cloudformation/grafana-config.yml` or the workflow file changes |
| Manual (`workflow_dispatch`) | Choose environment (`development` / `production`) and action (`deploy` / `delete`) |

**Jobs:**
1. **Validate** — validates the config CloudFormation template.
2. **Deploy Grafana Configuration** — deploys the `grafana-config-<env>` stack, passing workspace ID/endpoint and plugin IDs (e.g. `grafana-redshift-datasource`) as parameters.

---

### `cfn-deploy-alerts.yml` — Deploy Grafana Alerts

Manages alert rules, contact points, and notification policies. Alert JSON files are uploaded to S3 and loaded into Grafana via a Lambda-backed CloudFormation stack.

| Trigger | Details |
|---|---|
| Push to `main` | When `cloudformation/grafana-alerts.yml`, `alerts/**`, or the workflow file changes |
| Manual (`workflow_dispatch`) | Choose environment (`development` / `production`) and action (`deploy` / `delete`) |

**Jobs:**
1. **Validate** — validates the CloudFormation template and checks the `alerts/` folder structure. Verifies JSON syntax for all files and confirms required fields (`name`, `folderUID`, `rules`) on alert rule groups.
2. **Bootstrap S3 Bucket** — creates the `grafana-alerts-<env>-<account-id>` bucket if it doesn't exist, with versioning, AES-256 encryption, and public access blocked.
3. **Upload Alerts → S3** — resolves secrets (e.g. `SNS_TOPIC_ARN`, `CLOUDWATCH_DATASOURCE_UID`) via `envsubst` into a temporary copy of the alert files, then syncs each subfolder to its corresponding S3 prefix. Fails if any placeholders remain unresolved.
4. **Deploy CFN Stack** — deploys the `grafana-alerts-<env>` stack. Increments `AlertsVersion` on every run to guarantee the Lambda re-applies all alert resources.

---

### `cfn-load-dashboards.yml` — Load Grafana Dashboards

Syncs dashboard JSON files from the `dashboards/` folder to S3 and loads them into Grafana via a Lambda-backed CloudFormation stack.

| Trigger | Details |
|---|---|
| Push to `main` | When `cloudformation/grafana-dashboards.yml`, `dashboards/**`, or the workflow file changes |
| Manual (`workflow_dispatch`) | Choose environment (`development` / `production`) and action (`deploy` / `delete`) |

**Jobs:**
1. **Validate** — validates the CloudFormation template and checks that `dashboards/` contains at least one valid JSON file.
2. **Bootstrap S3 Bucket** — creates the `dashboards-<env>-<account-id>` bucket if it doesn't exist, with versioning, AES-256 encryption, and public access blocked.
3. **Upload Dashboards → S3** — syncs all `*.json` files from `dashboards/` to `s3://<bucket>/dashboards/`, deleting any files removed from the repo.
4. **Deploy CFN Stack** — deploys the `grafana-dashboard-loader-<env>` stack. Increments `DashboardsVersion` on every run to force the Lambda to re-import all dashboards.
5. **Delete** (manual only) — empties the S3 bucket and then deletes the CloudFormation stack.

---

## Required Secrets

Configure these in **Settings → Secrets and variables → Actions**, scoped per GitHub Environment (`development` / `production`):

Please note some workflows have different lifecycle than others, for example the alerts and and dashboards require the Grafana Service Account Token and Endpoint etc.

| Secret | Used by | Description |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | All workflows | IAM access key ID |
| `AWS_SECRET_ACCESS_KEY` | All workflows | IAM secret access key |
| `AWS_ACCOUNT_ID` | Alerts, Dashboards | Used to construct S3 bucket names |
| `GRAFANA_WORKSPACE_ID` | Alerts, Dashboards | AWS Managed Grafana workspace ID |
| `GRAFANA_WORKSPACE_ENDPOINT` | Alerts, Dashboards | Grafana workspace URL |
| `GRAFANA_API_KEY_SSM_PATH` | Alerts, Dashboards | SSM Parameter Store path for the Grafana service account token |
| `SNS_TOPIC_ARN` | Alerts | ARN of the SNS topic used in contact points |
| `CLOUDWATCH_DATASOURCE_UID` | Alerts | UID of the CloudWatch data source in Grafana |

> Use the principle of least privilege — IAM credentials should only have the permissions required to manage the relevant CloudFormation stacks, S3 buckets, and Grafana resources.

---
