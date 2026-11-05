# AGENTS.md

## Purpose

This repository stores Correspondence Flux manifests and wiring for DIS, in the same shape as `Altinn/dialogporten-manifests`.

## Current structure (authoritative)

- `manifests/`: app, job, and common bases plus per-environment overlays in `manifests/environments/<env>/`.
- `manifests/apps/correspondence/base/`: canonical API manifests consumed by each environment overlay.
- `manifests/environments/<env>/apps/correspondence/`: per-env app overlays (patch-only).
- `flux/syncroot/`: bootstrap wiring that selects `./environments/<env>`. The `at23` overlay also has the `Receiver` for on-demand reconcile.
- `flux/edge-syncroot/`: edge proxy routes (`Backend`, `BackendTLSPolicy`, `HTTPRoute`) per environment. The base uses `*.placeholder.invalid` hostnames and every `<env>` overlay must patch all of them; the `publish-edge-syncroot` job fails if any placeholder survives. `tt02` and `prod` include `components/platform-apim`, which routes through the old APIM via the platform-owned `platform/platform-upstream` `Backend`; reference it, do not redefine it. The cross-namespace `backendRef` needs a `ReferenceGrant` in the `platform` namespace.
- `.github/workflows/publish-flux-artifacts.yml`: publishes OCI artifacts for syncroot and app manifests.
- `.github/workflows/workflow-send-ci-cd-status-slack-message.yml`: reusable failure alerts for image-tag updates and artifact publishing.

## Registry model

- Runtime app image: `ghcr.io/altinn/altinn-correspondence`, tag set in `manifests/environments/<env>/kustomization.yaml`. The suspended migration CronJob uses the same image name, so one `images:` entry updates both.
- Flux manifests artifacts: ACR (`altinncr.azurecr.io`), published as:
  - `correspondence/correspondence-sync:main` (app manifests)
  - `correspondence/syncroot:at23` (syncroot; envs listed in the `publish-syncroot` matrix)
  - `correspondence/edge-syncroot:<env>` (edge-syncroot; envs listed in the `publish-edge-syncroot` matrix)
- Flux app `Kustomization` objects apply environment wrappers directly and do not use `postBuild.substituteFrom`. The Key Vault URL is patched in `manifests/environments/<env>/secretstore-vault-url.yaml`. The `${CORRESPONDENCE_KEY_VAULT_URL}` string in the base is a placeholder and must not survive an overlay.

## Workflow failure notifications

- Image-tag update alerts must cover the entire `update-tags` job, including validation, push, and publish dispatch.
- Artifact publishing alerts must depend on `publish-syncroot`, `publish-edge-syncroot`, and `publish-app-manifests` and report all results in one message.
- Keep notifications gated by `failure() && !cancelled()` so successful, skipped, and cancelled runs stay silent.
- Reuse `workflow-send-ci-cd-status-slack-message.yml` with `SLACK_BOT_TOKEN` and `SLACK_CHANNEL_ID_FOR_CI_CD_STATUS`. Keep dynamic payload values JSON-encoded and Slack delivery errors visible as job failures.
- Publishing authenticates to ACR with `DIS_SYNCROOT_AZURE_CLIENT_ID`, `DIS_SYNCROOT_AZURE_TENANT_ID`, and `DIS_SYNCROOT_AZURE_SUBSCRIPTION_ID`.

## On-demand reconcile

