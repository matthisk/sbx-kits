# Maven Artifactory sandbox kit

A Docker Sandboxes (`sbx`) mixin kit that lets build tools inside a sandbox (npm, yarn, Maven, Gradle, …) authenticate to Artifactory at `artifactory.tools.devrtb.com` without the real credentials ever entering the sandbox.

## What it does

- **Network policy:** allows outbound traffic from the sandbox to `artifactory.tools.devrtb.com`.
- **Credential injection:** the sandbox proxy adds an `Authorization: Basic <secret>` header to requests going to that domain. Inside the sandbox, `MAVEN_ARTIFACTORY_AUTH` holds only a sentinel value, and the real secret stays on the host.

The credential is optional, so the kit still loads when no secret has been set (requests then go out unauthenticated).

## Setup

1. Base64-encode your Artifactory username and password:

   ```sh
   printf '%s' '<username>:<password>' | base64
   ```

2. Store the result as an `sbx` secret on the host (you'll be prompted for the value):

   ```sh
   sbx secret set maven-artifactory
   ```

## Usage

Add the kit to a sandbox from Docker Hub:

```sh
sbx kit add <sandbox> docker.io/matthisk/artifactory-sbx:latest
```

Or use the local directory:

```sh
sbx kit add <sandbox> ./artifactory-sbx
```

Point your build tool at `https://artifactory.tools.devrtb.com/...` as usual. You don't need credentials in `settings.xml`, `.npmrc`, or `gradle.properties`, because the proxy injects the header.

## Development

```sh
sbx kit validate .

# Publish. kit.yaml is a v3 source kit, built by the docker/sandbox-kit:3
# BuildKit frontend. `sbx kit push` (v0.50.1) only accepts spec.yaml kits.
docker buildx build -f kit.yaml -t docker.io/matthisk/artifactory-sbx:latest --push .
```
