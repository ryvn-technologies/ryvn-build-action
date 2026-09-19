# ryvn-build-action

GitHub Action for Building and Publishing to Ryvn Registry

## Overview

This GitHub Action builds and pushes Docker images or Helm charts to the Ryvn Registry. It supports:

- Docker images built from Dockerfiles
- Docker images built using Nixpacks (without Dockerfile)
- Automatic Nixpacks detection from service configuration
- Helm charts

## Usage

```yaml
- name: Build and Push to Ryvn
  uses: ryvn-technologies/ryvn-build-action@v1
  with:
    service_name: my-service
    version: 1.0.0
    ryvn_client_id: ${{ secrets.RYVN_CLIENT_ID }}
    ryvn_client_secret: ${{ secrets.RYVN_CLIENT_SECRET }}
```

## Authentication

The action talks to the Ryvn API through the Ryvn CLI, which picks a credential in this order:

1. `RYVN_ACCESS_TOKEN` in the job environment (a pre-issued Ryvn token, used as-is).
2. **Keyless CI auth**: when the job has `permissions: id-token: write`, the CLI exchanges the job's GitHub OIDC token for a short-lived Ryvn credential scoped to the services this repository owns. No secrets are needed.
3. `ryvn_client_id` / `ryvn_client_secret` (static credentials). When the OIDC environment is present these are used **only** if the hub reports that keyless auth is not enabled for the organization (or that more than one organization trusts the repository and `ryvn_org_id` was not given); any other exchange failure is fatal and never falls back to the static secret.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - name: Build and Push to Ryvn (keyless)
        uses: ryvn-technologies/ryvn-build-action@v2
        with:
          service_name: my-service
          version: 1.0.0
