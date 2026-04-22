<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2023-2026 The Linux Foundation
-->

# 🔐 Nexus Docker Login Action

Performs `docker login` against the Docker registries exposed by a
Sonatype Nexus3 instance and, optionally, DockerHub, in a single step.

All inputs are optional. The action logs in to whichever targets the
caller has configured and skips the rest with a warning. This lets
callers wire the inputs up directly to organisation-level variables
and secrets (e.g. `vars.NEXUS3_REGISTRY`, `secrets.NEXUS3_PASSWORD`)
and leave anything the org does not define empty.

## nexus-docker-login-action

## Usage Example

### Basic usage (ONAP-style, full login)

Logs in to the four default Nexus3 registry ports and to DockerHub
with credentials supplied by the calling workflow:

<!-- markdownlint-disable MD013 -->

```yaml
steps:
  - name: "Docker Registry Login"
    id: docker_login
    uses: lfreleng-actions/nexus-docker-login-action@main
    with:
      nexus3-registry:    ${{ vars.NEXUS3_REGISTRY }}
      nexus3-user:        ${{ vars.NEXUS3_USER }}
      nexus3-password:    ${{ secrets.NEXUS3_PASSWORD }}
      dockerhub-user:     ${{ vars.DOCKERHUB_USER }}
      dockerhub-password: ${{ secrets.DOCKERHUB_PASSWORD }}
```

<!-- markdownlint-enable MD013 -->

### Nexus3 anonymous pull registry (no credentials available)

For organisations that do not configure authenticated Nexus3
credentials or DockerHub credentials. The action logs in to the
anonymous port (the first entry in `nexus3-ports`) and emits a
warning for each remaining port that it skips:

<!-- markdownlint-disable MD013 -->

```yaml
steps:
  - name: "Docker Registry Login"
    uses: lfreleng-actions/nexus-docker-login-action@main
    with:
      nexus3-registry: ${{ vars.NEXUS3_REGISTRY }}
```

<!-- markdownlint-enable MD013 -->

### DockerHub credentials without Nexus3

<!-- markdownlint-disable MD013 -->

```yaml
steps:
  - name: "Docker Registry Login"
    uses: lfreleng-actions/nexus-docker-login-action@main
    with:
      dockerhub-user:     ${{ vars.DOCKERHUB_USER }}
      dockerhub-password: ${{ secrets.DOCKERHUB_PASSWORD }}
```

<!-- markdownlint-enable MD013 -->

## Inputs

<!-- markdownlint-disable MD013 -->

| Name               | Required | Default                   | Description                                                                                                                                                                                                                                     |
| ------------------ | -------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| nexus3-registry    | False    |                           | Nexus3 registry hostname (no scheme, no path, no port), e.g. `nexus3.example.org`. Scheme and path get stripped automatically. When empty, the action skips all Nexus3 logins                                                                   |
| nexus3-user        | False    |                           | Nexus3 organisation username. Required for authenticated Nexus3 ports (all entries in `nexus3-ports` after the first). When `nexus3-user` or `nexus3-password` is empty, the action skips authenticated ports; the anonymous port still logs in |
| nexus3-password    | False    |                           | Nexus3 organisation user's password. Required alongside `nexus3-user` for authenticated Nexus3 ports                                                                                                                                            |
| nexus3-ports       | False    | `10001,10002,10003,10004` | Comma-separated Nexus3 Docker registry ports. The first entry is the anonymous/public port and authenticates with `docker`/`docker`; remaining entries require both `nexus3-user` and `nexus3-password`                                         |
| dockerhub-user     | False    |                           | DockerHub organisation username. Required alongside `dockerhub-password`. When `dockerhub-user` or `dockerhub-password` is empty, the action skips the DockerHub login                                                                          |
| dockerhub-password | False    |                           | DockerHub organisation user's password. Required alongside `dockerhub-user`                                                                                                                                                                     |
| dockerhub-registry | False    | `docker.io`               | DockerHub registry hostname. Override for alternative DockerHub-compatible registries                                                                                                                                                           |

<!-- markdownlint-enable MD013 -->

## Outputs

<!-- markdownlint-disable MD013 -->

| Name                 | Description                                                                                                                                                 |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| logged-in-registries | Comma-separated list of registries logged in to. Nexus3 entries take the form `host:port`; the DockerHub entry is a bare `host` (no port, e.g. `docker.io`) |

<!-- markdownlint-enable MD013 -->

## Recommended caller pattern

Composite actions cannot read `vars.*` or `secrets.*` directly; the
calling workflow must do that. The canonical pattern used across the
`lfreleng-actions` / `releng-reusable-workflows` ecosystem passes
them through from the workflow:

<!-- markdownlint-disable MD013 -->

```yaml
jobs:
  docker-login:
    runs-on: ubuntu-latest
    steps:
      - uses: lfreleng-actions/nexus-docker-login-action@v1
        with:
          nexus3-registry:    ${{ vars.NEXUS3_REGISTRY }}
          nexus3-user:        ${{ vars.NEXUS3_USER }}
          nexus3-password:    ${{ secrets.NEXUS3_PASSWORD }}
          dockerhub-user:     ${{ vars.DOCKERHUB_USER }}
          dockerhub-password: ${{ secrets.DOCKERHUB_PASSWORD }}
```

<!-- markdownlint-enable MD013 -->

These names match the variables and secrets already configured in the
ONAP, O-RAN-SC, and OpenDaylight organisations and used by the
`compose-docker-verify.yaml` / `compose-docker-release-verify.yaml`
reusable workflows. Leave any input empty when the organisation does
not define the corresponding variable or secret — the action skips
that section and continues.

## Requirements / Dependencies

- Docker must be available on the runner; the action calls
  `docker login --password-stdin` directly (no secrets appear on the
  command line).

## Notes

- The first entry in `nexus3-ports` follows the Sonatype Nexus3
  convention of being an anonymous/public pull registry using fixed
  `docker`/`docker` credentials. All remaining entries get treated as
  authenticated endpoints.
- The default port list (`10001,10002,10003,10004`) matches the ONAP
  / O-RAN-SC / OpenDaylight deployment layout:
  - Port `10001`: anonymous/public pull
  - Port `10002`: release push
  - Port `10003`: snapshot/daily push
  - Port `10004`: staging
- `nexus3-registry` gets sanitised automatically: the action strips
  leading `http://` / `https://` and any trailing path. The action
  rejects an embedded port (`host:port`) — use `nexus3-ports` instead.
- The action fails when *no* login can get performed (all inputs
  empty or effectively empty) or when an *attempted* login fails. A
  missing optional section (for example, absent DockerHub credentials)
  logs a warning and continues.
- DockerHub authentication defaults to `docker.io`; override via
  `dockerhub-registry` if you need a different DockerHub-compatible
  endpoint.
