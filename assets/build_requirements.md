# Build Requirements — MuleSoft GitHub Actions Migration

> **Scope:** All environments (Dev, QA, Stage, Perf, Prod)
> **Pipeline files reviewed:** `cicd-dev-ch2.yml`, `cicd-stage-ch2.yml`, `cicd-prod-ch2.yml`
> **Purpose:** All secrets, credentials, and infrastructure requirements needed to execute the Maven build step in GitHub Actions.

---

## 1. Overview

The Maven build step runs `mvn clean package` and executes MUnit tests. It requires credentials to authenticate against:

- **Medica's internal Nexus** — all Maven dependencies are proxied internally
- **Anypoint Exchange** — MuleSoft EE dependencies resolved via Exchange server entries
- **MuleSoft EE license** — required to download and use the MuleSoft EE runtime
- **sys-mule-platform-api-ch2** — internal Medica platform API called during build validation

The build step is **identical across all environments** with one exception — the `ADMIN_API_*` credentials use environment-specific variants.

---

## 2. GitHub Actions Runner Requirements

| Requirement | Detail |
|-------------|--------|
| Runner type | Self-hosted |
| Runner label | `medica-java-pool` |
| Java | Pre-installed or managed via `actions/setup-java@v5` |
| Maven | 3.9.9 pre-installed |
| Network access | Must reach `nexus.medica.com` (internal Medica network) |
| Node.js | 20+ (required by `actions/setup-java@v5`) |
| Runner agent | v2.327.1 or later |

> **Note:** GitHub-hosted runners (`ubuntu-latest`) cannot be used for this build. All Maven dependencies route through `nexus.medica.com` which is only accessible from inside Medica's internal network.

---

## 3. Repository Requirements

| Repository | Purpose | Required Action |
|-----------|---------|----------------|
| `medica-mule-assets` | Provides `settings.xml` with Nexus mirror and server configuration | Must be mirrored/moved to GitHub Enterprise |
| `prc-claims-payment-recon-api` | Application source code | Already on GitHub Enterprise |
| `medica-mule-actions` | Shared composite actions repo | Must be created in GitHub Enterprise |

---

## 4. Secrets Required — Build Step

### 4.1 Org-Level Secrets (Shared Across All APIs and Environments)

These credentials are the same regardless of which API is being built or which environment is targeted.

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|-------------|
| `MULE_EE_USER` | `MuleEEUser` | `onp-deployment` | MuleSoft Enterprise Edition username |
| `MULE_EE_PASSWORD` | `MuleEEPassword` | `onp-deployment` | MuleSoft Enterprise Edition password |
| `NEXUS_MEDICA_MULE_INTERNAL` | `nexusMedicaMuleInternal` | `onp-deployment` | Nexus internal repository password — fills `${nexus.medica.mule.internal}` in `settings.xml` |
| `MULE_CONNECTED_APP_TOKEN` | `mule.connected.app.token` | `onp-deployment` | Anypoint Connected App token — fills `${mule.connected.app.token}` in `settings.xml` for Exchange server entries (`anypoint-exchange-v2`, `anypoint-exchange-v3`, `mulesoft-exchange`) |

### 4.2 Environment-Specific Secrets — Build Step (Admin API)

The `ADMIN_API_*` credentials authenticate against `sys-mule-platform-api-ch2` during the build. They vary by environment.

| GitHub Secret Name | Dev | Stage | Prod | ADO Variable Group |
|-------------------|-----|-------|------|--------------------|
| `ADMIN_API_CLIENT_ID_<ENV>` | `PumaPropsProv.SysMuleAdmin.ClientID.QA` | `PumaPropsProv.SysMuleAdmin.ClientID.STG` | `PumaPropsProv.SysMuleAdmin.ClientID.PROD` | `PumaPropsProv` |
| `ADMIN_API_CLIENT_SECRET_<ENV>` | `PumaPropsProv.SysMuleAdmin.ClientSecret.QA` | `PumaPropsProv.SysMuleAdmin.ClientSecret.STG` | `PumaPropsProv.SysMuleAdmin.ClientSecret.PROD` | `PumaPropsProv` |
| `ADMIN_API_HOST_<ENV>` | `PumaPropsProv.SysMuleAdmin.URL.DEV` | `PumaPropsProv.SysMuleAdmin.URL.STG` | `PumaPropsProv.SysMuleAdmin.URL.PROD` | `PumaPropsProv` |
| `ADMIN_TOKEN_URL` | `PumaPropsProv.Token.URL.NonProd` | `PumaPropsProv.Token.URL.NonProd` | `PumaPropsProv.Token.URL.Prod` | `PumaPropsProv` |

> **Note:** Dev and Stage share `Token.URL.NonProd`. Prod uses a separate `Token.URL.Prod`.
> **Note:** Dev build uses `ClientID.QA` — not a Dev-specific credential. This appears intentional in the pipeline.

### 4.3 Code Quality Secret

| GitHub Secret Name | ADO Variable | Description |
|-------------------|-------------|-------------|
| `SONAR_TOKEN` | `sonar.token` (currently hardcoded) | SonarQube authentication token |

> ⚠️ **Critical:** The current ADO pipeline has the SonarQube token hardcoded in `cicd-dev-ch2.yml` Line 387 (`sqa_6587a7e6050a617a662bf05f525e988751199c22`). This token must be **rotated immediately** and stored as a proper secret before GitHub migration. Do not use the hardcoded value.

---

## 5. ADO Variable Groups to Export

Request the following variable groups from the client in full:

