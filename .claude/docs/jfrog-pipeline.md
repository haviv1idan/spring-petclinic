# JFrog Build Pipeline

## What This Does

The workflow `.github/workflows/jfrog-build.yml` implements a CI pipeline that:

1. Checks out source code
2. Resolves all Maven dependencies through JFrog Artifactory (not from Maven Central directly)
3. Builds and deploys the Spring PetClinic JAR to JFrog Artifactory
4. Publishes build metadata (build-info) to Artifactory
5. Scans the build with JFrog Xray (non-failing — results are visible in the JFrog UI)

Triggered by: `workflow_dispatch` (manual) or `pull_request` targeting `main`.

## JFrog Platform Setup

Before the pipeline can run, create the following in your JFrog instance.

### Repositories

| Key | Type | Notes |
|---|---|---|
| `petclinic-maven-central` | Remote (Maven) | URL: `https://repo1.maven.org/maven2` — proxies Maven Central |
| `petclinic-libs-release-local` | Local (Maven) | Stores published release JARs |
| `petclinic-libs-snapshot-local` | Local (Maven) | Stores published snapshot JARs |
| `petclinic-virtual` | Virtual (Maven) | Aggregates all three above; default deployment: `petclinic-libs-release-local` |

Enable **Xray indexing** on all four repositories (each repo's Xray tab → Index Repository).

### Access Token

Administration → Identity and Access → Access Tokens → Generate Token.

Required permissions:
- Read on `petclinic-virtual`
- Deploy/write on `petclinic-libs-release-local` and `petclinic-libs-snapshot-local`
- Publish Build Info

### Xray Watches and Policies (Optional)

Security & Compliance → Xray → Watches → create a watch covering the repositories.
Attach a Policy to define CVE severity thresholds.
Without a policy, Xray indexes and reports but never blocks builds.

## GitHub Secrets

Add these in Repository → Settings → Secrets and variables → Actions:

| Secret | Value |
|---|---|
| `JF_URL` | `https://yourinstance.jfrog.io` (no trailing slash) |
| `JF_ACCESS_TOKEN` | Access token generated above |

## How the Pipeline Works

### Why JFrog CLI

`jfrog/setup-jfrog-cli@v4` is JFrog's recommended integration for GitHub Actions. It:
- Authenticates with `JF_URL` + `JF_ACCESS_TOKEN` from secrets
- Provides `jf mvn-config` to inject Artifactory repository routing into Maven at build time (no `settings.xml` edits required)
- Collects build-info (dependency list + artifact checksums) that Xray uses for scanning

### Why Maven is Installed Separately

The workflow uses `jf mvn` — JFrog's wrapper around Maven. Under the hood, `jf mvn` just calls the regular `mvn` command, so it needs `mvn` to exist on the machine.

The problem: `ubuntu-latest` (the GitHub Actions runner) ships with Maven 3.6, but this project requires Maven 3.9 (that's what the `./mvnw` wrapper downloads when you run it locally).

You might wonder: why not just use `./mvnw`? Because `jf mvn` doesn't know about the Maven wrapper — it looks for the `mvn` binary on `PATH`, not `./mvnw`.

So the install step downloads Maven 3.9.9 from the official Apache archive and puts it on `PATH` so that when `jf mvn` runs, it finds the right version.

### Release vs Snapshot Repositories

Maven automatically routes artifacts based on the version string in `pom.xml`:

- Version ends in `-SNAPSHOT` (e.g., `4.0.0-SNAPSHOT`) → deploys to `petclinic-libs-snapshot-local`
- Version has no `-SNAPSHOT` suffix (e.g., `4.0.0`) → deploys to `petclinic-libs-release-local`

No pipeline changes are needed — `jf mvn-config` already has both repos configured. Changing the version in `pom.xml` is enough for Maven to pick the correct target.

### Why `mvn deploy` (Not `package` or `verify`)

`jf mvn deploy` runs the full Maven lifecycle through the `deploy` phase, which uploads the JAR to Artifactory. JFrog CLI intercepts the deploy phase and routes artifacts to the configured local repositories — no `<distributionManagement>` in `pom.xml` is required. Build-info (the metadata that Xray traces) is published separately in the next step.

### Why No Maven Cache

The `actions/setup-java@v4` step intentionally omits `cache: maven`. Enabling the GitHub cache would serve dependencies from disk on subsequent runs, bypassing Artifactory. All dependency resolution must flow through `petclinic-virtual` to satisfy the assignment requirement.

## Verifying the Pipeline

### Run Log

| Step | Expected output |
|---|---|
| Set up JFrog CLI | "Connected to JFrog Platform" |
| Configure Maven | Step completes; JFrog CLI prints the configured repo keys |
| Build and deploy | Maven download URLs reference `yourinstance.jfrog.io/artifactory/petclinic-virtual/...` not `repo.maven.apache.org` |
| Publish build info | "Build info successfully published" |
| Xray scan | "Scan of build spring-petclinic/N completed" — exit 0 regardless |

### Artifactory UI

- Artifacts → `petclinic-libs-snapshot-local` → `org/springframework/samples/spring-petclinic/4.0.0-SNAPSHOT/` — JAR, POM, and checksums present
- Application → Builds → `spring-petclinic` → click the run number → Artifacts and Dependencies tabs

### Xray Results

Application → Security → Builds → `spring-petclinic` → run number → Security Issues tab.
Spring PetClinic contains known vulnerabilities by design, so this list should be non-empty.
