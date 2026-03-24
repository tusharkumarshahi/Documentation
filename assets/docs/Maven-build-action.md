# Composite Action — Maven Build

## Scope

Reusable GitHub Actions composite action that handles the Maven build step for any Java or MuleSoft Maven project. Designed to be called from any repository workflow within the `medica-dev-platform` organization via `github-common-actions`.

Covers:
- Java setup with Maven dependency caching
- Optional `settings.xml` injection for internal repository authentication
- Maven goals execution with configurable flags
- Built artifact detection and output exposure for downstream steps

Does not cover deployment, Exchange publish, or Docker packaging — those are handled by separate composite actions.

---

## Logic

The action executes four steps in sequence:

**1. Set up Java**
Sets up the requested JDK version using `actions/setup-java@v5` with Maven dependency caching enabled. Java version and distribution are configurable via inputs — defaults to Temurin 17.

**2. Validate settings.xml**
If a `settings-xml-path` input is provided, the action verifies the file exists at that path before attempting the build. Fails fast with a clear error message if not found. Skips silently if no path is provided — allowing the action to work with public Maven Central without any configuration.

**3. Maven build**
Runs `mvn clean package` (configurable via `maven-goals` input). The `settings.xml` path is resolved via an environment variable to avoid shell quoting issues. Additional `-D` flags are passed via `build-args` input — keeping the action generic and decoupled from any specific project's credential requirements.

**4. Find built artifact**
Scans `target/` for the primary `.jar`, excluding `-sources` and `-javadoc` jars. Exposes the artifact path and filename as step outputs for use in downstream steps such as Docker build or artifact upload.

---

## Why This Approach?

**Composite action over reusable workflow**  
Runs in the same job (shared runner/workspace), so downstream steps can use the built `.jar` without extra artifact handling.

**Credentials via `build-args` (not hardcoded)**  
All MuleSoft flags are passed from the caller, keeping the action generic and reusable.

**`settings.xml` via env var**  
Using `SETTINGS_PATH` avoids shell quoting/escaping issues.

**`setup-java@v5` for caching**  
Enables Maven caching (`cache: maven`) to speed up builds.

---

## Secret / Environment Handling

The action itself holds no secrets. All credentials are passed by the calling workflow via the `build-args` input and injected as Maven `-D` properties at build time.

| Input | Required | Description |
|-------|----------|-------------|
| `java-version` | No (default: `17`) | JDK version |
| `java-distribution` | No (default: `temurin`) | JDK distribution |
| `maven-goals` | No (default: `clean package`) | Maven goals to run |
| `skip-tests` | No (default: `false`) | Skip test execution |
| `working-directory` | No (default: `.`) | Directory containing `pom.xml` |
| `build-args` | No | Additional Maven `-D` flags |
| `settings-xml-path` | No | Path to `settings.xml` relative to workspace root |
---

## Code / PR Link

| Item | Link |
|------|------|
| Composite action file | `github-common-actions/.github/actions/maven-build/action.yml` |
| Branch | `feature/maven-build-action` |

---

## Walkthrough / Runbook

### Action Location

```
github-common-actions
└── .github/
    └── actions/
        └── maven-build/
            └── action.yml
```

### How to Call This Action

**Minimal (public Maven Central, no auth):**
```yaml
- name: Build
  id: build
  uses: medica-dev-platform/github-common-actions/.github/actions/maven-build@main
  with:
    java-version: '17'
```

**Using outputs in a downstream step:**
```yaml
- name: Upload artifact
  uses: actions/upload-artifact@v4
  with:
    name: app-jar
    path: ${{ steps.build.outputs.artifact-path }}
```

### Runner Requirements

| Requirement | Status |
|-------------|--------|
| Java 17 | Installed by `setup-java@v5` automatically |
| Maven 3.9.9 | Must be pre-installed on self-hosted runner |
| Network access to `nexus.medica.com` | Required for MuleSoft builds |
| Runner label | `self-hosted` |

### Testing Status

| Test | Environment | Runner | Result |
|------|-------------|--------|--------|
| Action structure and wiring | Personal GitHub (`test-mule-actions`) | `ubuntu-latest` | ✅ Passed |
| Cross-repo action call | Client GHE (`prc-claims-payment-recon-api`) | `dlx110-ratp002-1` (self-hosted) | ✅ Action loaded, Maven executed |
| Full MuleSoft build | Client GHE | `dlx110-ratp002-1` | ⏳ Pending — blocked on secrets provisioning and Nexus access |

### Outputs Reference

| Output | Description | Example |
|--------|-------------|---------|
| `artifact-path` | Relative path to built `.jar` | `target/my-app-1.0.0.jar` |
| `artifact-name` | Filename of built `.jar` | `my-app-1.0.0.jar` |