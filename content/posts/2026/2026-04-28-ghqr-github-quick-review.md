---
author: Carlos Mendible
categories:
- devops
date: "2026-04-28"
description: "GHQR: GitHub Quick Review"
draft: false
tags: ["github", "compliance", "security", "ghqr"]
title: "GHQR: GitHub Quick Review"
---

If you are looking for a way to quickly assess the configuration of your GitHub organizations and repositories, you might want to try [GitHub Quick Review (ghqr)](https://github.com/microsoft/ghqr): a CLI tool that scans your GitHub enterprises, organizations, and repositories and generates a detailed report with findings and recommendations based on the [GitHub Well-Architected Framework](https://wellarchitected.github.com).

## The GitHub Well-Architected Framework

The [GitHub Well-Architected Framework](https://wellarchitected.github.com) is a community-driven set of best practices curated by GitHub's own engineers, partners, and customers. It's structured around five pillars:

- **Productivity** – streamlined workflows and automation to accelerate team velocity.
- **Collaboration** – transparent project management and inclusive practices.
- **Application Security** – embedding security at every stage of development.
- **Governance** – policies and controls that balance innovation with oversight.
- **Architecture** – scalable, resilient GitHub environments at enterprise scale.

[ghqr](https://github.com/microsoft/ghqr) acts as an automated lens over these pillars, translating them into concrete, actionable findings against your actual GitHub configuration. Think of it as a Well-Architected review you can run from your terminal in seconds — covering everything from branch protections and secret scanning to Copilot policies and audit log streaming.

## What ghqr evaluates

The tool covers a broad surface area, spanning both organization-level and repository-level settings:

| Area | Scope | What it looks at |
|------|-------|---------|
| **Security** | Org, Repo | Dependabot alerts, secret scanning, code scanning, GHAS enablement |
| **Access Control** | Org, Repo | 2FA enforcement, member privileges, SAML SSO, CODEOWNERS files |
| **Branch Protection** | Repo | Required reviews, status checks, admin enforcement, force push restrictions |
| **Copilot** | Org | Seat usage, content exclusions, policy settings, MCP configuration |
| **Governance** | Org | IP allow lists, repository creation policies, fork policies |
| **Audit Log** | Enterprise, Org | Audit log streaming configuration |
| **Community** | Repo | Contributing guide, issue templates, code of conduct presence |
| **Actions** | Org, Repo | Workflow permissions, allowed actions, self-hosted runner exposure |
| **Dependencies** | Repo | Dependabot version updates, security updates configuration |
| **Metadata** | Repo | Description, topics, visibility, archival status |

Findings are prioritized by severity and category, so you can focus remediation efforts where they matter most. The output is available in **Markdown**, **Excel (.xlsx)** (default), and **JSON** — making it easy to attach results to a ticket, share with other teams, or feed into a pipeline.

## Installation

Pick whatever fits your environment:

### Linux / macOS

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/microsoft/ghqr/main/scripts/install.sh)"
```

### Windows

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/microsoft/ghqr/main/scripts/install.ps1'))
```

### Docker

```bash
docker pull ghcr.io/microsoft/ghqr:latest
```

You can also grab a binary directly from the [releases page](https://github.com/microsoft/ghqr/releases).

## Authentication

ghqr uses a **Personal Access Token (PAT)** set as an environment variable:

```bash
export GITHUB_TOKEN=<your-personal-access-token>
```

To get meaningful results across all check categories, make sure your token carries the right scopes:

| Scope | Why it's needed |
|-------|---------|
| `read:org` | Organization settings and member list |
| `read:enterprise` | Enterprise-level settings |
| `repo` | Repository settings, branch protection, security features |
| `read:audit_log` | Audit log streaming configuration |
| `read:user` | User information |
| `copilot` | Copilot seat and policy data |

> A token with missing scopes will still run, but findings for those areas will be skipped silently. Double-check your scopes if results look incomplete.

## Running your first scan

Scanning an organization is a single command:

```bash
ghqr scan -o my-org
```

For a full GitHub Enterprise scan:

```bash
ghqr scan -e my-enterprise
```

If your organization uses **GitHub Enterprise Cloud with Data Residency** (ghe.com), point ghqr at your custom subdomain:

```bash
ghqr scan -o my-org -H mycompany.ghe.com
# or
export GH_HOST=mycompany.ghe.com
ghqr scan -o my-org
```

For large enterprises, don't worry about rate limits — ghqr handles GitHub API rate limiting automatically with exponential backoff.

## Going further: the MCP Server

One of the more interesting features of ghqr is its built-in **MCP (Model Context Protocol) server**, which lets AI assistants like GitHub Copilot trigger scans and query results conversationally. This opens up workflows like asking Copilot directly: *"What are the branch protection issues in my org?"* and getting a structured answer back without leaving your editor.

Start the MCP server in stdio mode for IDE integration:

```bash
ghqr mcp
```

Or expose it over HTTP for remote or web-based access:

```bash
ghqr mcp --mode http --addr :8080
```

To wire it up with VS Code and GitHub Copilot, add this to your `.vscode/mcp.json`:

```json
{
  "servers": {
    "ghqr": {
      "type": "stdio",
      "command": "ghqr",
      "args": ["mcp"],
      "env": {
        "GITHUB_TOKEN": "${input:githubToken}"
      }
    }
  }
}
```

Once configured, the following MCP tools become available to your AI assistant:

| Tool | Description |
|------|-------------|
| `scan` | Scan GitHub enterprises, organizations, or repositories for best practices and security recommendations |

## A practical complement to the Well-Architected assessment

The [GitHub Well-Architected Framework](https://wellarchitected.github.com) provides self-guided assessment checklists for each pillar. ghqr is a great way to back those checklists with real data from your environment — instead of answering *"Is secret scanning enabled?"* from memory, you get a concrete answer with a list of affected repositories and a direct link to the remediation documentation.

> Running ghqr before a Well-Architected review gives you a head start and a clear remediation backlog to work from.

If you have any feedback, questions or issues, head over to the GitHub issues page: https://github.com/microsoft/ghqr/issues

Hope it helps!

**References:**

* [ghqr GitHub repo](https://github.com/microsoft/ghqr)
* [GitHub Well-Architected Framework](https://wellarchitected.github.com)