| Variable Group | Contains |
|---------------|---------|
| `onp-deployment` | `MuleEEUser`, `MuleEEPassword`, `nexusMedicaMuleInternal`, `mule.connected.app.token` |
| `PumaPropsProv` | All `SysMuleAdmin.*` and `Token.URL.*` credentials per environment |

> **Note:** The exact name of the `PumaPropsProv` variable group in ADO Library needs to be confirmed with the client. `PumaPropsProv` is the alias used inside the pipeline — the actual group name may differ.

---

## 6. settings.xml Server Entries — Credential Mapping

`settings.xml` from `medica-mule-assets` contains the following server entries that require credentials at build time:

| Server ID | Username | Password Placeholder | GitHub Secret |
|-----------|---------|---------------------|---------------|
| `mulesoft-internal` | `mulesoftDeploy` (hardcoded) | `${nexus.medica.mule.internal}` | `NEXUS_MEDICA_MULE_INTERNAL` |
| `anypoint-exchange-v3` | `~~~Token~~~` (literal) | `${mule.connected.app.token}` | `MULE_CONNECTED_APP_TOKEN` |
| `anypoint-exchange-v2` | `~~~Token~~~` (literal) | `${mule.connected.app.token}` | `MULE_CONNECTED_APP_TOKEN` |
| `mulesoft-exchange` | `~~~Token~~~` (literal) | `${mule.connected.app.token}` | `MULE_CONNECTED_APP_TOKEN` |

---

## 7. Complete Maven Build Command Reference

```bash
mvn clean package \
  -B \
  -s medica-mule-assets/settings.xml \
  -DMule.EE.User="${MULE_EE_USER}" \
  -DMule.EE.Password="${MULE_EE_PASSWORD}" \
  -Dnexus.medica.mule.internal="${NEXUS_MEDICA_MULE_INTERNAL}" \
  -Dmule.connected.app.token="${MULE_CONNECTED_APP_TOKEN}" \
  -DADMIN_API_CLIENT_ID="${ADMIN_API_CLIENT_ID}" \
  -DADMIN_API_CLIENT_SECRET="${ADMIN_API_CLIENT_SECRET}" \
  -DADMIN_API_HOST="${ADMIN_API_HOST}" \
  -DADMIN_TOKEN_URL="${ADMIN_TOKEN_URL}" \
  -Denv="${DEPLOY_ENV}" \
  -DskipAST \
  -DdeploymentInstance="${INSTANCE_NAME}" \
  -DAPI_Type="${API_TYPE}" \
  -DrepoName="${REPO_NAME}"
```

---

## 8. Non-Secret Pipeline Variables (Computed at Runtime)

These are not secrets — they are computed or set by the pipeline itself.

| Variable | Source | Description |
|----------|--------|-------------|
| `Deploy.Env` | Pipeline variable | Environment name e.g. `CIDEV`, `CIQA`, `CISTAGE` |
| `instanceName` | `configLookup.yml` | Deployment instance name read from `configs.txt` |
| `API_TYPE` | Detected at runtime | `REST API Detected!` or `Async API Detected!` — detected by searching for `http:listener` in source |
| `build.repository.name` | ADO built-in | Repository name — set automatically |
| `buildOptions` | Pipeline variable | Dynamic flags accumulated during pipeline run |
| `customLog4j` | Pipeline variable | Controls whether default `log4j2.xml` is copied |

---

## 9. Summary — Total Secrets Required for Build

| # | Secret | Scope | Source Variable Group |
|---|--------|-------|-----------------------|
| 1 | `MULE_EE_USER` | Org-level | `onp-deployment` |
| 2 | `MULE_EE_PASSWORD` | Org-level | `onp-deployment` |
| 3 | `NEXUS_MEDICA_MULE_INTERNAL` | Org-level | `onp-deployment` |
| 4 | `MULE_CONNECTED_APP_TOKEN` | Org-level | `onp-deployment` |
| 5 | `ADMIN_API_CLIENT_ID_DEV` | Environment | `PumaPropsProv` |
| 6 | `ADMIN_API_CLIENT_SECRET_DEV` | Environment | `PumaPropsProv` |
| 7 | `ADMIN_API_HOST_DEV` | Environment | `PumaPropsProv` |
| 8 | `ADMIN_TOKEN_URL_NONPROD` | Environment | `PumaPropsProv` |
| 9 | `ADMIN_API_CLIENT_ID_STG` | Environment | `PumaPropsProv` |
| 10 | `ADMIN_API_CLIENT_SECRET_STG` | Environment | `PumaPropsProv` |
| 11 | `ADMIN_API_HOST_STG` | Environment | `PumaPropsProv` |
| 12 | `ADMIN_API_CLIENT_ID_PROD` | Environment | `PumaPropsProv` |
| 13 | `ADMIN_API_CLIENT_SECRET_PROD` | Environment | `PumaPropsProv` |
| 14 | `ADMIN_API_HOST_PROD` | Environment | `PumaPropsProv` |
| 15 | `ADMIN_TOKEN_URL_PROD` | Environment | `PumaPropsProv` |
| 16 | `SONAR_TOKEN` | Org-level | Rotate existing — new token required |

**Total: 16 secrets**

---

## 10. Open Items / Confirmations Required

| # | Item | Owner |
|---|------|-------|
| 1 | Confirm actual ADO Library name for `PumaPropsProv` variable group | Client |
| 2 | Rotate hardcoded SonarQube token before migration | Client / Security team |
| 3 | Confirm `medica-mule-assets` migration to GitHub Enterprise is on plan | Migration team |
| 4 | Confirm self-hosted runner registration in GitHub Enterprise org | Client infra team |
| 5 | Confirm runner network access to `nexus.medica.com` | Client networking team |
| 6 | Confirm `java17flag` source — not found in reviewed pipeline files | Client |
