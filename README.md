# billings-gitops-poc

Throwaway proof-of-concept for the **Argo CD GitOps CD layer** evaluated during the
GitLab → GitHub migration (org `gobots-ai`). This is the *desired-state* (manifest) repo
that Argo CD pulls from — the downstream half of the architecture in
`CUTOVER-RUNBOOK-billings.md` / `cicd-platform-comparison.html`.

## What it proves

- **One `ApplicationSet` → N Applications.** `appset.yaml` uses a Git directory generator
  over `apps/*`. Each folder becomes its own Argo CD `Application` with **zero per-repo
  YAML**. This is the mechanism that scales to ~278 repos.
- **Helm-native.** Each `apps/<service>/` is a normal Helm chart with per-app `values.yaml`
  — the real billing charts plug in unchanged.
- **Pull-based, self-healing.** `syncPolicy.automated.selfHeal: true` means a manual
  `kubectl` change to the cluster is reverted to match Git. No kubeconfig in CI.

## Layout

```
appset.yaml                       ApplicationSet (applied to the argocd namespace)
apps/
  billing-service/                Helm chart (stand-in workload)
  billing-stripe-publisher/
  billing-engine-service/
  billing-panel/
  closing-script/
```

> Stand-in workloads use `traefik/whoami` because the real images live in private GAR and
> the real deploy target is AKS — neither is reachable from the local PoC cluster. The
> GitOps **mechanics** (generation, sync, drift/self-heal) are identical regardless of image.

## Real-world deltas (not in this PoC)

- Manifest repo lives in `gobots-ai`, private, with Argo CD repo credentials.
- Image tag bumped by GitHub Actions (or Argo CD Image Updater) after the CI build pushes to GAR.
- Secrets via **External Secrets Operator** (replacing GitLab `K8S_SECRET_*`) — neither
  Argo CD nor Flux solves this natively.
- Argo CD runs inside AKS; destination is the AKS cluster, not `kubernetes.default.svc`.
