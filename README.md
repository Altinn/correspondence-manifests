# Correspondence Flux manifests

This repository contains the Flux wiring and workload manifests for Correspondence on DIS. Workloads are packaged per environment as OCI artifacts and pulled by Flux.

The layout follows [dialogporten-manifests](https://github.com/Altinn/dialogporten-manifests). Correspondence runs as one API, `ghcr.io/altinn/altinn-correspondence`, plus a suspended database migration CronJob. Hangfire workers run inside the API process.

## Layout

- `manifests/`: shared bases (`apps/`, `jobs/`, `common/`) plus per-environment overlays collected under `manifests/environments/<env>/`.
- `manifests/apps/correspondence/base/`: canonical API manifests (Deployment, Service, Traefik `HTTPRoute`, KEDA `ScaledObject`, `ApplicationIdentity`).
- `manifests/environments/<env>/apps/correspondence/`: per-env patches for the ASP.NET environment, Azure subscription, hostname, and, where the Container Apps template is larger, CPU, memory, and replica counts.
- `manifests/environments/<env>/kustomization.yaml`: env entrypoint. It pulls the app and job overlays and sets the image tag.
- `flux/syncroot/`: bootstrap wiring (`OCIRepository` and a Kustomization that targets `./environments/<env>`). The `at23` overlay also has the `Receiver` for on-demand reconcile.
- `flux/edge-syncroot/`: edge proxy routing for the public `/correspondence` mount point (Envoy Gateway `Backend`, `BackendTLSPolicy`, and `HTTPRoute`). The base uses placeholder hostnames; each `<env>` overlay sets the public hostname and the `correspondence.<env>.dis-core.*` backend hostname. `tt02` and `prod` still route through the old APIM via the `components/platform-apim` component, which points the `HTTPRoute` at the platform-owned `platform/platform-upstream` `Backend` instead of `corr-dis-upstream`.

Current environments: `at23`, `tt02`, `yt01`, `prod`.

## OCI flow (high level)

1. CI publishes Flux OCI artifacts to ACR (`altinncr.azurecr.io`): syncroot from `flux/syncroot`, edge-syncroot from `flux/edge-syncroot` (`correspondence/edge-syncroot:<env>`), and app manifests from `manifests/` (`correspondence/correspondence-sync:main`).
2. `flux/syncroot/` defines an `OCIRepository` pointing to `oci://altinncr.azurecr.io/correspondence/correspondence-sync` with `tag: main`.
3. Flux pulls that OCI artifact, and the environment-specific `Kustomization` in `flux/syncroot/<env>` targets the `./environments/<env>` path within the artifact.
4. After an image-tag update, CI asks the environment's cluster to pull the new artifact right away instead of at the next 10-minute poll. See [On-demand reconcile](docs/summary.md#on-demand-reconcile).

Application runtime images remain GHCR-hosted and are pinned by tags in `manifests/environments/<env>/kustomization.yaml`.

## Failure notifications

`Update all image tags` and `Publish Flux artifacts` send one Slack alert per failed workflow run through `.github/workflows/workflow-send-ci-cd-status-slack-message.yml`. Alerts include the repository, workflow, job results, a link to the run, and the requested environment/image tag or published ref/commit. Image-update alerts also cover input validation, checkout, tool installation, push, and publish-dispatch failures. Publishing alerts cover any artifact job, including validation failures. Successful, skipped, and cancelled runs do not send alerts.

These GitHub Actions secrets are the same ones Dialogporten uses:

- `DIS_SYNCROOT_AZURE_CLIENT_ID`, `DIS_SYNCROOT_AZURE_TENANT_ID`, `DIS_SYNCROOT_AZURE_SUBSCRIPTION_ID`: workload identity for pushing OCI artifacts to `altinncr.azurecr.io`.
- `SLACK_BOT_TOKEN`: Slack bot token with permission to post to the CI/CD status channel.
- `SLACK_CHANNEL_ID_FOR_CI_CD_STATUS`: ID of that channel; the bot must have access to it.

Slack delivery errors fail the notification job so an undelivered alert is visible in Actions.

See `docs/summary.md` for more detail. Agent/maintenance rules live in `AGENTS.md`.
