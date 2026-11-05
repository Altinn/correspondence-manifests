# Correspondence Flux manifests summary

This repo separates workload definitions, environment overlays, and Flux wiring so Correspondence can run on DIS the same way Dialogporten does.

## Structure

- `manifests/`: bases for the API, the migration job, and common config. Per-environment overlays live under `manifests/environments/<env>/`.
- `manifests/environments/<env>/`: env entrypoint that pulls the overlays and sets the image tag.
- `manifests/apps/correspondence/base/`: canonical API manifests used by environment overlays.
- `manifests/environments/<env>/apps/correspondence/`: per-env patches.
- `flux/syncroot/`: `OCIRepository` plus a Kustomization that selects `./environments/<env>` inside the manifests artifact. The `at23` overlay also has the `Receiver` for on-demand reconcile. The platform applies this artifact; it does not create the `product-correspondence` namespace.
- `flux/edge-syncroot/`: edge proxy routing (Envoy Gateway `Backend`, `BackendTLSPolicy`, `HTTPRoute`). The base uses `*.placeholder.invalid` hostnames; each `<env>` overlay patches the public hostname and the backend hostname. `tt02` and `prod` include `components/platform-apim`, which sends traffic to the old APIM through the platform-owned `platform/platform-upstream` `Backend` (and its `BackendTLSPolicy`). `corr-dis-upstream` is still rendered there, unused, so cutover is removing the component. Published as `correspondence/edge-syncroot:<env>`. `yt01` has no edge overlay.

## Flow

1. CI publishes Flux OCI artifacts to ACR (`altinncr.azurecr.io`) on every commit to `main`.
2. `flux/syncroot/` defines an `OCIRepository` pointing to `oci://altinncr.azurecr.io/correspondence/correspondence-sync` with `tag: main`.
3. Flux pulls the OCI artifact and the environment-specific `Kustomization` in `flux/syncroot/<env>` targets `./environments/<env>` inside the artifact.
4. The API image stays on GHCR (`ghcr.io/altinn/altinn-correspondence`) and is pinned in `manifests/environments/<env>/kustomization.yaml`.

Current environments: `at23`, `tt02`, `yt01`, `prod`.

`ASPNETCORE_ENVIRONMENT` selects the `appsettings` file shipped in the image:

| Cluster env | `ASPNETCORE_ENVIRONMENT` | Key Vault |
| --- | --- | --- |
| `at23` | `at23` | `https://altinn-corr-at23-kv.vault.azure.net/` |
| `tt02` | `staging` | `https://altinn-corr-staging-kv.vault.azure.net/` |
| `yt01` | `yt01` | `https://altinn-corr-yt01-kv.vault.azure.net/` |
| `prod` | `production` | `https://altinn-corr-prod-kv.vault.azure.net/` |

`AzureResourceManagerOptions__SubscriptionId` is patched to the subscription that owns that vault (`Altinn-Correspondence-Test` for `at23` and `yt01`, `Altinn-Correspondance-Staging` for `tt02`, `Altinn-Correspondance-Prod` for `prod`). Other non-secret settings stay in the image's `appsettings.<environment>.json`.

Secrets are read from that vault by External Secrets, using the same secret names as the Container Apps template (`.azure/modules/containerApp/main.bicep`): connection strings, Redis, storage, Maskinporten, Slack, Application Insights, and the subscription keys. The `correspondence-secrets` `ApplicationIdentity` is the workload identity for the `SecretStore`. The API and the migration job each have their own `ApplicationIdentity`.

Maskinporten JWK rotation stays disabled (`MaskinportenJwkRotationSettings__Enabled=false`). The current rotator restarts Azure Container Apps, which these clusters do not run.

## On-demand reconcile

