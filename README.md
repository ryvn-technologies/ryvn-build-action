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

## Inputs

| Name                 | Description                                               | Required | Default |
| -------------------- | --------------------------------------------------------- | -------- | ------- |
| `service_name`       | Name of the service                                       | Yes      |         |
| `version`            | Semantic version to tag the image/chart with              | Yes      |         |
| `build_only`         | Build only, don't push to registry                        | No       | `false` |
| `ryvn_client_id`     | Ryvn Client ID for authentication                         | Yes      |         |
| `ryvn_client_secret` | Ryvn Client Secret for authentication                     | Yes      |         |
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
