# ryvn-build-action

GitHub Action for Building and Publishing to Ryvn Registry

## Overview

This GitHub Action builds and pushes Docker images or Helm charts to the Ryvn Registry. It supports:

- Docker images built from Dockerfiles
- Docker images built using Nixpacks (without Dockerfile)
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
3. Authenticates with AWS ECR
4. Builds the Docker image or packages the Helm chart
5. Pushes to the Ryvn Registry (unless `build_only` is set to true)

## License

MIT
