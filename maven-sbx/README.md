# maven-sbx

SBX v3 mixin kit that gives a sandbox your Artifactory Maven setup without a
hand-written `~/.m2/settings.xml` and without the password entering the sandbox.

For Gradle, see the sibling kit in [`gradle/`](gradle/README.md).

## One-time host setup

The secret is the base64 of `username:password` (v3 kits build the header from
`format: "Basic %s"`):

```sh
printf '%s' 'matthisk:YOUR_PASSWORD' | base64 | tr -d '\n' | sbx secret set maven-artifactory
```

If `sbx secret set` only prompts interactively, run `sbx secret set maven-artifactory`
and paste the base64 output of the command above.

## Use

Credentials are applied at sandbox creation, so add the kit when creating it
(`sbx kit add` can't add credentials to a running sandbox):

```sh
sbx run <agent> --kit /path/to/maven-sbx
```

Approve the credential request on first run (recorded in
`~/.config/sbx/credentials.yaml`). Unattended/`--detach` runs don't prompt, so
run once interactively first.

## What it does

- `credential@1` service `maven-artifactory`: the proxy sets
  `Authorization: Basic <secret>` on requests to `artifactory.tools.devrtb.com`.
  `MAVEN_ARTIFACTORY_AUTH` in the sandbox holds only a sentinel.
- `network-policy@2`: allows that host.
- `lifecycle@1` file: writes `/home/agent/.m2/settings.xml` (repos, plugin repos,
  active profile; no `<servers>`). `overwrite: false` keeps any existing file.

- `lifecycle@1` `<proxies>` in `settings.xml`: Maven 3.10 ignores `HTTPS_PROXY` and the
  `-Dhttps.proxyHost` flags in `JAVA_TOOL_OPTIONS`, so it would bypass the sbx proxy
  (and its credential injection) and get a 401. The proxy host/port
  (`gateway.docker.internal:3128`) is hardcoded.
- `lifecycle@1` startup hook: on every boot, builds `~/.m2/sbx-truststore.p12` from
  `/etc/ssl/certs/ca-certificates.crt` with `openssl pkcs12 -export -nokeys`. The sbx
  proxy CA is per sandbox and already in that bundle, which is why curl works but a Nix
  JDK (own `cacerts`) fails with `PKIX path building failed`. No keytool/JRE needed, and
  the CA is never copied into the kit.
- `lifecycle@1` install hook: appends a guarded `MAVEN_OPTS` export
  (`-Djavax.net.ssl.trustStore=...`) to `/etc/sandbox-persistent.sh`, idempotently.
  Not `~/.mvn/jvm.config`, because a repo's own `.mvn/jvm.config` wins.

Limits: `MAVEN_OPTS` only covers the Maven JVM, not forked surefire JVMs. `settings.xml`
has `overwrite: false`, so sandboxes created with an older version of this kit keep the
old file. Create a new sandbox to pick up the `<proxies>` block.

The sandbox needs Maven itself (from your workload or another kit).
If your `central` and `snapshots` logins use different passwords, split this into
two services; this kit assumes one login.
