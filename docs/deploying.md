# Deploying SLIM with ArgoCD

How to point a cluster at this repo, and how to run your own copy against a
different cluster.

## Prerequisites

- A Kubernetes cluster, and `kubectl` pointing at it
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/getting_started/) installed
  in the `argocd` namespace
- Enough privilege for ArgoCD to create namespaces and cluster-scoped
  resources — SPIRE installs CRDs and a CSI driver

Nothing needs to be built. The Helm charts are published from the
[slim](https://github.com/agntcy/slim) repo and pulled from
`ghcr.io/agntcy/slim/helm/<chart>`.

## Bootstrap

Two manifests, in order:

```bash
kubectl apply -f projects/slim/dev/slim-dev.yaml
```

```bash
kubectl apply -f projectapps/slim/dev/slim-dev-projectapp.yaml
```

The first creates the `slim-dev` AppProject. The second creates an Application
pointing at `applicationsets/slim/dev`, which renders the ApplicationSet, which
in turn creates one Application per component.

You end up with four Applications, syncing in wave order:

| Wave | App | Namespace |
| --- | --- | --- |
| 0 | `slim-spire` | `slim-dev-spire` |
| 1 | `slim-control-plane` | `slim-dev-control-plane` |
| 2 | `slim` | `slim-dev-slim` |
| 3 | `slim-channel-manager` | `slim-dev-channel-manager` |

The order matters: SPIRE issues the identities the rest authenticate with, and
`slim-channel-manager` dials the data plane at `slim:46357`.

## Watching it come up

```bash
kubectl get applications -n argocd -l env=dev
```

Each should reach `Synced` / `Healthy`. A later wave staying `Progressing`
usually means an earlier one has not gone healthy yet, so start from wave 0.

## What is environment-specific

Two things, and nothing else:

- `clusters/slim/dev/cluster-config.json` — `cluster_address` is the ArgoCD
  destination. `https://kubernetes.default.svc` means the cluster ArgoCD runs
  in; point it elsewhere to deploy to a remote cluster registered with ArgoCD.
- `applications/<app>/dev/values.yaml` — Helm values layered on the chart
  defaults. Empty means "chart defaults".

A `gen.values.yaml` is also read if present, for values produced by a script.
It is optional: `ignoreMissingValueFiles` is set.

## Upgrading a component

Bump `chart_version` in that app's `config.json`. Nothing else changes.

## Running your own copy

Fork the repo, then repoint five references at your fork:

| File | Lines |
| --- | --- |
| `applicationsets/slim/dev/applicationset.yaml` | 40, 54, 79 |
| `projectapps/slim/dev/slim-dev-projectapp.yaml` | 10 |
| `projects/slim/dev/slim-dev.yaml` | 19 (`sourceRepos`) |

Miss the `sourceRepos` entry and ArgoCD refuses the Application with a project
permission error, which reads as a permissions problem rather than a stale URL.

For a different environment, copy the `dev` directories to a new name and
update the `list` generator elements plus the `files` paths in the
ApplicationSet.

## Checking images before you deploy

```bash
task deps:vuln:images
```

Renders every chart, extracts the container images and scans them with trivy.
`task deps:vuln:images:list` prints just the image list.
