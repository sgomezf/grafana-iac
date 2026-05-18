# CloudFormation GitHub Actions Deploy Flow

A CI/CD flow for deploying AWS Grafana Workspace via CloudFormation stack.

---

## Flow Overview

```
```

---

## Required Secrets

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS_ACCESS_KEY_ID |
| `AWS_SECRET_ACCESS_KEY` | AWS_SECRET_ACCESS_KEY |

---

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── cfn-deploy-grafana.yml         # Runs on merge to main — deploys
├── cloudformation/
│   ├── grafana-workspace.yml               # Your CFN template
```
