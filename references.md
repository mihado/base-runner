# References — every runner component assessed, one verdict each

Read this before proposing a different component: each entry holds the verdict and the reason, with upstream links. The live design stays in README.md; this file is the memory.

Scope: the **development execution plane** — CI jobs and dev agent work on our own iron. Sandboxing inside a product's own runtime is a separate design, recorded at the end so the two are not conflated.

## Execution

- **Microsandbox** ([superradcompany/microsandbox](https://github.com/superradcompany/microsandbox), Apache-2.0) — microVMs for untrusted workloads: sub-100 ms boot, OCI images, snapshots and branching, per-sandbox egress allowlists, secrets that never enter the VM. Documented for exactly this use: GitHub Actions jobs in disposable microVMs. The official recipe is the JIT flow — a single-use config from `gh api .../generate-jitconfig` piped into the guest, no inbound port, no host directory or Docker socket mounted, the runner long-polling outbound. Beta — pin versions. Needs KVM; nested flags inside virtualized workers.
- **GitHub self-hosted runners** ([hosting docs](https://docs.github.com/en/actions/hosting-your-own-runners)) — poll GitHub, run ephemeral just-in-time, one job per microVM, untrusted and own-branch pools separated.
- **E2B** ([e2b-dev/E2B](https://github.com/e2b-dev/E2B), Apache-2.0 core; infrastructure as Terraform) — Firecracker microVMs, the self-hosted alternative if microsandbox disappoints. Verdict: not adopted. Self-hosting is cloud-shaped (Terraform on AWS/GCP), advanced features such as the egress proxy are cloud/BYOC-gated, and there is no GPU. Revisit only if the chosen substrate stalls.
- **Firecracker / Kata Containers / Cloud Hypervisor** — DIY microVM substrates. Verdict: rejected. More operations than the isolation is worth at this size; microsandbox already wraps a microVM runtime with OCI, snapshot, and egress ergonomics.

## Controllers (what spawns a job's VM)

- **Thin JIT provisioner** — the default shape: a small service that watches for queued jobs, mints a single-use JIT config, boots the microVM, pipes the config in, and tears the VM down. The microsandbox recipe above is the reference implementation; nothing to adopt.
- **myshoes + shoes-microsandbox** ([whywaita/myshoes](https://github.com/whywaita/myshoes), [whywaita/shoes-microsandbox](https://github.com/whywaita/shoes-microsandbox), Apache-2.0) — off-the-shelf scheduler plus a provider that boots a microsandbox VM per job and removes it after. Verdict: candidate, not adopted. The provider returns a VM **IP**, which implies an inbound setup path (SSH) the JIT flow does not need; it also adds a scheduler service, a database, and a young community plugin. Adopt only if the in-house provisioner proves to be the wrong tax.
- **actions-runner-controller** — Kubernetes scale sets over pods. Verdict: rejected on this substrate; it manages containers, not microVMs.

## Managed pools (fallbacks and comparisons, never the iron)

- **Modal** (gVisor, hosted only), **Vercel Sandbox** (Firecracker, Vercel-only), **Northflank** (Kata/Cloud Hypervisor, BYOC), **Cloudflare Sandbox** (VM-isolated containers on Workers, edge-only). Verdict: useful as untrusted-fallback capacity and as a benchmark against the local pool; none can run these jobs on our own iron. Cloudflare's egress credential injection, snapshot-on-sleep, preview URLs, PTYs, and persistent interpreter contexts are the feature bar to match, not a platform to adopt.
- **Daytona** — Verdict: rejected. Moved closed-source in June 2026 and the public repository is no longer maintained.

## Not this repo's decision

Product-runtime sandboxing — per-tenant agent execution, injection containment, in-app authorization bypass — is a separate track with its own design. Two entries that keep appearing here belong to it, not to base-runner:

- **agentOS** ([rivet-dev/agentos](https://github.com/rivet-dev/agentos), Apache-2.0, beta) — agents run in-process in a software VM (V8 isolates plus WebAssembly, no KVM) and escalate browsers, native binaries, and compilation to external sandboxes. A candidate for product agent execution; never for CI jobs.
- **Rivet** ([rivet-dev/rivet](https://github.com/rivet-dev/rivet), Apache-2.0) — durable actor orchestration, self-hosted as a single binary or container with Postgres. Product-side infrastructure; a runner provisioner is not an actor runtime.

Note for that track: at this stage the product risk is prompt injection and in-app authorization, which a sandbox does not fix — sandboxing bounds blast radius, it does not decide what an actor is allowed to do.

## Sibling

- **base-browser** ([mihado/base-browser](https://github.com/mihado/base-browser)) — the browser space: verifier gates, browsing agents, and R&D in its own VM and network tier, with public egress and edge-mediated internal access. Runners execute, browsers verify — and explore. A job's own preview is verified by an in-sandbox browser, not the shared pool.
