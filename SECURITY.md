# Security

fleet is the Flux GitOps repo for the `nsc2108` k3s cluster. Flux applies `clusters/nsc2108/`
from `main` every minute, so **a merge to `main` is a deploy**, and this repository is public.
This file lists what is worth attacking and where the defenses live. It is for security reviews;
how to work here is in `CLAUDE.md`.

## Reporting a vulnerability

Do not open a public issue. Contact the maintainer privately through GitHub. Include the file
and commit, what an attacker would need, and what they gain on the cluster.

## Who can deploy

| Path | What it reaches | Defense |
| --- | --- | --- |
| Merging to `main` | Anything Flux applies: every namespace, at Flux's cluster-admin rights | Only the owner can merge. `main` requires a PR and the `kustomize` check. |
| The `flux-system` Secret | Read access to this repo over HTTPS (`gotk-sync.yaml`) | Read-only; the repo is public anyway. It grants no push. |
| The exemplar release pipeline | The whole `exemplar` namespace: Flux follows the `latest` tag of `oci://ghcr.io/koblas/exemplar-manifests` (`clusters/nsc2108/apps/exemplar.yaml`) | Whoever can publish an exemplar release, or push to that GHCR package, deploys to `nsc2108`. The artifact pins both images by digest, but the artifact itself is followed by a moving tag. To hold a version, set `tag:` or `semver:` there. |
| Helm charts | `radar` and `buildkite` namespaces, plus each chart's RBAC | Pinned by chart version (`apps/*/helmchart.yaml`), not by digest: a re-published version at the same number changes the cluster on the next helm-controller run. |

## CI on the cluster

Buildkite's agent-stack-k8s (`apps/buildkite/helmchart.yaml`) runs jobs from the `kubernetes`
queue as Kubernetes Jobs in the `buildkite` namespace, at most four at once. Its Role reaches
nothing outside that namespace.

- **Agent token** (Secret `buildkite-agent-token`): lets anyone holding it take jobs on the
  queue. Made by hand, never committed.
- **Git SSH key** (Secret `git-ssh-credentials`): mounted into every job's checkout container.
  Any pipeline step that runs on this queue can read it, so it should be a read-only deploy key.
- **Untrusted code**: a job runs whatever the pipeline's steps say. A Buildkite pipeline that
  builds pull requests from forks runs outside code on this node with the key above. Keep fork
  builds off for every pipeline that targets `queue=kubernetes`. That is a Buildkite setting, not
  in this repo.
- **Git mirrors** (PVC `buildkite-git-mirrors`): shared by every job. A job that can write to
  it could poison another job's checkout. The chart mounts it only into the checkout container.

## Secrets

No secret value is in this repository. Each one is created by hand with `kubectl`, and the
manifest that needs it says how. In a public repo a committed secret is leaked the moment it is
pushed, even if the commit is reverted: rotate it, don't just delete it.

| Secret | Namespace | Holds |
| --- | --- | --- |
| `flux-system` | `flux-system` | Flux's read credential for this repo |
| `buildkite-agent-token` | `buildkite` | Buildkite cluster agent token |
| `git-ssh-credentials` | `buildkite` | SSH key for private-repo checkouts |
| `exemplar-cloudflared` | `exemplar` | Cloudflare Tunnel credentials |
| Exemplar's own Secrets | `exemplar` | Filled by exemplar's setup page; Flux never prunes them (see `exemplar.yaml`) |

## Exposure

The node has no inbound route. Exemplar is reached only through its Cloudflare Tunnel.
Radar has no auth and a ClusterIP Service only. Reach it with `kubectl port-forward`, and its MCP
endpoint is read-only (`--mcp-read-only`).
