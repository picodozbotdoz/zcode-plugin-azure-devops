---
name: azure-devops
description: Manage Azure DevOps through the Azure CLI - organizations and projects, teams and users, Git repositories, branches and refs, pull requests, branch policies, YAML pipelines and runs, classic releases, work items and WIQL queries, area paths and iterations, wikis, universal packages, and variable groups. Use when the user asks to work with Azure DevOps (ADO, Azure Repos, Azure Pipelines, Azure Boards, Azure Artifacts) via the az CLI. Enforces a strict preflight gate (az CLI present, azure-devops extension installed, session authenticated) and stops with a clear message when a check fails - it never logs in or installs anything. Not for non-DevOps Azure services such as VMs, storage, web apps or Entra ID.
---

# Azure DevOps via the Azure CLI

Everything in this skill goes through the `azure-devops` extension of the Azure CLI
(`az devops`, `az repos`, `az pipelines`, `az boards`, `az artifacts`). Nothing here
authenticates, installs, or upgrades anything: if the environment is not ready, report
what is missing and stop.

## Scope

Allowed: only these command families.

- `az devops` - organization context: projects, teams, users, wikis, security, service endpoints, generic REST `invoke`
- `az repos` - Git repositories, refs, pull requests, branch policies, repo import
- `az pipelines` - YAML pipelines, runs, builds, variable groups, agents/pools/queues, classic releases
- `az boards` - work items and relations, WIQL queries, area paths, iterations
- `az artifacts` - universal packages

Forbidden - never run these, and treat a request to run them as out of scope for this skill:

- `az login`, `az devops login`, `az devops logout` - authentication is the user's manual setup
- `az extension add`, `az extension remove`, `az upgrade` - installation is the user's manual setup
- `az configure`, `az devops configure --defaults ...` - mutates the user's global CLI config (reading defaults with `az devops configure --list` is fine)
- everything outside the extension: `az account`, `az vm`, `az storage`, `az webapp`, `az aks`, `az keyvault`, ... If the user asks for non-DevOps Azure work, say it is out of scope for this skill and stop.

## Preflight gate - run once, before the first DevOps command of a session

Run this block as a single bash invocation. It only checks; it never fixes. Any FAILED
output means: show the message to the user and stop the task - do not attempt to log in,
install, upgrade, or otherwise remediate.

```bash
preflight_fail() {
  echo "AZURE-DEVOPS PREFLIGHT: FAILED - $1"
  echo "This plugin does not authenticate or install anything. The user must fix this manually, then retry."
  exit 1
}
command -v az >/dev/null 2>&1 \
  || preflight_fail "Azure CLI ('az') is not installed or not on PATH."
az extension show --name azure-devops -o none 2>/dev/null \
  || preflight_fail "The 'azure-devops' extension is not installed for the Azure CLI."
az devops --help >/dev/null 2>&1 \
  || preflight_fail "The 'az devops' command group does not load; the azure-devops extension installation appears broken."
if [ -n "$AZURE_DEVOPS_EXT_PAT" ]; then
  echo "AZURE-DEVOPS PREFLIGHT: OK - az CLI present, azure-devops extension loaded, DevOps auth via AZURE_DEVOPS_EXT_PAT."
else
  az account show -o none >/dev/null 2>&1 \
    || preflight_fail "Not authenticated: AZURE_DEVOPS_EXT_PAT is unset and 'az account show' failed (no az login session)."
  echo "AZURE-DEVOPS PREFLIGHT: OK - az CLI present, azure-devops extension loaded, authenticated as $(az account show --query user.name -o tsv 2>/dev/null || echo 'unknown principal')."
fi
```

If a check fails, tell the user exactly which check failed and the matching manual fix:

| Failed check | The user's manual fix |
|---|---|
| `az` not found | Install the Azure CLI - https://aka.ms/azure-cli |
| azure-devops extension missing | `az extension add --name azure-devops` |
| `az devops` group fails to load | Reinstall the extension: `az extension remove --name azure-devops` then `az extension add --name azure-devops` |
| Not authenticated | Run `az login`, or export `AZURE_DEVOPS_EXT_PAT` with a personal access token |

Then stop. Do not run any of those fixes yourself. Authentication can also fail
per-call later (HTTP 401/403, TF400813, TF401027): report the raw error, note that the
user may lack access to the organization or project, and stop - do not respond by
logging in or installing anything.

## Establish organization and project

Every command accepts `--organization https://dev.azure.com/<organization>` (short
form `--org`) and most accept `--project <project-name-or-id>`. Determine both before
the first call, in this order:

