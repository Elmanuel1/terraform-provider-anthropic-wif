# terraform-provider-anthropic

> Manage **Anthropic Managed Agents** — agents, the environments they run in, scheduled deployments, skills, and MCP credential vaults — as Terraform.

[![Terraform Registry](https://img.shields.io/badge/registry-Elmanuel1%2Fanthropic-7B42BC?logo=terraform)](https://registry.terraform.io/providers/Elmanuel1/anthropic/latest)

**Registry:** [Elmanuel1/anthropic](https://registry.terraform.io/providers/Elmanuel1/anthropic/latest) · **WIF setup:** [authentication guide](docs/guides/authentication.md) · **Changelog:** [CHANGELOG.md](CHANGELOG.md)

---

## Get started in 60 seconds

The fastest path uses a workspace API key. (Running in CI? Skip to [Authentication](#authentication) and use WIF instead.)

```terraform
terraform {
  required_providers {
    anthropic = {
      source  = "Elmanuel1/anthropic"
      version = "<LATEST_VERSION>" # see the registry: https://registry.terraform.io/providers/Elmanuel1/anthropic/latest
    }
  }
}

provider "anthropic" {
  workspace_api_key = var.anthropic_workspace_api_key # sk-ant-api03-...
}

resource "anthropic_agent" "hello" {
  name   = "hello-agent"
  model  = "claude-sonnet-4-6"
  system = "You are a helpful assistant."
}
```

```bash
export TF_VAR_anthropic_workspace_api_key="sk-ant-api03-..."
terraform init && terraform apply
```

That's it — you now have an agent managed in Terraform. Next, give it somewhere to run (an `environment`) and put it live (a `deployment`); see [`examples/full-stack`](examples/full-stack).

---

## What you can manage

Every resource also has a matching `data.anthropic_*` data source.

| Resource | Auth | What it is |
|---|---|---|
| `anthropic_agent` | API key or WIF | An agent: model, system prompt, tools, MCP servers, skills, multiagent config |
| `anthropic_environment` | API key or WIF | Where agents run: networking, allowed hosts, packages, cloud or self-hosted |
| `anthropic_deployment` | API key or WIF | Binds an agent to an environment; optional cron `schedule`; pause/unpause |
| `anthropic_skill` | API key or WIF | A skill uploaded from a local directory containing a `SKILL.md` |
| `anthropic_vault` | API key or WIF | A workspace-scoped vault holding MCP server credentials |
| `anthropic_vault_credential` | API key or WIF | A credential in a vault (`static_bearer` or `mcp_oauth`) — secrets are write-only |
| `anthropic_workspace` | Admin key | A workspace |
| `anthropic_memory_store` | Admin key | A memory store for agent persistence |

Workspace-scoped resources (the first six) accept either a workspace API key or WIF. Org-level resources (`workspace`, `memory_store`) use the Admin API key.

---

## Authentication

Pick the method that matches where Terraform runs.

| | Workspace API key | Workload Identity Federation (WIF) |
|---|---|---|
| **Use when** | Local dev, a single workspace, trying it out | CI / Terraform Cloud and production — managing multiple workspaces, or federating an existing OIDC workload identity (Terraform Cloud, Kubernetes, AWS, GitHub Actions) |
| **Setup** | Paste one key | ~5 min one-time console setup |
| **Secrets in CI** | A long-lived key | None — short-lived tokens per run |

**API key** — set `workspace_api_key` (for all workspace-scoped resources) or `admin_api_key` (for workspaces and memory stores) in the provider block. A workspace API key is scoped to one workspace, so managing several means juggling several keys.

**WIF** — the provider exchanges an OIDC token for a short-lived, workspace-scoped token, so nothing long-lived is stored. One service account and federation rule can mint tokens for many workspaces (set `workspace_id` per resource), which is why WIF scales across a fleet better than per-workspace keys. Configure three IDs in the provider block:

```terraform
provider "anthropic" {
  federation_rule_id = var.anthropic_federation_rule_id # fdrl_...
  organization_id    = var.anthropic_organization_id    # org UUID
  service_account_id = var.anthropic_service_account_id  # svac_...
}
```

The token exchange is a standard OAuth2 `jwt-bearer` flow, so any OIDC issuer registered on Anthropic works. Terraform Cloud injects the OIDC token automatically. Other issuers — Kubernetes projected service account tokens, AWS, GitHub Actions — work too: place the JWT in `TFC_WORKLOAD_IDENTITY_TOKEN` (or `TFC_WORKLOAD_IDENTITY_TOKEN_ANTHROPIC`) and register the matching issuer in the Anthropic console.

> 📖 **Setting up WIF? Start here → [docs/guides/authentication.md](docs/guides/authentication.md)**
> The complete guide: console setup (issuer, service account, federation rule), the CEL condition, token-lifetime tuning, and every failure reason mapped to a fix. To iterate locally without Terraform Cloud, see [local WIF testing](docs/wif-local-testing.md).

When both WIF and `workspace_api_key` are set on a resource that supports either, **WIF wins**.

### Provider attributes

| Attribute | Value | Needed for |
|---|---|---|
| `admin_api_key` | `sk-ant-admin-...` | `anthropic_workspace`, `anthropic_memory_store` |
| `workspace_api_key` | `sk-ant-api03-...` | All workspace-scoped resources (agents, environments, deployments, skills, vaults, vault credentials), when not using WIF |
| `federation_rule_id` | `fdrl_...` | WIF resources |
| `organization_id` | org UUID | WIF resources |
| `service_account_id` | `svac_...` | WIF resources |

---

## Examples

Runnable configs in [`examples/`](examples):

| Example | Shows |
|---|---|
| [`full-stack`](examples/full-stack) | Workspace + agent + environment + vault wired together |
| [`wif-test`](examples/wif-test) | WIF authentication end to end |
| [`deployment-test`](examples/deployment-test) | Scheduled and on-demand deployments |
| [`skill-test`](examples/skill-test) | Uploading a skill from a local directory |
| [`workspace-test`](examples/workspace-test) | Workspace management via the Admin API |

---

## Local development

```bash
go build -o terraform-provider-anthropic .
```

```hcl
# ~/.terraformrc
provider_installation {
  dev_overrides {
    "Elmanuel1/anthropic" = "/path/to/provider/binary"
  }
  direct {}
}
```

---

## Docs

- [Authentication & WIF guide](docs/guides/authentication.md)
- [Local WIF testing with ngrok](docs/wif-local-testing.md)
- [Provider matrix](docs/guides/provider-matrix.md)
- [Resources](docs/resources) · [Data sources](docs/data-sources)
- [Changelog](CHANGELOG.md)
