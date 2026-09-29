# az CLI - Azure DevOps command reference

Reference for the `azure-devops` extension (tested against azure-cli 2.x / extension 1.x).
Command names below reflect the real extension surface; the flags listed are the commonly
used ones - run `az <command> --help` for the complete set. See the SKILL.md scope rules:
`az devops login` / `az devops logout` / `az extension add|remove` are never run by the agent.

## Global conventions

- `--organization <url>` (short `--org`): `https://dev.azure.com/<organization>`. Required on nearly every call unless `--detect` can infer it.
- `--project <name-or-id>`: required by project-scoped commands.
- `--detect`: infer org/project from the git remote of the current directory. Only works inside a clone of the project.
- `--output json` + `--query "<jmespath>"` for parsing; `-o tsv` for one scalar; `--only-show-errors` to silence warnings.
- Paging: `--top <n>`, `--skip <n>`.
- `az devops configure --list` shows current defaults (read-only). Never run `az devops configure --defaults` or `az configure`.

## `az devops` - organization context

Subgroups: `admin`, `extension`, `migrations` (preview), `project`, `security`,
`service-endpoint`, `team`, `user`, `wiki`. Top-level commands: `configure`, `invoke`,
`login`, `logout` (login/logout are user-managed authentication - never run them).

```bash
# Projects
az devops project list --organization <org> --top 100 --output json
az devops project show --organization <org> --project <project> --output json
az devops project create --organization <org> --name <name> --description <text> \
  --visibility private --source-control git
az devops project delete --organization <org> --id <project-id> --yes   # CONFIRM FIRST

# Teams
az devops team list --organization <org> --project <project> --output json
az devops team list-member --organization <org> --project <project> --team <team> --output json

# Users (organization level)
az devops user list --organization <org> --top 100 --output json
```

`az devops security group` and `az devops security permission` manage ACLs - read-only
listing is useful for diagnosing access problems; permission changes need explicit
user confirmation.

### `az devops invoke` - generic REST

For any DevOps REST endpoint the extension does not wrap:

```bash
az devops invoke --organization <org> --area git --resource repositories \
  --route-parameters project=<project> --api-version 7.1 --output json
```

Key flags: `--area`, `--resource`, `--route-parameters key=value`, `--query-parameters
key=value`, `--http-method` (non-GET writes need explicit user confirmation), `--in-file`,
`--out-file`, `--api-version`. Use JSON output only; the response shape is not fixed.

## `az repos` - Git

Subgroups: `import`, `policy`, `pr`, `ref`.

```bash
az repos list --organization <org> --project <project> --output json
az repos show --organization <org> --project <project> --repository <repo> --output json
az repos create --organization <org> --project <project> --name <name>
az repos delete --organization <org> --project <project> --id <repo-id> --yes   # CONFIRM FIRST

# Refs / branches
az repos ref list --organization <org> --project <project> --repository <repo> \
  --filter heads/ --output json
az repos ref lock --organization <org> --project <project> --repository <repo> \
  --name refs/heads/main     # CONFIRM FIRST
```

Note: ref commands take fully qualified names (`refs/heads/<branch>`); `pr create`
takes the bare branch name.

### `az repos pr` - pull requests

Subgroups: `policy`, `reviewer`, `work-item` (PR<->work-item links).

```bash
az repos pr list --organization <org> --project <project> [--repository <repo>] \
  [--status active|abandoned|all|completed] [--source-branch <b>] [--target-branch <b>] \
  [--creator <user>] [--top 50 --skip 0] --output json
az repos pr show --id <id> --organization <org> --output json
az repos pr create --organization <org> --project <project> --repository <repo> \
  --source-branch <branch> --target-branch main --title <text> [--description <text>] \
  [--draft] [--work-items <id> <id>]
az repos pr update --id <id> --organization <org> [--title <text>] [--description <text>] \
  [--status active|abandoned|completed] [--auto-complete] [--squash] \
  [--delete-source-branch] [--merge-commit-message <text>]   # status/abandon: CONFIRM FIRST
az repos pr set-vote --id <id> --organization <org> --vote approve|approve-with-suggestions|reset|reject|wait
az repos pr checkout --id <id> --organization <org>   # checks out the PR branch locally
az repos pr reviewer list --id <id> --organization <org> --output json
```

`--bypass-policy` exists but must only be used when the user explicitly asks for it.