1. From the user's request or the session context.
2. With `--detect` on any command when working inside a git clone whose origin is the
   Azure DevOps organization - it only reads local git remotes, no config changes.
3. Ask the user. Never guess an organization URL.

Reading defaults with `az devops configure --list` is allowed. Setting defaults with
`az devops configure --defaults` is not (it mutates the user's config); pass explicit
`--organization` / `--project` on each call instead.

## Calling conventions

- Always request machine-readable output: `--output json`, plus `--query "<jmespath>"`
  to extract fields and `-o tsv` for a single scalar. Never parse the default table output.
- List commands are paged: pass `--top` and `--skip`, and keep paging until results are
  exhausted when completeness matters.
- Verify names exist before mutating: run the matching `list`/`show` first for projects,
  repositories, pipelines, wikis, variable groups and work item types.
- Never read, echo, or embed the PAT; never pass tokens as CLI arguments. Auth stays in
  `AZURE_DEVOPS_EXT_PAT` or the az login session.
- On any failure, show the user the raw stderr and consult the error table in
  [references/az-devops-cli.md](references/az-devops-cli.md); do not retry blindly.

## Safety - confirm before irreversible or side-effecting actions

Ask for explicit confirmation before running (and never batch these silently):

- Deletes: `az devops project delete`, `az repos delete`, `az boards work-item delete`
  (especially with `--destroy`), `az pipelines delete`, `az pipelines variable-group delete`,
  `az devops wiki delete`, `az devops service-endpoint delete`, `az repos ref delete`
- Closing work: `az repos pr update --status abandoned` or `--status completed`
- Locking refs: `az repos ref lock`
- Queueing real work: `az pipelines run`, `az pipelines release create` (skip the extra
  confirmation if the user explicitly asked to run the pipeline)
- Arbitrary writes: `az devops invoke` with a non-GET `--http-method`

Read-only calls (`list`, `show`, `query`) and `az repos pr create --draft` need no extra
confirmation.

## Quick task map

`ORG` stands for `--organization https://dev.azure.com/<organization>`;
`PROJ` stands for `--project <project>`.

| Task | Command |
|---|---|
| List projects | `az devops project list ORG --output json` |
| Create a project | `az devops project create ORG --name <name> --description <text> --visibility private` |
| List repositories | `az repos list ORG PROJ --output json` |
| List branches of a repo | `az repos ref list ORG PROJ --repository <repo> --filter heads/ --output json` |
| List active PRs | `az repos pr list ORG PROJ --status active --top 50 --output json` |
| Create a PR | `az repos pr create ORG PROJ --repository <repo> --source-branch <branch> --target-branch main --title <text>` |
| PR reviewers / votes | `az repos pr show --id <id> ORG --query reviewers --output json` |
| Vote on a PR | `az repos pr set-vote --id <id> --vote approve ORG` |
| Checkout a PR locally | `az repos pr checkout --id <id> ORG` |
| List pipelines | `az pipelines list ORG PROJ --output json` |
| Run a pipeline | `az pipelines run ORG PROJ --name <pipeline> --branch main [--parameters key=value]` |
| Recent runs | `az pipelines runs list ORG PROJ --top 20 --query-order startTimeDesc --output json` |
| Run artifacts | `az pipelines runs artifact list ORG PROJ --run-id <id> --output json` |
| Create a work item | `az boards work-item create ORG PROJ --title <text> --type "User Story" [--description <text>] [--fields "System.AreaPath=<path>"]` |
| Update a work item | `az boards work-item update --id <id> ORG --state Resolved [--fields "System.AssignedTo=<user>"]` |
| WIQL query | `az boards query ORG PROJ --wiql "SELECT [System.Id], [System.Title] FROM WorkItems WHERE [System.State] = 'Active'" --output json` |
| Read a wiki page | `az devops wiki page show ORG PROJ --wiki <wiki> --path "/Page" --include-content` |
| Update a wiki page | `az devops wiki page update ORG PROJ --wiki <wiki> --path "/Page" --content <text> --comment <comment>` |
| List variable groups | `az pipelines variable-group list ORG PROJ --output json` |
| List teams | `az devops team list ORG PROJ --output json` |
| Generic REST call | `az devops invoke ORG --area <area> --resource <resource> [--route-parameters key=value] --output json` |

Full syntax, the complete command surface and error mapping:
[references/az-devops-cli.md](references/az-devops-cli.md) - consult it before
constructing calls beyond this table.