The `OCIRepository` polls every 10 minutes, so a published artifact can wait up to 10 minutes before the cluster notices it. An image-tag update asks the cluster to fetch it right away ([RFC 0017](https://github.com/Altinn/altinn-platform/blob/main/rfcs/0017-on-demand-flux-reconcile.md)):

1. `workflow-update-all-image-tags.yml` dispatches `publish-flux-artifacts.yml` with the `environment` whose tags changed.
2. After the app artifact is pushed, the `Trigger Flux reconcile` step calls the shared `Altinn/altinn-platform/actions/flux/trigger-reconcile` action for that environment.
3. The action posts to the webhook endpoint of the environment, `https://ops.<env>.dis.altinn.cloud/flux-webhook`, with the job's GitHub Actions OIDC token. The DIS edge gateway checks the token and the rate limits, then forwards the request to the Flux notification-controller in the DIS core cluster.
4. The `Receiver` `correspondence-manifests` in `flux/syncroot/at23/correspondence-receiver.yaml` verifies the token again and asks Flux to reconcile the `OCIRepository`. The `Kustomization` follows by itself when the source reports a new revision.

The `Receiver` has type `generic-oidc` and shares no secret. It accepts a token only when it is issued to this repository (matched on the immutable numeric repository and organization ids), for `refs/heads/main`, with the webhook endpoint of its environment as audience. A token for one environment is not accepted in another.

The trigger is best effort (`continue-on-error: true`): the request only tells Flux to check its source, and the next poll still picks the artifact up if it fails. A failed trigger shows as a failed step and does not send a Slack alert.

Only `at23` is triggered today. It is the only environment on DIS core, and only its syncroot overlay has the `Receiver`, because a cluster without Flux 2.9 rejects the `generic-oidc` type. Runs without an `environment` (pushes to `main`, manual dispatch without the input) skip the trigger and rely on polling.

## Workflow failure alerts

- `workflow-update-all-image-tags.yml` updates tags from `repository_dispatch` and explicitly dispatches `publish-flux-artifacts.yml` after pushing a change. Its failure notification depends on the entire update job, including input validation and the publish dispatch.
- `publish-flux-artifacts.yml` runs on pushes to `main` and manual dispatch. Its failure notification waits for `publish-syncroot`, `publish-edge-syncroot`, and `publish-app-manifests`, reports all three results, and sends one alert if any job fails.
- Both use `workflow-send-ci-cd-status-slack-message.yml`. The reusable workflow needs `SLACK_BOT_TOKEN` and `SLACK_CHANNEL_ID_FOR_CI_CD_STATUS`. Publishing also needs `DIS_SYNCROOT_AZURE_CLIENT_ID`, `DIS_SYNCROOT_AZURE_TENANT_ID`, and `DIS_SYNCROOT_AZURE_SUBSCRIPTION_ID`.
- Alerts contain operation context and a link to the failing run. They run without checkout or tool installation, JSON-encode dynamic text, and fail visibly on Slack delivery errors. Successful, skipped, and cancelled runs remain silent.

## Routing

Public traffic reaches Correspondence through the edge proxy for `platform.<env>.altinn.cloud` (or `platform.altinn.no` / `platform.tt02.altinn.no`), which owns the `/correspondence` mount point and forwards to `correspondence.<env>.dis-core.*`. The prefix is kept. The API routes are `/correspondence/api/v1/...`.

| Path on the cluster host | What answers |
| --- | --- |
| `/` | `correspondence` |
| `/correspondence/api/v1/...` | `correspondence` |
| `/health` | database health check, open to the same callers as the API |
| `/healthz` | process check, also the kubelet probe path |

Linkerd reserves only `/healthz` for the node network (`manifests/common/base/linkerd-policies.yaml`). Startup, liveness, and readiness all probe `/healthz`, so a database blip does not fail the kubelet probes. `/health` remains available through Traefik.

The edge request timeout is disabled (`0s`). Correspondence accepts large attachment uploads, and the Envoy Gateway default of 15 seconds cuts those off.

## Scaling model

The API autoscales with KEDA (`ScaledObject`, `keda.sh/v1alpha1`). CPU and memory triggers sit at 70% of the container request. Replica bounds match the Container Apps template:

| Env | CPU / memory | min replicas | max replicas |
| --- | --- | --- | --- |
| `at23` | 500m / 1Gi | 1 | 1 |
| `tt02`, `yt01`, `prod` | 2 / 4Gi | 5 | 10 |

Requests equal limits, as Container Apps does. The requests are required: KEDA's cpu/memory triggers read utilisation as a share of the request.

KEDA (AKS managed add-on) must be present in the target cluster. KEDA's admission webhook rejects a `ScaledObject` whose target Deployment is still owned by another HPA, so any pre-existing `HorizontalPodAutoscaler` must be deleted before these manifests reconcile. Use trigger-level `metricType: Utilization`.

## Migration job

`migration-job` is a suspended CronJob, same pattern as Dialogporten's migration job. It is wired to `correspondence-migration-connection-string` and the environment's `ASPNETCORE_ENVIRONMENT`. The image entrypoint is still the API, and the Container Apps pipeline runs migrations as a separate EF bundle, so the CronJob stays suspended. Create a Job from the template only after that image runs the bundle.

Change-maintenance rules are defined in `AGENTS.md`.