```

Set `ryvn_org_id` when more than one Ryvn organization trusts the same GitHub repository. Static credentials keep working unchanged for CI systems without OIDC and during the rollout.

Keyless auth needs a Ryvn CLI with the OIDC credential source; there is no separate login step — every `ryvn` invocation exchanges the job's GitHub OIDC token on its own (cached within the process). Both the action and the reusable workflow (`.github/workflows/release.yml`) install the CLI and accept a `ryvn_cli_version` input (e.g. `v1.278.0`) to pin a release; when unset the action installs `v1.278.0`, the oldest release carrying the publishing commands (`create/delete registry-config`, `package chart`, `push chart`, `create release-file`, `describe image`), the managed Helm destination fix, and the image digest in `describe image -o artifact`. An older explicit pin fails at the first missing command with the CLI's own `unknown command` error. A CLI that predates keyless auth ignores `id-token: write` and uses the static credentials if set; otherwise its first `ryvn` call fails on missing credentials.

The reusable workflow inherits the `GITHUB_TOKEN` permissions granted by the calling job, so the caller decides the auth mode: `contents: write` plus `id-token: write` for keyless auth, or `contents: write` alone to force the static `RYVN_CLIENT_ID`/`RYVN_CLIENT_SECRET` credentials (e.g. for services whose spec has no GitHub `repo` locator, such as public terraform module `source`s).

The reusable workflow also accepts an optional `tag_prefix` input (e.g. `gcp-gke@`) that overrides the tag prefix derived from the service definition — needed for terraform services using a public module `source` in a repository that releases several services.

## Inputs

| Name                 | Description                                               | Required | Default |
| -------------------- | --------------------------------------------------------- | -------- | ------- |
| `service_name`       | Name of the service                                       | Yes      |         |
| `version`            | Semantic version to tag the image/chart with              | Yes      |         |
| `build_only`         | Build only, don't push to registry                        | No       | `false` |
| `ryvn_client_id`     | Ryvn Client ID (static credentials; see Authentication)   | No       |         |
| `ryvn_client_secret` | Ryvn Client Secret (static credentials; see Authentication)| No       |         |
| `ryvn_org_id`        | Ryvn organization ID, for repositories trusted by more than one organization | No |   |
| `ryvn_cli_version`   | Ryvn CLI release to install (must be `v1.278.0` or newer) | No | `v1.278.0` |
| `build_args`         | Build arguments to pass to the Docker build               | No       |         |
| `use_nixpacks`       | Use Nixpacks to build Docker images instead of Dockerfile | No       | `false` |
| `nixpacks_pkgs`      | Additional Nix packages to install in the environment     | No       | `""`    |
| `nixpacks_apt`       | Additional Apt packages to install in the environment     | No       | `""`    |
| `nixpacks_cache`     | Use the Nixpacks build cache                              | No       | `true`  |

## Outputs

| Name              | Description                                                                 |
| ----------------- | --------------------------------------------------------------------------- |
| `build_artifacts` | JSON array of the artifacts this run published, in the shape `ryvn create release --artifacts-file` accepts |
| `release_file` | Generated release artifact inventory for Helm services; empty for container services and build-only runs. Pass it to `ryvn create release -f` |

For a container service the array holds one entry with the published image and its digest: the digest the build step reported for the push when available, otherwise the one the registry resolves for the tag (the image index digest for a multi-platform build). A pushed image that does not resolve fails the action, because the release would otherwise have no immutable identity (`ryvn describe image`; carrying the digest into the artifact requires a CLI release that includes it — see the minimum version below). The reusable workflow creates container releases with `--artifacts-file <build_artifacts> --inventory`, producing an immutable release with the digest-pinned image as its primary artifact. Deployments still pull by tag; the digest is recorded, not enforced at deploy time.

```json
[{"name": "api", "image": {"repository": "…/acme/api", "tag": "1.2.3", "digest": "sha256:…", "exposedPorts": ["8080/tcp"], "exposedPortsSource": "image"}}]
```

For a Helm chart service it holds the pushed chart and its digest:

```json
[{"name": "my-chart", "type": "helm-chart", "helmChart": {"repoUrl": "oci://…/acme", "chartName": "my-chart", "version": "1.2.3", "digest": "sha256:…"}}]
```

With `build_only: true` nothing is published and nothing is inspected: the output is `[]` for every service type.

## Examples

### Basic usage with Docker

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build and Push Docker Image
        uses: ryvn-technologies/ryvn-build-action@v1
        with:
          service_name: backend-api
          version: 1.2.3
          ryvn_client_id: ${{ secrets.RYVN_CLIENT_ID }}
          ryvn_client_secret: ${{ secrets.RYVN_CLIENT_SECRET }}
```

### Using Nixpacks (no Dockerfile needed)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build with Nixpacks
        uses: ryvn-technologies/ryvn-build-action@v1
        with:
          service_name: frontend-app
          version: 2.0.0
          use_nixpacks: true
          nixpacks_pkgs: "nodejs_20"
          ryvn_client_id: ${{ secrets.RYVN_CLIENT_ID }}
          ryvn_client_secret: ${{ secrets.RYVN_CLIENT_SECRET }}
```

### Automatic Nixpacks Detection

When your service is configured with `buildpack: "nixpack"` in the Ryvn service definition, the action will automatically use Nixpacks without needing to set `use_nixpacks: true`:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build with Auto-detected Nixpacks
        uses: ryvn-technologies/ryvn-build-action@v1
        with:
          service_name: nixpack-test # Service configured with buildpack: "nixpack"
          version: 1.0.0
          ryvn_client_id: ${{ secrets.RYVN_CLIENT_ID }}
          ryvn_client_secret: ${{ secrets.RYVN_CLIENT_SECRET }}
```

The action will automatically use the build command from your service definition (e.g., `npm install -g pnpm@9.0.2 && cd apps/cronjob && pnpm i && pnpm run build`).

### Building a Helm Chart

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build and Push Helm Chart
        uses: ryvn-technologies/ryvn-build-action@v1
        with:
          service_name: my-chart
          version: 1.0.0
          ryvn_client_id: ${{ secrets.RYVN_CLIENT_ID }}
          ryvn_client_secret: ${{ secrets.RYVN_CLIENT_SECRET }}
