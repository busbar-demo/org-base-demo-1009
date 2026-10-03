# org-base — a Busbar sidecar for your Salesforce org

This is the **base sidecar repository** for a Salesforce org that runs
[Busbar](https://github.com/busbar-actions) as a *headless delivery platform*.
You don't fork this by hand — the **Busbar console** forks it into your GitHub
organization when you connect your first repository, and wires it to your org.

Once connected, this repo can **pull live context out of your Salesforce org
into GitHub** — schema, metadata, a dependency graph, a security audit — as
reviewable git history. **Nothing here deploys to or changes your org.** Every
workflow is read-only; the most it ever writes is a commit *back to this repo*.

## How it talks to Salesforce (zero stored credentials)

There are **no Salesforce secrets in this repository.** Each workflow grants
`permissions: id-token: write`, and the Busbar actions exchange the GitHub
Actions runner's short-lived **OIDC token** for an equally short-lived
Salesforce session — in-process, held only in memory, revoked at job end
(RFC 8693 token exchange against your Busbar-equipped org). Your org decides,
from the OIDC claims alone, whether to trust this repo + workflow.

> **First run is expected to stop with a "pending trust" error.** That's the
> security model working: a brand-new repo/workflow isn't trusted until an admin
> approves it in the Busbar console (Trust surface) — or runs
> `sf busbar trust request approve`. Approve it once, re-run, and you're live.

## Which org does it target?

Workflows read the org instance URL from the repository variable
**`SF_INSTANCE_URL`** (the Busbar console sets this for you when it forks the
repo). Every workflow also accepts a `target_instance` input so you can point a
single run at a different Busbar-equipped org without changing the variable.

## What's in the box

| Workflow | What it does | Org access |
|---|---|---|
| **Collect Org Context** (`collect-org-context.yml`) | The full snapshot: schema + metadata + dependency graph + permissions audit, committed back as a diffable mirror of your org. Run this first. | read-only |
| **Snapshot Schema** (`snapshot-schema.yml`) | Per-SObject REST `describe` JSON → `.busbar/schema/`. Schema drift as git diffs. | read-only |
| **Retrieve Metadata** (`retrieve-metadata.yml`) | Source-format metadata tree → `org/metadata/`. | read-only |
| **Analyze Dependencies** (`analyze-dependencies.yml`) | Builds a component dependency graph + Voronoi package map (JSON + SVG). | read-only |
| **Audit Permissions** (`audit-permissions.yml`) | Cedar security analysis (permissions, sharing, FLS, flows) → findings JSON + SARIF in the Security tab. | read-only |
| **Query Org** (`query-org.yml`) | Runs one declarative `sf` action (default: a SOQL query) inside the **nono** capability sandbox (Landlock-confined on Linux) with a full audit trail. | read-only |
| **Busbar Delivery** (`busbar-delivery.yml`) | Runs in a **provisioned fork**: the Busbar Hub dispatches it after creating the fork's ephemeral scratch org and GitHub Environment `org`. Authenticates by OIDC against that org's per-run trust rule, confirms the org, and writes a job summary. | read-only |

All seven are **on-demand** (`workflow_dispatch`) — kick any of them off from the
GitHub **Actions** tab, or from the Busbar console.

## Run it

1. Open the **Actions** tab.
2. Pick a workflow (start with **Collect Org Context**).
3. **Run workflow** → optionally override `target_instance` → **Run**.
4. First time from a new workflow: approve the trust request in the Busbar
   console, then re-run.

## Composition

`busbar.json` declares this repo as a Busbar **Kit** — a composable bundle of
read-only context-collection workflows — so the Exchange can describe and
distribute it. It is the seed; extend it with more
[`busbar-actions`](https://github.com/busbar-actions) as your org's needs grow.
