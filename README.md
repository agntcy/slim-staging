# slim-staging

GitOps deployment for [SLIM](https://github.com/agntcy/slim), synced by ArgoCD.

This repo holds deployment configuration only. It does not build or publish
anything: the Helm charts are published from the `slim` repo as OCI artifacts
and consumed here by coordinate.

## Layout

| Path | Contents |
| --- | --- |
| `applications/<app>/<env>/config.json` | chart coordinates — `chart_repo`, `chart_name`, `chart_version` |
| `applications/<app>/<env>/values.yaml` | per-environment Helm values |
| `applicationsets/slim/<env>/` | the ApplicationSet that renders one ArgoCD Application per app |
| `clusters/slim/<env>/` | cluster address and name |
| `projects/slim/<env>/` | the ArgoCD AppProject |
| `projectapps/slim/<env>/` | the Application that bootstraps the ApplicationSet |

## Components and ordering

Applications sync in waves, because the stack has a startup order:

| Wave | App | Why |
| --- | --- | --- |
| 0 | `slim-spire` | issues the SPIFFE identities the rest authenticate with |
| 1 | `slim-control-plane` | manages routes for the data plane |
| 2 | `slim` | the data plane node |
| 3 | `slim-channel-manager` | dials the data plane at `slim:46357` |

## Upgrading a component

Bump `chart_version` in that app's `config.json`. ArgoCD picks up the new
chart on its next sync; nothing else needs to change.

## Checking images for vulnerabilities

`task deps:vuln:images` renders every chart listed in `Taskfile.yml`, extracts
the container images and scans them with trivy. CI runs the same task.