```

An existing `<service>-release.ryvn.yaml` beside the build can contribute dependencies, migrations, and labels to the generated inventory. It must not define `artifacts`: the build generates them, and the command fails if it does.

## How It Works

The action is orchestration only: runner setup, Buildx, `docker/build-push-action`, and step outputs. Everything that needs to know about Ryvn registries, Helm charts, or release artifacts is a Ryvn CLI command, so the same flow works from any CI system.

1. Installs the Ryvn CLI (`ryvn_cli_version`, default `v1.278.0`). Creates a private per-invocation workspace under `runner.temp` for every transient file (service and registry metadata, registry config, chart package, artifacts), so several invocations in one job never share state and nothing is written into the checkout; the `always()` cleanup step removes it.
2. `ryvn get service <name> -o json` for the service definition; ordinary fields (`type`, `build.*`, `image`) become step outputs and build inputs. The `ryvn_api_url`/`ryvn_auth_url`/`ryvn_org_id`/`ryvn_project_id` inputs apply to the action's own CLI steps only (falling back to the job's `RYVN_*` environment); they are not exported to later steps.
3. Detects the build method:
   - If `buildpack: "nixpack"` is set in service definition, uses Nixpacks with the service's build command
   - If `use_nixpacks: true` is provided as input, uses Nixpacks
   - Otherwise uses standard Docker build with Dockerfile
4. Unless `build_only`, reads the bound registry (`ryvn get registry <definition.registry> -o json`) and routes on its `definition.type` — see Registries below.
5. Container services: `docker/build-push-action` builds (and pushes) the image; unless `build_only`, `ryvn describe image <ref> --service <name> -o artifact` resolves the pushed tag in the registry and records its digest and exposed ports. When the build step reported a push digest (Buildx does, Nixpacks does not) it is passed as `--digest`, and the CLI inspects that exact content (`repo@digest`) so the artifact describes what this run pushed even if the tag has since been moved by another push; the current tag is not compared against it. Without a push digest the tag is resolved as it is at that moment. The image must resolve; only exposed-port metadata is best effort. The reusable workflow passes `--artifacts-file <build_artifacts> --inventory` when creating the release, so the digest-pinned image is the release's primary artifact.
6. Helm services: `ryvn package chart <chartPath> --version <v> -o json` packages locally; `ryvn push chart <pkg> --service <name> -o artifact` resolves the service's chart destination and publishes; `ryvn create release-file` renders the packaged chart, discovers every image reference, resolves all image digests, and writes the release artifact inventory.
7. A service produces at most one artifact array (`describe image` or `push chart`; `[]` for a `build_only` run), passed through as the `build_artifacts` output. Helm services additionally produce the `release_file` output for `ryvn create release -f`.

Buildable service types are `web-server.v1`, `job.v1` and `helm-chart.v1`.

Helm charts are packaged from `definition.build.chartPath`. That field has always been required for a source-built `helm-chart.v1` (the API's `HelmChartBuildSettings` requires it and the action has always failed without it); the reusable release workflow checks it up front instead of failing later in the build step. No new Helm or action parameter is introduced.

### The same flow outside GitHub Actions

```bash
ryvn create registry-config "$RUNNER_TEMP/ryvn-auth" --service api -o env > auth.env && . ./auth.env   # GAR; ECR keeps aws ecr get-login-password
docker buildx build --push -t "$IMAGE:$VERSION" .
# container services
ryvn describe image "$IMAGE:$VERSION" --service api -o artifact > image.json
ryvn create release api "$VERSION" --channel "$CHANNEL" --artifacts-file image.json --inventory
# helm-chart.v1 services
ryvn package chart ./chart --version "$VERSION" -o json
ryvn push chart "api-$VERSION.tgz" --service api -o artifact > chart.json
ryvn create release-file api "$VERSION" --chart "api-$VERSION.tgz" --artifacts-file chart.json --output-file release.yaml
ryvn create release api "$VERSION" --channel "$CHANNEL" -f release.yaml
ryvn delete registry-config "$RUNNER_TEMP/ryvn-auth"
```

When `release_file` is used, a repo-local `<service>-release.ryvn.yaml` is not auto-loaded; the `-f` file replaces it.

## Registries

The bound registry's `definition.type` decides how the action authenticates; it is never derived from the organization name or from the image URL.

| Registry type | Login |
|---|---|
| `elasticContainerRegistry` (Ryvn-managed ECR) | Assumes the org's GitHub Actions role and uses `amazon-ecr-login`, as before. `ryvn push chart --registry-config $DOCKER_CONFIG/config.json` reads that same Docker login, so no separate `helm registry login` is needed and a stale `HELM_REGISTRY_CONFIG` in the job cannot shadow it |
| `googleArtifactRegistry` (Ryvn-managed Google Artifact Registry) | `ryvn create registry-config <dir> --service <name>` writes a disposable Docker/Helm config holding a short-lived token; the action writes it inside its own per-invocation temp workspace, exports its `DOCKER_CONFIG`/`HELM_REGISTRY_CONFIG` for the build, push, and inspect steps, and an `always()` cleanup step restores the previous variables and removes the whole workspace (credentials plus the `.token_seed*` files Docker drops into `$DOCKER_CONFIG` during a build); a failed removal fails the step. The caller's own Docker/Helm/buildx directories are never touched. `ryvn delete registry-config` is for configs placed in a shared directory (see the custom-CI example) |

Any other registry type (including bring-your-own registries) fails with an explicit error rather than publishing with a different credential. If Ryvn refuses to issue credentials (missing `service.build`/`registry.publish`, publisher access not provisioned yet), the action fails; it never falls back to AWS or anonymous pushes. 
**Authorization:** reading the bound registry (`ryvn get registry`) needs `registry.read` for every published build, ECR keyless CI included, not only Google Artifact Registry. The orchestrator reconciles that binding onto the keyless CI identity for every registry a repository's services publish to (`ryvn-orchestrator` >= 0.896.1), so no manual grant is needed.

Google Artifact Registry notes:

- Credentials live for one hour and are issued immediately before the build-and-push step. A single build whose build-and-push phase takes longer than the credential lifetime will still fail at push time; split long builds or use `build_only` plus a separate publish job.
- `create registry-config` seeds the disposable Docker config from the caller's (buildx builders, other registries and their `credsStore`/`credHelpers` stay as they are), pins only the GAR host to file-backed auth so the token is never handed to a credential helper that would keep it after cleanup, and preserves `DOCKER_CERT_PATH` so Docker TLS still finds the caller's certificates. The buildx builder is created before the switch to the isolated config, so `docker/setup-buildx-action`'s post-job cleanup still finds it. The password never reaches action logs or outputs.
- `build_only: true` never requests publish credentials, logs in, reads the registry, or pushes; it only needs read access to the service, and emits no artifacts.
- The CLI steps read `RYVN_API_URL`/`RYVN_AUTH_URL`/`RYVN_PROJECT_ID`/`RYVN_ORG_ID` the same way the initial `ryvn get service` does: a job-level `env` value is inherited and the corresponding action input overrides it only when set, so credentials are always issued by the hub the service was read from.

## Nixpacks Configuration

The action supports two ways to use Nixpacks:

1. **Manual**: Set `use_nixpacks: true` in your workflow
2. **Automatic**: Configure your service with `buildpack: "nixpack"` in the Ryvn service definition

When using automatic detection, the action will:

- Extract the `build.command` from your service definition
- Use it as the `NIXPACKS_BUILD_CMD` environment variable
- Build with Nixpacks using your service's specific build command

## License

MIT
