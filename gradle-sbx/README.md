# gradle-artifactory sbx kit

SBX v3 mixin kit that gives a sandbox your Artifactory Gradle setup without
hand-written credentials and without the password entering the sandbox. It is
the Gradle counterpart of the Maven kit in the repository root.

## One-time host setup

The secret is the base64 of `username:password` (v3 kits build the header from
`format: "Basic %s"`). It is a separate service from the Maven kit's
`maven-artifactory`, but it holds the same value:

```sh
printf '%s' 'matthisk:YOUR_PASSWORD' | base64 | tr -d '\n' | sbx secret set gradle-artifactory
```

If `sbx secret set` only prompts interactively, run `sbx secret set gradle-artifactory`
and paste the base64 output of the command above.

## Use

Credentials are applied at sandbox creation, so add the kit when creating it
(`sbx kit add` can't add credentials to a running sandbox):

```sh
sbx run <agent> --kit /path/to/maven-sbx/gradle
```

Approve the credential request on first run (recorded in
`~/.config/sbx/credentials.yaml`). Unattended/`--detach` runs don't prompt, so
run once interactively first.

## What it does

- `credential@1` service `gradle-artifactory`: the proxy sets
  `Authorization: Basic <secret>` on requests to `artifactory.tools.devrtb.com`.
  `GRADLE_ARTIFACTORY_AUTH` in the sandbox holds only a sentinel.
- `network-policy@2`: allows that host, plus `services.gradle.org` and
  `downloads.gradle.org` for the wrapper's distribution download.
- `lifecycle@1` file `~/.gradle/gradle.properties`: Gradle ignores `HTTPS_PROXY`, so
  `systemProp.http(s).proxyHost/Port` point it at the sbx proxy
  (`gateway.docker.internal:3128`, hardcoded). Without that it bypasses the proxy and
  its credential injection and gets a 401. The same file sets
  `systemProp.javax.net.ssl.trustStore*` to the truststore below. User-home
  `gradle.properties` wins over a project's, and both the wrapper and the daemon
  apply these properties. `overwrite: false` keeps any existing file.
- `lifecycle@1` file `~/.gradle/init.d/sbx-artifactory.gradle`: adds Artifactory
  (no credentials block) to the plugin, settings buildscript, `dependencyResolutionManagement`
  (Gradle 6.8+), project buildscript and project repositories, registered before
  anything the build declares, so it is tried first. Project repositories are only
  touched when `repositoriesMode` is `PREFER_PROJECT` (the default), so
  `FAIL_ON_PROJECT_REPOS` builds don't break.
- `lifecycle@1` startup hook: on every boot, builds `~/.gradle/sbx-truststore.p12`
  from `/etc/ssl/certs/ca-certificates.crt` with `openssl pkcs12 -export -nokeys`.
  The sbx proxy CA is per sandbox and already in that bundle, which is why curl works
  but a JDK with its own `cacerts` fails with `PKIX path building failed`.

## Limits

- The truststore and proxy are system properties of the wrapper and daemon JVMs. Forked
  test JVMs and other workers don't inherit them.
- A build's own `mavenCentral()` / `gradlePluginPortal()` stay in the list after
  Artifactory. Misses on Artifactory then fall through to hosts the network policy
  blocks. Artifactory `java-virtual` has to serve the Gradle plugins you use.
- If the wrapper's distribution download redirects to a host that isn't allowed
  (for example GitHub release assets), allow it with `sbx policy allow network <host>`,
  or point `distributionUrl` at an Artifactory mirror.
- `gradle.properties` has `overwrite: false`, so sandboxes created with an older
  version of this kit keep the old file.

The sandbox needs a JDK (and Gradle, or a project wrapper) from your workload or another kit.
