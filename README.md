# sbx-kits

A collection of [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/customize/author/) (`sbx`) **v3 mixin kits**. Each kit adds one capability to any sbx workload, such as a toolchain loader or authenticated access to a private artifact repository. Credentials stay on the host: the sandbox proxy injects them into outgoing requests, so the real secrets never enter the sandbox.

## Kits

| Kit | What it does |
|---|---|
| [`nix-direnv-kit`](nix-direnv-kit/README.md) | Adds Nix (single-user, flakes), direnv and nix-direnv. The sandbox loads the project's `.envrc` (for example `use flake`) in interactive shells and in agent commands. Supports prewarming and an optional host-side binary cache. |
| [`maven-sbx`](maven-sbx/README.md) | Writes `~/.m2/settings.xml` for the Artifactory Maven repository. The proxy injects credentials, and the kit configures the proxy and a JDK truststore for Maven. |
| [`gradle-sbx`](gradle-sbx/README.md) | The Gradle counterpart of `maven-sbx`. It sets up `gradle.properties` (proxy and truststore), an init script that adds Artifactory repositories, and network access for the Gradle wrapper. |
| [`artifactory-sbx`](artifactory-sbx/README.md) | The minimal Artifactory kit. It allows the Artifactory host and injects credentials for any build tool (npm, yarn, Maven, Gradle, …) with no tool-specific config. |

## Prerequisites

- Docker Sandboxes (`sbx`). The kits are developed against v0.50.
- For the Artifactory kits: a one-time host secret holding the base64 of `username:password`, set with `sbx secret set <service>`. Each kit's README names its service and gives the exact command.

## Usage

Kits can come from Docker Hub, a local directory or git:

```sh
# Published kit
sbx run <WORKLOAD> ~/code/my-project --kit docker.io/matthisk/nix-direnv-kit:<VERSION>

# Local checkout
sbx run <WORKLOAD> ~/code/my-project --kit ./maven-sbx
```

Notes:

- `<WORKLOAD>` must be a v3 workload kit. See the `nix-direnv-kit` README for setting up `kit.shortcuts`.
- Credential kits (`maven-sbx`, `gradle-sbx`) apply credentials at sandbox creation, so pass them when creating the sandbox. On first run, approve the credential request interactively.
- You can combine kits by passing `--kit` more than once, for example Nix for the JDK plus `maven-sbx` for Maven access.

## Repository layout

```
<kit>/
  kit.yaml | <kit>.yaml   # v3 source kit (capabilities: network policy, credentials, lifecycle hooks)
  README.md               # kit-specific setup and details
```

`nix-direnv-kit` also contains a Dockerfile, helper scripts under `scripts/`, and an nginx config for the optional local cache.

## Development

Validate a kit and publish it with BuildKit. The kits are v3 source kits built by the `docker/sandbox-kit:3` frontend:

```sh
sbx kit validate ./<kit>
docker buildx build -f <kit>/kit.yaml -t docker.io/<user>/<kit>:<tag> --push <kit>
```

Check each kit's README for any kit-specific build or publish steps.
