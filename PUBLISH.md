# Publishing to Maven Central

This guide describes how to publish `rsocket-broker` and `rsocket-broker-client` artifacts to Maven Central via Sonatype OSSRH.

## Prerequisites

### 1. Sonatype OSSRH Account

- Register at [https://central.sonatype.com](https://central.sonatype.com)
- If you already have credentials for `oss.sonatype.org`, they still work with the existing Gradle config
- Verify you own the `io.rsocket.broker` namespace (already done for this project)

### 2. GPG Signing Key

Generate a GPG key pair:

```bash
gpg --full-generate-key    # Choose RSA 4096-bit, no expiration
gpg --list-secret-keys --keyid-format LONG    # Note the key ID
gpg --keyserver keyserver.ubuntu.com --send-keys <KEY_ID>    # Publish public key
```

Export the private key for CI use:

```bash
gpg --armor --export-secret-keys <KEY_ID> | base64    # This is your signingKey
```

### 3. Configure GitHub Secrets

In **both** repos (`rsocket-broker` and `rsocket-broker-client`), go to **Settings → Secrets and variables → Actions** and add:

| Secret             | Value                                            |
| ------------------ | ------------------------------------------------ |
| `signingKey`       | Base64-encoded GPG private key (armor format)    |
| `signingPassword`  | Passphrase for the GPG key                       |
| `sonatypeUsername` | Your Sonatype OSSRH username (or token username) |
| `sonatypePassword` | Your Sonatype OSSRH password (or token password) |

> **Recommended**: Use Sonatype **token credentials** instead of account passwords. Generate them at Sonatype → Profile → User Token.

## Release Process

### Step 1: Release rsocket-broker-client first

`rsocket-broker` depends on `rsocket-broker-client`, so always release the client first.

Edit `rsocket-broker-client/gradle.properties` and change the version from SNAPSHOT to release:

```properties
version=0.4.0
```

### Step 2: Tag and push rsocket-broker-client

```bash
cd rsocket-broker-client
git add -A
git commit -m "Release 0.4.0"
git tag 0.4.0
git push origin main --tags
```

This triggers the `gradle-release.yml` workflow, which:

1. Builds the project (skipping tests)
2. Signs all artifacts with GPG
3. Publishes to the Sonatype staging repository

### Step 3: Update and release rsocket-broker

Edit `rsocket-broker/gradle.properties`:

```properties
version=0.4.0
rsocketBrokerClientVersion=0.4.0
```

Then tag and push:

```bash
cd rsocket-broker
git add -A
git commit -m "Release 0.4.0"
git tag 0.4.0
git push origin main --tags
```

### Step 4: Promote the staging repository to Maven Central

After the CI publishes to Sonatype, artifacts land in a **staging repository**:

1. Log in to [https://oss.sonatype.org](https://oss.sonatype.org)
2. Navigate to **Staging Repositories**
3. Find your repository (named like `iorsocketbroker-XXXX`)
4. **Close** the repository — this triggers validation (POM, signatures, javadoc, sources)
5. If validation passes, **Release** the repository
6. Artifacts sync to Maven Central within ~10–30 minutes

### Step 5: Bump to next SNAPSHOT

In both repos, update `gradle.properties`:

```properties
version=0.5.0-SNAPSHOT
rsocketBrokerClientVersion=0.5.0-SNAPSHOT  # rsocket-broker only
```

Commit and push:

```bash
git commit -am "Prepare next development version 0.5.0-SNAPSHOT"
git push origin main
```

## Local Testing

### Publish to local Maven repository

```bash
./gradlew publishMavenPublicationToMavenLocal \
  -Pversion=0.4.0 \
  -PsigningKey="$(cat ~/.gnupg/private-key.asc)" \
  -PsigningPassword="your-passphrase"
```

Artifacts go to `~/.m2/repository/io/rsocket/broker/`.

### Manual publish to Sonatype (without CI)

```bash
./gradlew clean build -x test
./gradlew -Pversion=0.4.0 \
  -PsonatypeUsername=your-username \
  -PsonatypePassword=your-password \
  -PsigningKey="$(cat ~/.gnupg/private-key.asc)" \
  -PsigningPassword="your-passphrase" \
  sign publishMavenPublicationToSonatypeRepository
```

## Notes

- The Gradle config in `gradle/sonotype.gradle` uses the legacy OSSRH URL (`oss.sonatype.org`). For existing namespaces like `io.rsocket.broker`, this continues to work. New namespaces require the Central Portal at `central.sonatype.com`.
- The `gradle/publications.gradle` file configures the POM metadata (license, developers, SCM) required by Maven Central.
- Signing is handled via in-memory PGP keys, suitable for both local development and CI.
