# base-runner

Ephemeral GitHub runners, one job per microVM. Sibling to [base-browser](https://github.com/mihado/base-browser): browsers verify the work, runners execute it. Status: **concept.** Decisions below are recorded; nothing here is deployed yet. Open questions are marked `TBD`.

## Design: disposable runners, sandboxed jobs

Each CI job runs in its own microVM ([microsandbox](https://github.com/superradcompany/microsandbox)), spawned just-in-time and destroyed after. The runner host holds no job state, no credentials, and no trust between jobs: a compromised dependency, a poisoned test fixture, or a malicious build script dies with its VM. Runners poll GitHub — the direction is already correct, and nothing inbound is ever opened for them.

Two pools, never mixed: one for untrusted work (fork PRs, first-time contributors), one for own branches and automation. Fork code never touches iron that holds secrets or a privileged network position — GitHub-hosted runners take that side entirely, or a quarantined pool with no secrets and no internal routes. Own branches still run sandboxed per job: they install the world's arbitrary code too. The host also serves dispatched agent work and the browser pool; runners share the host, jobs share nothing.

Fork-PR routing is by author association over one reusable workflow: same-repo branches run local, everything else runs cloud. Both sides required in branch protection (a skipped required check counts as satisfied). `pull_request_target` stays out — base-branch workflow plus secrets plus one careless checkout reopens everything.

## Requirements

MicroVMs need KVM on Linux. Inside an already-virtualized worker that means nested virtualization flags (host CPU type plus nesting enabled) — verify before sizing anything. The runner host itself is cattle: rebuild from image, never mutate, never store anything a fresh clone cannot reproduce. Site-specific credentials (registration tokens, tunnel identities, network placement) stay out of this repo entirely.

## Open questions (TBD)

- Runner controller: actions-runner-controller scale sets vs a small just-in-time provisioner.
- Pool sizing and the microVM image set per pool.
- Secrets home for runner registration (short-lived JIT tokens preferred).
