# References — every runner component assessed, one verdict each

Read this before proposing a different component: each entry holds the verdict and the reason, with upstream links. The live design stays in README.md; this file is the memory.

## Execution

- **Microsandbox** ([superradcompany/microsandbox](https://github.com/superradcompany/microsandbox), Apache-2.0) — microVMs for untrusted workloads: sub-100 ms boot, OCI images, snapshots and branching, per-sandbox egress allowlists, secrets that never enter the VM. Documented for exactly this use: GitHub Actions jobs in disposable microVMs. Beta — pin versions. Needs KVM; nested flags inside virtualized workers.
- **GitHub self-hosted runners** ([hosting docs](https://docs.github.com/en/actions/hosting-your-own-runners)) — poll GitHub, run ephemeral just-in-time, one job per microVM, untrusted and own-branch pools separated.

## Sibling

- **base-browser** ([mihado/base-browser](https://github.com/mihado/base-browser)) — the shared headless browser pool the runners' verifier gates drive. Browsers verify, runners execute.