- `publish-flux-artifacts.yml` asks the cluster to fetch a new app artifact right after publishing, through the `Receiver` in `flux/syncroot/at23/correspondence-receiver.yaml`. See `docs/summary.md` for the flow.
- Keep the `Trigger Flux reconcile` step after the step that pushes and tags the app artifact, and keep `continue-on-error: true`. Polling is the fallback; a failed trigger must not fail publishing or raise a Slack alert.
- The step runs only for the environment passed as the `environment` dispatch input, and only for environments in its `if:` list. `workflow-update-all-image-tags.yml` must keep passing `-f environment=...`.
- Put a `Receiver` only in the overlay of an environment whose cluster runs Flux 2.9 or later, not in `flux/syncroot/base/`. An older cluster rejects the `generic-oidc` type, and the syncroot then fails to apply.
- The `Receiver` audience must be the webhook endpoint of its environment, `https://ops.<env>.dis.altinn.cloud/flux-webhook`. The step sends the request to that URL, and the action uses it as the token audience.
- To enable another environment, add a `Receiver` to `flux/syncroot/<env>/` and add the environment to the `if:` list of the step.
- The `Receiver` lists only the `OCIRepository`. Do not add the `Kustomization`: it reconciles when the source revision changes, and triggering it directly can run it before the new artifact is fetched.
- The `Receiver` validations pin this repository by numeric id (`repository_id` `1445489683`, `repository_owner_id` `5949420`) and the `main` ref. Do not relax them to names or to the organization alone: GitHub issues OIDC tokens to every repository.

## Change hygiene (required)

When changing structure, environments, Flux source wiring, workflow publish logic, or registry/source strategy, update all relevant guidance in the same PR:

- `README.md`
- `docs/summary.md`
- `AGENTS.md`
- `.github/workflows/publish-flux-artifacts.yml` (if env list/path/source assumptions change)

## Environment additions/removals

If the environment set changes, update all of:

- `flux/syncroot/<env>/kustomization.yaml` — must patch `spec.path` to `./environments/<env>`. The base pins `spec.path` to `at23`, so an overlay that omits the patch silently deploys at23's manifests. CI enforces this in both `pull-request.yml` and the `publish-syncroot` job of `publish-flux-artifacts.yml`.
- validation loops in `.github/workflows/publish-flux-artifacts.yml`
- `README.md`, `docs/summary.md`, and this file

## Scaling model

- The API autoscales via KEDA `ScaledObject` (`keda.sh/v1alpha1`) in the app base, not a plain `HorizontalPodAutoscaler`. Replica counts and container size mirror the Container Apps `scale` and `resources` in `altinn-correspondence` (`.azure/modules/containerApp/main.bicep`).
- `at23` stays on the base: 500m CPU / 1Gi, `minReplicaCount` 1, `maxReplicaCount` 1.
- `tt02`, `yt01`, and `prod` patch to 2 CPU / 4Gi and `minReplicaCount` 5 / `maxReplicaCount` 10, matching the bicep `prodLikeEnvironment` set (`staging`, `yt01`, `production`).
- Do not drop container `resources.requests`. KEDA's cpu/memory triggers are a share of the request; without it the autoscaler reports `<unknown>` and never scales.
- The `ScaledObject` CRD is a hard dependency — a cluster without the KEDA add-on will fail to reconcile these manifests.

## Routing model

- The edge proxy owns the public `/correspondence` mount point. It forwards that prefix unchanged. Correspondence controllers are declared at `/correspondence/api/v1/...` and the process does not call `UsePathBase`, so stripping the prefix would miss every controller.
- On `correspondence.<env>.dis-core.*`, Traefik matches `/` and does not rewrite. `/health` and `/healthz` are served on that host without the `/correspondence` prefix.
- `tt02` and `prod` send the edge route to `platform/platform-upstream` and rewrite the hostname to the public platform host, same as Dialogporten. APIM must keep the `/correspondence` prefix.
- The Linkerd catch-all route in `manifests/common/base/linkerd-policies.yaml` admits every path except exactly `/healthz`. Kubelet probes use `/healthz`. `/health` stays on the catch-all so availability checks through Traefik can read the database check. A new probe path must be added to the `correspondence-health` route, or the kubelet gets 403.

## Validation baseline

Run before commit when relevant:

- `actionlint` for workflow changes
- `kustomize build manifests/environments/at23`
- `kustomize build manifests/environments/tt02`
- `kustomize build manifests/environments/yt01`
- `kustomize build manifests/environments/prod`
- `kustomize build flux/syncroot/at23` (and `tt02`, `yt01`, `prod`)
- `kustomize build flux/edge-syncroot/at23` (and `tt02`, `prod`)
