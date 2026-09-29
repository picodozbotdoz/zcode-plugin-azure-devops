# zcode-plugin-azure-devops

A ZCode plugin that lets the agent operate **Azure DevOps** through the Azure CLI's
official `azure-devops` extension - and nothing else.

## What it does

The plugin ships one skill, `azure-devops`, which teaches the agent to drive Azure DevOps
with the `az` CLI:

- `az devops` - organizations, projects, teams, users, wikis, security, generic REST (`az devops invoke`)
- `az repos` - Git repositories, branches/refs, pull requests, branch policies, repo import
- `az pipelines` - YAML pipelines, runs, artifacts, builds, variable groups, classic releases, agent pools
- `az boards` - work items, relations, WIQL queries, area paths, iterations
- `az artifacts` - universal packages

## What it deliberately does NOT do

- **No authentication.** It never runs `az login` or `az devops login`. If the CLI is not
  authenticated (and `AZURE_DEVOPS_EXT_PAT` is not set), the skill's preflight gate prints
  what is missing and stops.
- **No extension installation.** It never runs `az extension add` / `remove` / `az upgrade`.
  A missing `azure-devops` extension is reported and the task stops.
- **No non-DevOps Azure features.** No `az vm`, `az storage`, `az webapp`, `az aks`,
  `az account`, etc. The skill explicitly forbids them.
- **No config mutation.** No `az configure` / `az devops configure --defaults`; it passes
  explicit `--organization` / `--project` per call.

## Prerequisites (manual, one-time, on the machine running ZCode)

1. Azure CLI - <https://aka.ms/azure-cli>
2. `az extension add --name azure-devops`
3. Authenticate, either:
   - `az login` (Entra ID session), or
   - `export AZURE_DEVOPS_EXT_PAT=<personal access token>` with a PAT that has the
     scopes your tasks need

Tested with azure-cli 2.90.0 and azure-devops extension 1.0.8.

## Install into ZCode

This repository is itself a plugin marketplace (catalog at
`.claude-plugin/marketplace.json`, plugin source at `azure-devops/`):

1. Clone the repository (or download it) onto the machine where ZCode runs.
2. In ZCode, open **Plugin Marketplace → Add → Add Plugin Marketplace**.
3. Paste the cloned repository directory (the folder that contains
   `.claude-plugin/marketplace.json`) and add it.
4. Open **Personal → picodoz-azure-devops → Azure DevOps (az CLI)** and click **Install**.

The plugin appears under **Settings → Plugins**, where it can be enabled or disabled.

## Try it

In a new task, select the installed **Azure DevOps (az CLI)** skill in the composer picker
and send something like:

> List the active pull requests in project `Contoso` of organization `fabrikam`,
> showing title, branch and reviewers.

Expected: the agent first runs the preflight gate (checks `az`, the `azure-devops`
extension and authentication - printing `AZURE-DEVOPS PREFLIGHT: OK ...`), then runs
`az repos pr list ... --output json` and summarizes the results. If any prerequisite is
missing, it instead reports exactly which one and stops.

## Repository layout

```text
.claude-plugin/marketplace.json     marketplace catalog (this repo is the market root)
azure-devops/                       plugin source
  .zcode-plugin/plugin.json         plugin manifest
  skills/azure-devops/SKILL.md      the skill: scope, preflight gate, conventions, safety
  skills/azure-devops/references/   az devops CLI command reference (loaded on demand)
```

## License

[MIT](LICENSE)
