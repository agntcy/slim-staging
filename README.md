# slim-staging

GitOps deployment for [SLIM](https://github.com/agntcy/slim), synced by ArgoCD.

This repo holds the SLIM Helm charts and the ArgoCD configuration that deploys
them. Charts are published as OCI artifacts to
`ghcr.io/agntcy/slim/helm/<chart>` and consumed here by
coordinate, so they can also be installed directly with `helm`.

To point a cluster at this deployment, or to run your own copy of it, see
[docs/deploying.md](docs/deploying.md).

## Layout

| Path | Contents |
| --- | --- |
| `charts/<chart>/` | the Helm charts — `slim`, `slim-control-plane`, `slim-channel-manager`, `slim-spire` |
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

## Releasing a chart

Push a `helm-<chart>-v<version>` tag, e.g. `helm-slim-v2.2.0`. CI lints,
packages and pushes it to `ghcr.io/agntcy/slim/helm/<chart>`.
The version in the tag is what gets published, so keep it in step with the
chart's `Chart.yaml`.

## Upgrading a deployed component

Bump `chart_version` in that app's `config.json`. ArgoCD picks up the new
chart on its next sync; nothing else needs to change.

## Checking images for vulnerabilities

`task deps:vuln:images` renders every chart listed in `Taskfile.yml`, extracts
the container images and scans them with trivy. CI runs the same task.
