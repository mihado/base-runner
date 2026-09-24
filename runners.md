# GitHub self-hosted runners — updates, selective routing, lockdown, templates

Operating notes for service runners that execute Docker workloads. Complements README.md (ephemeral design); this file is procedures. Examples use a `UAT` group — substitute any group name.

## Updates

- Runners auto-update by default: before each job the runner checks GitHub and self-updates, and GitHub enforces a minimum version. A year-old default install is almost certainly current — verify on the Runners settings page or `./run.sh --version`, don't assume.
- Manual path (only if registered with `--disableupdate`): `sudo ./svc.sh stop`, extract the latest tarball over the install dir (registration in `.runner`/`.credentials` survives — it's not in the tarball), `sudo ./svc.sh start`.
- Auto-update covers the runner binary only. OS patches and the Docker engine on those boxes stay yours.
- Service (non-container) runners are the right shape for Docker workloads: host socket, no DinD tax.

## Selective routing: group plus PR label plus author

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]
jobs:
  int-runner:
    name: int.runner
    if: contains(github.event.pull_request.labels.*.name, 'run-uat') && contains(fromJSON('["OWNER", "MEMBER"]'), github.event.pull_request.author_association)
    runs-on:
      group: UAT
```

- `on:` decides when the workflow wakes up. `labeled` (and `unlabeled` for re-evaluation) must be listed — defaults are opened/synchronize/reopened, and adding the label otherwise fires nothing.
- Jobs `if:` decides whether the job runs once awake. Without it every wake-up burns your iron; without `labeled` in `on`, the label does nothing until the next push. Both halves required.
- The label gates intent, the author gate locks it to trusted people: anyone with triage permission can label, so the label alone is necessary but not sufficient. `OWNER` on personal repos, `MEMBER` (and `COLLABORATOR` if you add any) on orgs.
- A skipped required check counts as satisfied in branch protection, so label-less PRs merge cleanly while labeled ones run.
- Decide Dependabot explicitly (its PRs won't match owner/member — usually what you want, but make it a decision, not an accident).

## Two tiers: default cloud CI, internal on label

- Default CI (`ci.yml`) runs on GitHub-hosted runners for every PR activity — fast, isolated, no trust questions. Keep `labeled` out of its trigger types so labeling doesn't wastefully re-run it.
- Internal jobs (`ci-uat.yml`) use the gated shape above: `labeled` in types, label plus author gate, `runs-on` on the group. Secrets, heavy GPUs, Docker builds live here only.
- Branch protection requires both. Unlabeled PRs show the internal job skipped (satisfies the requirement); labeled ones run it.
- Cost shape: cloud minutes on every push, local iron only on demand. The label is the budget switch.

## Lockdown for trusted people

- Audit triage+ now: everyone with write/triage/maintain can apply the label. Prune while small — the list only grows.
- Escalation dial: put the job in an environment with required reviewers and nothing executes until a named human approves that run. Maximum assurance, per-run friction — reserve for when the author gate stops feeling sufficient.
- CODEOWNERS and branch protection gate the merge, never the execution: the code already ran on your iron by review time.
- Fork settings stay on (approval for outside collaborators) for the public flip; the author gate is what actually keeps fork code off your runners.

## Naming: IDs, display names, groups

- Job IDs forbid dots (start letter/`_`, then alphanumerics/`-`/`_` only) because expressions dereference with dots (`needs.int.runner` would parse as three levels). Use `int-runner` as the ID, `name: int.runner` for display.
- The group is a plain string value matched exactly — dots would parse fine there, but keep names clean anyway.
- Group plus runner `labels:` combine with AND: the runner must be in the group and carry every label. Keep PR labels (`run-uat`) visually distinct from runner labels (`self-hosted`, `linux`) — different universes, one gates whether, the other gates where.
- The group must grant the repo access or jobs queue forever. Runner groups themselves are a paid-tier feature: if org Settings shows no Runner groups section, route labels-only (`runs-on: [self-hosted, uat]`) — same effect on any plan, minus group-level access control that the author gate already replaces.

## Templating across orgs

- `runs-on` cannot see `env` or `secrets` — only `github`, `vars`, and reusable-workflow `inputs`. Anything varying per org lives in Variables or caller inputs, never env.
- `runs-on: { group: ${{ vars.RUNNER_GROUP }} }`, label and associations likewise (`TRUSTED_ASSOCIATIONS` as a JSON list, since personal repos need `OWNER` where orgs need `MEMBER`).
- Org-level Variables with selected-repo scoping when the plan allows; repo-level Variables otherwise (all plans, repo admin each time). Environment-level variables don't reach runner selection.
- Shape: one reusable gated workflow plus thin callers per segment (UAT now, GPU/prod later) — new segment means new caller plus new Variable values, zero workflow surgery. A template repo holds the files plus the setup checklist (group, repo access, variables, label, branch protection, fork approval).
- Onboarding N repos without org features: loop `gh variable set` per repo (org flag variant where available). Variables resolve in the caller repo's context, so one shared file serves every repo.

## Variables: definition and setup commands

Every value differing per org or repo lives in Variables — `runs-on` cannot see `env`. The parameter set:

- `RUNNER_GROUP` — runner group name (`UAT`, later `GPU`, `PROD`).
- `UAT_LABEL` — PR label opting a run in (`run-uat`).
- `TRUSTED_ASSOCIATIONS` — JSON list of trusted PR author associations (`["OWNER"]` on personal repos, `["MEMBER", "COLLABORATOR"]` on orgs).
- `UAT_ENVIRONMENT` — environment name if the approval gate is used (`uat`).

Names take letters, digits, underscores; the `GITHUB_` prefix is reserved. Values are readable to anyone who can view runs, so tokens go in Secrets, never Variables. Repo level wins when both levels define a name — org defaults plus repo exceptions; environment level never reaches runner selection.

```sh
gh variable set RUNNER_GROUP --body "UAT" --repo myorg/myrepo
gh variable set UAT_LABEL --body "run-uat" --repo myorg/myrepo
gh variable set TRUSTED_ASSOCIATIONS --body '["MEMBER"]' --repo myorg/myrepo
gh variable set UAT_ENVIRONMENT --body "uat" --repo myorg/myrepo
```

Org variant (one definition, many repos): swap `--repo myorg/myrepo` for `--org myorg --repos myrepo,otherrepo`. Verify with `gh variable list`, and a canary step echoing `${{ vars.RUNNER_GROUP }}` — safe to print. Bulk onboarding loops the repo list above; `gh variable delete NAME` retires a stale one.