## `az pipelines` - pipelines, builds, releases

Subgroups: `agent`, `build`, `folder`, `pool`, `queue`, `release`, `runs`,
`variable`, `variable-group`.

```bash
az pipelines list --organization <org> --project <project> [--name <p>] [--top 50] --output json
az pipelines show --organization <org> --project <project> --name <pipeline> --output json
az pipelines run --organization <org> --project <project> --name <pipeline> \
  [--branch main] [--parameters key=value ...] [--variables key=value ...]   # queues real work
az pipelines runs list --organization <org> --project <project> \
  [--pipeline-ids <id> ...] [--branch <b>] [--status ...] [--result succeeded|failed|canceled] \
  [--top 20 --query-order startTimeDesc] --output json
az pipelines runs show --organization <org> --project <project> --id <run-id> --output json
az pipelines runs artifact list --organization <org> --project <project> --run-id <id> --output json

# Variable groups
az pipelines variable-group list --organization <org> --project <project> --output json
az pipelines variable-group show --group-id <id> --organization <org> --output json
az pipelines variable-group create --organization <org> --project <project> \
  --name <name> --variables key=value ... [--authorize]
az pipelines variable-group delete --group-id <id> --organization <org> --yes   # CONFIRM FIRST
```

Classic releases: `az pipelines release list|create|show` and
`az pipelines release definition`. Queueing a release is real deployment work -
confirm first.

## `az boards` - work tracking

Subgroups: `area`, `iteration`, `work-item`.

```bash
az boards query --organization <org> --project <project> \
  --wiql "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.State] = 'Active'" \
  --output json
# or by stored query: --id <query-id>  /  --path "Shared Queries/My Query"

az boards work-item create --organization <org> --project <project> \
  --title <text> --type "User Story" [--description <text>] [--area <path>] [--iteration <path>] \
  [--assigned-to <user>] [--fields "System.Tags=foo;bar"]
az boards work-item show --id <id> --organization <org> [--expand all] --output json
az boards work-item update --id <id> --organization <org> [--title <text>] [--state <state>] \
  [--description <text>] [--assigned-to <user>] [--iteration <path>] [--fields "Key=Value"]
az boards work-item delete --id <id> --organization <org> [--destroy] --yes   # CONFIRM FIRST
az boards work-item relation add --id <id> --organization <org> \
  --relation-type parent --target-id <id>
```

Work item types, states and area/iteration paths are project-specific - list them
(`az boards work-item ... --help`, `az boards area list`, `az boards iteration list`)
before assuming values.

## `az artifacts` - packages

```bash
az artifacts universal download --organization <org> --project <project> \
  --feed <feed> --name <package> --version <version> --path <dir>
az artifacts universal publish --organization <org> --project <project> \
  --feed <feed> --name <package> --version <version> --path <dir> --description <text>
```

## `az devops wiki`

```bash
az devops wiki list --organization <org> --project <project> --output json
az devops wiki show --organization <org> --project <project> --wiki <wiki>
az devops wiki create --organization <org> --project <project> --name <wiki> \
  [--repository <repo> --mapped-path "/"]   # code wiki
az devops wiki page show --organization <org> --project <project> \
  --wiki <wiki> --path "/Page" --include-content
az devops wiki page create --organization <org> --project <project> \
  --wiki <wiki> --path "/NewPage" --content <text>   # or --file-path <file>
az devops wiki page update --organization <org> --project <project> \
  --wiki <wiki> --path "/Page" --content <text> --comment <comment>   # or --file-path <file>
```

## Failure patterns

| Error signature | Meaning | Action |
|---|---|---|
| `Please run 'az login' to setup account.` | No az login session (and no PAT env var) | Stop; tell the user to `az login` or export `AZURE_DEVOPS_EXT_PAT`. Never log in yourself. |
| HTTP 401 / `TF400813` | Not authorized for the organization (expired/invalid PAT, wrong tenant, no org membership) | Stop; report raw error; user fixes credentials or access. |
| HTTP 403 / `TF401027` | Authenticated, but permission denied for this operation | Stop; report raw error; user needs the permission granted. |
| `TF401019` / "does not exist" style project or repo errors | Name mismatch or no access to the project | Verify names with the matching `list` call before retrying. |
| Any other `TF...`/HTTP error | Unknown | Show raw stderr to the user, do not retry blindly. |
