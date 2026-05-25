# grafana-iac

Infrastructure as Code for AWS Managed Grafana, managed via GitHub Actions CI/CD workflows.

This repository provisions and manages an AWS Managed Grafana Workspace using CloudFormation, and keeps dashboards, alerts, and other Grafana resources version-controlled and automatically deployed.

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── cfn-deploy-grafana.yml    # Deploys the CloudFormation stack on merge to main
├── alerts/                           # Grafana alert rule definitions
├── cloudformation/
│   └── grafana-workspace.yml         # CloudFormation template for the Grafana Workspace
├── dashboards/                       # Grafana dashboard JSON definitions
└── README.md
```

---

## Workflows

### `cfn-deploy-grafana.yml`

Triggered on every merge to `main`. Deploys (or updates) the AWS CloudFormation stack that provisions the Managed Grafana Workspace.

**What it does:**
- Authenticates to AWS using repository secrets
- Runs `aws cloudformation deploy` with the template in `cloudformation/grafana-workspace.yml`
- Waits for the stack operation to complete and reports the result

---

## Prerequisites

- An AWS account with permissions to create and manage CloudFormation stacks and Managed Grafana Workspaces
- GitHub repository secrets configured (see below)

---

## Required Secrets

Configure these in **Settings → Secrets and variables → Actions**:

| Secret                  | Description                          |
|-------------------------|--------------------------------------|
| `AWS_ACCESS_KEY_ID`     | IAM user or role access key ID       |
| `AWS_SECRET_ACCESS_KEY` | IAM user or role secret access key   |

> It is recommended to follow the principle of least privilege — the IAM credentials should only have the permissions required to manage the Grafana workspace stack.

---

## Getting Started

1. **Clone the repository**
   ```bash
   git clone https://github.com/sgomezf/grafana-iac.git
   cd grafana-iac
   ```

2. **Configure AWS secrets** in your GitHub repository settings (see table above).

3. **Customize the CloudFormation template** at `cloudformation/grafana-workspace.yml` to match your desired workspace configuration (name, authentication providers, data sources, etc.).

4. **Add dashboards** by placing Grafana dashboard JSON files in the `dashboards/` directory.

5. **Add alert rules** by placing alert definitions in the `alerts/` directory.

6. **Push to `main`** — the deploy workflow will automatically run and provision your Grafana workspace.


---

## Contributing

1. Create a feature branch from `main`.
2. Make your changes to dashboards, alerts, or the CloudFormation template.
3. Open a pull request — review the planned changes before merging.
4. Merge to `main` to trigger the automated deployment.
