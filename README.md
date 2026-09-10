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

Keyless auth needs a Ryvn CLI with the OIDC credential source; there is no separate login step — every `ryvn` invocation exchanges the job's OIDC token on its own (cached within the process). Both the action and the reusable workflow (`.github/workflows/release.yml`) install the CLI and accept a `ryvn_cli_version` input (e.g. `v1.190.0`) to pin a release when the installer's default is older. A CLI that predates keyless auth ignores `id-token: write` and uses the static credentials if set; otherwise its first `ryvn` call fails on missing credentials.

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
| `ryvn_cli_version`   | Ryvn CLI release to install (e.g. `v1.190.0`); default is the installer's pin | No |   |
| `build_args`         | Build arguments to pass to the Docker build               | No       |         |
| `use_nixpacks`       | Use Nixpacks to build Docker images instead of Dockerfile | No       | `false` |
| `nixpacks_pkgs`      | Additional Nix packages to install in the environment     | No       | `""`    |
| `nixpacks_apt`       | Additional Apt packages to install in the environment     | No       | `""`    |
| `nixpacks_cache`     | Use the Nixpacks build cache                              | No       | `true`  |

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

## How It Works

This action:

1. Installs the Ryvn CLI
2. Retrieves service configuration from Ryvn API
3. Automatically detects build method:
   - If `buildpack: "nixpack"` is set in service definition, uses Nixpacks with the service's build command
   - If `use_nixpacks: true` is provided as input, uses Nixpacks
   - Otherwise uses standard Docker build with Dockerfile
4. Authenticates with AWS ECR
5. Builds the Docker image or packages the Helm chart
6. Pushes to the Ryvn Registry (unless `build_only` is set to true)

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
