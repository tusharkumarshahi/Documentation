# Deployment Requirements — MuleSoft GitHub Actions Migration

> **Scope:** All environments (Dev, QA, Stage, Perf, Prod)
> **Pipeline files reviewed:** `cicd-dev-ch2.yml`, `cicd-stage-ch2.yml`, `cicd-prod-ch2.yml`
> **Purpose:** All secrets, credentials, and infrastructure requirements needed to execute the Exchange Publish and CloudHub 2.0 Deploy steps in GitHub Actions.

---

## 1. Overview

Deployment consists of two sequential steps:

```
Step 1 — Publish to Anypoint Exchange
  → Publishes the built .jar to Exchange
  → Required before CloudHub 2.0 deploy
  → Uses: mvn clean deploy

Step 2 — Deploy to CloudHub 2.0
  → Deploys the published artifact to CloudHub 2.0
  → Uses: mvn clean deploy with -DmuleDeploy flag
  → Requires Exchange publish to have succeeded
```

Both steps run `mvn clean deploy` but with different flags. The deploy step adds `-DmuleDeploy` which triggers the Mule deployment plugin.

---

## 2. Environment Overview

| Environment | Trigger Branch | Auto-Promote To | Approval Required |
|-------------|---------------|-----------------|-------------------|
| Dev (CIDEV) | `develop-ch2` | QA (CIQA) | No |
| QA (CIQA) | Auto from Dev | — | No |
| Stage (CISTAGE) | `release-ch2` | Perf (CIPERF) | No |
| Perf (CIPERF) | Auto from Stage | — | No |
| Prod (CIPROD) | `main-ch2` | — | Yes — approval gated |

---

## 3. Secrets Required — Exchange Publish

### 3.1 Shared Across Dev, QA, Stage, Perf

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|-------------|
| `MULE_EE_USER` | `MuleEEUser` | `onp-deployment` | MuleSoft EE username |
| `MULE_EE_PASSWORD` | `MuleEEPassword` | `onp-deployment` | MuleSoft EE password |
| `NEXUS_MEDICA_MULE_INTERNAL` | `nexusMedicaMuleInternal` | `onp-deployment` | Nexus internal repo password |
| `MULE_CONNECTED_APP_TOKEN` | `mule.connected.app.token` | `onp-deployment` | Anypoint Connected App token |
| `CH2_CLIENT_ID` | `ch2.clientID` | `cloudhub-deployment` | CloudHub 2.0 Connected App client ID — non-prod |
| `CH2_CLIENT_SECRET` | `ch2.clientSecret` | `cloudhub-deployment` | CloudHub 2.0 Connected App client secret — non-prod |
| `EXCHANGE_DEPLOYMENT_URL` | `exchangeDeploymentUrl` | `cloudhub-deployment` | Anypoint Exchange deployment repository URL |

### 3.2 Prod Exchange Publish (Different Credentials)

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|-------------|
| `CH2_CLIENT_ID_PROD` | `ch2.clientID-Prod` | `cloudhub-deployment` | CloudHub 2.0 Connected App client ID — prod only |
| `CH2_CLIENT_SECRET_PROD` | `ch2.clientSecret-Prod` | `cloudhub-deployment` | CloudHub 2.0 Connected App client secret — prod only |

> **Note:** Prod Exchange publish uses separate `ch2.clientID-Prod` credentials — not the same as non-prod.

---

## 4. Secrets Required — CloudHub 2.0 Deploy

### 4.1 Shared Across All Environments

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|-------------|
| `MULE_EE_USER` | `MuleEEUser` | `onp-deployment` | MuleSoft EE username |
| `MULE_EE_PASSWORD` | `MuleEEPassword` | `onp-deployment` | MuleSoft EE password |
| `NEXUS_MEDICA_MULE_INTERNAL` | `nexusMedicaMuleInternal` | `onp-deployment` | Nexus internal repo password |
| `MULE_CONNECTED_APP_TOKEN` | `mule.connected.app.token` | `onp-deployment` | Anypoint Connected App token |
| `BUSINESS_GROUP_CLIENT_ID` | `BusinessGroup.ClientId` | `Mulesoft-Cloudhub2.0-deployment` | Business group client ID for Anypoint platform auth |
| `BUSINESS_GROUP_CLIENT_SECRET` | `BusinessGroup.ClientSecret` | `Mulesoft-Cloudhub2.0-deployment` | Business group client secret |
| `BUSINESS_GROUP_ID` | `M_H_I_2-BusinessGroup.id` | `Mulesoft-Cloudhub2.0-deployment` | Anypoint business group ID |
| `ANYPOINT_CLIENT_ID` | `M_H_I_2-BusinessGroup.clientID` | `Mulesoft-Cloudhub2.0-deployment` | Anypoint platform client ID |
| `ANYPOINT_CLIENT_SECRET` | `M_H_I_2-BusinessGroup.clientSecret` | `Mulesoft-Cloudhub2.0-deployment` | Anypoint platform client secret |
| `ENCRYPTION_KEY` | `Encryption.Key.Dev` | `Mulesoft-Cloudhub2.0-deployment` | Application encryption key |
| `PUMA_GETTOKEN_PASSWORD` | `Puma.getToken.password` | `PumaPropsProv` | Puma token retrieval password |
| `SPLUNK_SOURCE_TYPE_CH2` | `splunkSourceType-CH2` | `Mulesoft-Cloudhub2.0-deployment` | Splunk source type — shared across all envs |

> ⚠️ **Important:** `Encryption.Key.Dev` is used in **all environments** including Stage and Prod. There is no separate `Encryption.Key.Stage` or `Encryption.Key.Prod`. This should be confirmed with the client as it may be intentional or a configuration gap.

### 4.2 Environment-Specific Secrets — CloudHub Deploy

#### Dev / QA

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|----|
| `CH2_CLIENT_ID_DEV` | `ch2.clientID` | `cloudhub-deployment` | Connected App client ID |
| `CH2_CLIENT_SECRET_DEV` | `ch2.clientSecret` | `cloudhub-deployment` | Connected App client secret |
| `ADMIN_API_CLIENT_ID_DEV` | `PumaPropsProv.SysMuleAdmin.ClientID.QA` | `PumaPropsProv` | Admin API client ID — QA credential used for Dev |
| `ADMIN_API_CLIENT_SECRET_DEV` | `PumaPropsProv.SysMuleAdmin.ClientSecret.QA` | `PumaPropsProv` | Admin API client secret |
| `ADMIN_API_HOST_DEV` | `PumaPropsProv.SysMuleAdmin.URL.DEV` | `PumaPropsProv` | Admin API host URL |
| `ADMIN_TOKEN_URL_NONPROD` | `PumaPropsProv.Token.URL.NonProd` | `PumaPropsProv` | Token endpoint — shared with Stage |
| `SPLUNK_HOST_DEV` | `splunkHost-CH2-Dev` | `Mulesoft-Cloudhub2.0-deployment` | Splunk host |
| `SPLUNK_URL_DEV` | `splunkURL-CH2-Dev` | `Mulesoft-Cloudhub2.0-deployment` | Splunk URL |
| `SPLUNK_TOKEN_DEV` | `splunkToken-CH2-Dev` | `Mulesoft-Cloudhub2.0-deployment` | Splunk auth token |
| `SPLUNK_INDEX_DEV` | `splunkIndex-CH2-Dev` | `Mulesoft-Cloudhub2.0-deployment` | Splunk index |
| `SPLUNK_SOURCE_DEV` | `splunkSource-CH2-Dev` | `Mulesoft-Cloudhub2.0-deployment` | Splunk source |
| `CUSTOM_DOMAIN_DEV` | `customDomain` | `cloudhub-deployment` | Custom DNS domain |
| `DEPLOY_TARGET_DEV` | `Deploy.Target` | `cloudhub-deployment` | CloudHub 2.0 deployment target / cluster |

#### Stage / Perf

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|----|
| `CH2_CLIENT_ID_STG` | `ch2.clientID` | `cloudhub-deployment` | Connected App client ID |
| `CH2_CLIENT_SECRET_STG` | `ch2.clientSecret` | `cloudhub-deployment` | Connected App client secret |
| `ADMIN_API_CLIENT_ID_STG` | `PumaPropsProv.SysMuleAdmin.ClientID.STG` | `PumaPropsProv` | Admin API client ID |
| `ADMIN_API_CLIENT_SECRET_STG` | `PumaPropsProv.SysMuleAdmin.ClientSecret.STG` | `PumaPropsProv` | Admin API client secret |
| `ADMIN_API_HOST_STG` | `PumaPropsProv.SysMuleAdmin.URL.STG` | `PumaPropsProv` | Admin API host URL |
| `SPLUNK_HOST_STG` | `splunkHost-CH2-Stage` | `Mulesoft-Cloudhub2.0-deployment` | Splunk host |
| `SPLUNK_URL_STG` | `splunkURL-CH2-Stage` | `Mulesoft-Cloudhub2.0-deployment` | Splunk URL |
| `SPLUNK_TOKEN_STG` | `splunkToken-CH2-Stage` | `Mulesoft-Cloudhub2.0-deployment` | Splunk auth token |
| `SPLUNK_INDEX_STG` | `splunkIndex-CH2-Stage` | `Mulesoft-Cloudhub2.0-deployment` | Splunk index |
| `SPLUNK_SOURCE_STG` | `splunkSource-CH2-Stage` | `Mulesoft-Cloudhub2.0-deployment` | Splunk source |
| `CUSTOM_DOMAIN_STG` | `customDomain` | `cloudhub-deployment` | Custom DNS domain |
| `DEPLOY_TARGET_STG` | `Deploy.Target` | `cloudhub-deployment` | CloudHub 2.0 deployment target / cluster |

#### Prod

| GitHub Secret Name | ADO Variable | ADO Variable Group | Description |
|-------------------|-------------|-------------------|----|
| `CH2_CLIENT_ID_PROD` | `ch2.clientID-Prod` | `cloudhub-deployment` | Connected App client ID — prod specific |
| `CH2_CLIENT_SECRET_PROD` | `ch2.clientSecret-Prod` | `cloudhub-deployment` | Connected App client secret — prod specific |
| `ADMIN_API_CLIENT_ID_PROD` | `PumaPropsProv.SysMuleAdmin.ClientID.PROD` | `PumaPropsProv` | Admin API client ID |
| `ADMIN_API_CLIENT_SECRET_PROD` | `PumaPropsProv.SysMuleAdmin.ClientSecret.PROD` | `PumaPropsProv` | Admin API client secret |
| `ADMIN_API_HOST_PROD` | `PumaPropsProv.SysMuleAdmin.URL.PROD` | `PumaPropsProv` | Admin API host URL |
| `ADMIN_TOKEN_URL_PROD` | `PumaPropsProv.Token.URL.Prod` | `PumaPropsProv` | Token endpoint — prod specific |
| `SPLUNK_HOST_PROD` | `splunkHost-CH2-Prod` | `Mulesoft-Cloudhub2.0-deployment` | Splunk host |
| `SPLUNK_URL_PROD` | `splunkURL-CH2-Prod` | `Mulesoft-Cloudhub2.0-deployment` | Splunk URL |
| `SPLUNK_TOKEN_PROD` | `splunkToken-CH2-Prod` | `Mulesoft-Cloudhub2.0-deployment` | Splunk auth token |
| `SPLUNK_INDEX_PROD` | `splunkIndex-CH2-Prod` | `Mulesoft-Cloudhub2.0-deployment` | Splunk index |
| `SPLUNK_SOURCE_PROD` | `splunkSource-CH2-Prod` | `Mulesoft-Cloudhub2.0-deployment` | Splunk source |
| `CUSTOM_DOMAIN_PROD` | `customDomain` | `cloudhub-deployment` | Custom DNS domain |
| `DEPLOY_TARGET_PROD` | `Deploy.Target` | `cloudhub-deployment` | CloudHub 2.0 deployment target / cluster |

---

## 5. ADO Variable Groups to Export

Request all of the following variable groups in full from the client:

| Variable Group | Used In |
|---------------|---------|
| `onp-deployment` | Build + Deploy — all envs |
| `cloudhub-deployment` | Deploy — all envs |
| `Mulesoft-Cloudhub2.0-deployment` | Deploy — all envs |
| `PumaPropsProv` | Build + Deploy — all envs (confirm actual group name in ADO Library) |

---

## 6. Complete Maven Command Reference

### Exchange Publish — Non-Prod (Dev, QA, Stage, Perf)

```bash
mvn clean deploy \
  -U \
  -B \
  -s medica-mule-assets/settings.xml \
  -DMule.EE.User="${MULE_EE_USER}" \
  -DMule.EE.Password="${MULE_EE_PASSWORD}" \
  -Dnexus.medica.mule.internal="${NEXUS_MEDICA_MULE_INTERNAL}" \
  -DConnectedApp.ID="${CH2_CLIENT_ID}" \
  -DConnectedApp.Secret="${CH2_CLIENT_SECRET}" \
  -DaltDeploymentRepository="${EXCHANGE_DEPLOYMENT_URL}" \
  -Dmule.connected.app.token="${MULE_CONNECTED_APP_TOKEN}" \
  -DskipTests \
  -DskipAST \
  -DdeploymentInstance="${INSTANCE_NAME}" \
  -DAPI_Type="${API_TYPE}" \
  -DrepoName="${REPO_NAME}"
```

### Exchange Publish — Prod

Same as above but:
```bash
  -DConnectedApp.ID="${CH2_CLIENT_ID_PROD}" \
  -DConnectedApp.Secret="${CH2_CLIENT_SECRET_PROD}" \
```

### CloudHub 2.0 Deploy — Non-Prod

```bash
mvn clean deploy \
  -U \
  -B \
  -s medica-mule-assets/settings.xml \
  -DConnectedApp.ID="${CH2_CLIENT_ID}" \
  -DConnectedApp.Secret="${CH2_CLIENT_SECRET}" \
  -DMule.EE.User="${MULE_EE_USER}" \
  -DMule.EE.Password="${MULE_EE_PASSWORD}" \
  -Dnexus.medica.mule.internal="${NEXUS_MEDICA_MULE_INTERNAL}" \
  -DmuleDeploy \
  -DskipTests \
  -Dmule.connected.app.token="${MULE_CONNECTED_APP_TOKEN}" \
  -Dmule.clientId="${BUSINESS_GROUP_CLIENT_ID}" \
  -Dmule.clientSecret="${BUSINESS_GROUP_CLIENT_SECRET}" \
  -DADMIN_API_CLIENT_ID="${ADMIN_API_CLIENT_ID}" \
  -DADMIN_API_CLIENT_SECRET="${ADMIN_API_CLIENT_SECRET}" \
  -DADMIN_API_HOST="${ADMIN_API_HOST}" \
  -DADMIN_TOKEN_URL="${ADMIN_TOKEN_URL}" \
  -Denv="${DEPLOY_ENV}" \
  -Dencryption.key="${ENCRYPTION_KEY}" \
  -DDeploy.Target="${DEPLOY_TARGET}" \
  -Danypoint.platform.client_id="${ANYPOINT_CLIENT_ID}" \
  -Danypoint.platform.client_secret="${ANYPOINT_CLIENT_SECRET}" \
  -DDeploy.businessGrpID="${BUSINESS_GROUP_ID}" \
  -DskipAST \
  -DsplunkHost-CH2="${SPLUNK_HOST}" \
  -DsplunkURL-CH2="${SPLUNK_URL}" \
  -DsplunkToken-CH2="${SPLUNK_TOKEN}" \
  -DsplunkIndex-CH2="${SPLUNK_INDEX}" \
  -DsplunkSource-CH2="${SPLUNK_SOURCE}" \
  -DsplunkSourceType-CH2="${SPLUNK_SOURCE_TYPE}" \
  -DcustomDomain="${CUSTOM_DOMAIN}" \
  -DdeploymentInstance="${INSTANCE_NAME}" \
  -DrepoName="${REPO_NAME}" \
  -DPuma.getToken.password="${PUMA_GETTOKEN_PASSWORD}" \
  -DAPI_Type="${API_TYPE}"
```

### CloudHub 2.0 Deploy — Prod

Same as above but:
```bash
  -DConnectedApp.ID="${CH2_CLIENT_ID_PROD}" \
  -DConnectedApp.Secret="${CH2_CLIENT_SECRET_PROD}" \
```

---

## 7. Non-Secret Pipeline Variables (Computed at Runtime)

| Variable | Source | Description |
|----------|--------|-------------|
| `Deploy.Env` | GitHub environment name | `CIDEV`, `CIQA`, `CISTAGE`, `CIPERF`, `CIPROD` |
| `Deploy.Target` | Variable group | CloudHub 2.0 cluster/target per environment |
| `instanceName` | `configLookup.yml` | Read from `medica-mule-config` repo `configs.txt` |
| `API_TYPE` | Detected at runtime | `REST API Detected!` or `Async API Detected!` |
| `exchangeDeploymentUrl` | Variable group | Exchange repository URL for artifact publishing |
| `customDomain` | Variable group | Custom DNS domain for the deployed API |
| `buildOptions` / `deployOptions` | Accumulated pipeline vars | Dynamic flags set during pipeline run |
| `sysMuleAdminOptions` | Dev pipeline only | Additional admin options — Dev environment only |
| `build.repository.name` | GitHub built-in | Repository name — set automatically |

---

## 8. Prod-Specific Requirements

Prod has additional steps and requirements not present in lower environments:

| Requirement | Detail |
|-------------|--------|
| Approval gate | Manual approval required before prod deploy — approvers group must be configured in GitHub Environments |
| Separate CH2 credentials | `ch2.clientID-Prod` and `ch2.clientSecret-Prod` — separate from non-prod |
| Prod Admin API credentials | `PumaPropsProv.SysMuleAdmin.ClientID.PROD` — separate set |
| Prod token URL | `PumaPropsProv.Token.URL.Prod` — different endpoint from non-prod |
| Anypoint access token fetch | `fetch_access_token.py` runs with prod credentials before build |
| Dynatrace integration | Post-deploy Dynatrace registration — prod only |
| Anypoint alerts | Post-deploy alert configuration — prod only |

---

## 9. GitHub Environments Configuration

GitHub Environments must be created in the API repo to manage environment-specific secrets and approval gates:

| GitHub Environment | Maps To | Protection Rules |
|-------------------|---------|-----------------|
| `CIDEV` | Dev | None |
| `CIQA` | QA | None |
| `CISTAGE` | Stage | None |
| `CIPERF` | Perf | None |
| `CIPROD` | Prod | Required reviewers — approval gate |

---

## 10. Summary — Total Secrets Required for Deployment

| Category | Count |
|----------|-------|
| Org-level shared secrets (build + deploy) | 4 |
| CloudHub Connected App credentials (non-prod) | 2 |
| CloudHub Connected App credentials (prod) | 2 |
| Business Group / Anypoint credentials | 4 |
| Exchange deployment URL | 1 |
| Encryption key | 1 |
| Puma token password | 1 |
| Admin API credentials — Dev/QA | 4 |
| Admin API credentials — Stage/Perf | 4 |
| Admin API credentials — Prod | 4 |
| Splunk credentials — Dev | 5 |
| Splunk credentials — Stage | 5 |
| Splunk credentials — Prod | 5 |
| Splunk source type (shared) | 1 |
| Custom domain per env | 3 |
| Deploy target per env | 3 |
| **Total deploy-specific secrets** | **49** |

---

## 11. Open Items / Confirmations Required

| # | Item | Owner |
|---|------|-------|
| 1 | Confirm actual ADO Library name for `PumaPropsProv` variable group | Client |
| 2 | Confirm `Encryption.Key.Dev` is intentionally used for Stage and Prod or if separate keys exist | Client |
| 3 | Confirm `customDomain` value per environment | Client |
| 4 | Confirm `Deploy.Target` value per environment (cluster ID) | Client |
| 5 | Confirm `exchangeDeploymentUrl` value | Client |
| 6 | Confirm approvers for prod deployment gate in GitHub Enterprise | Client |
| 7 | Confirm Dynatrace and Anypoint alert configuration for prod post-deploy steps | Client |
| 8 | Confirm `sysMuleAdminOptions` usage — appears in Dev pipeline only | Client |
| 9 | Confirm QA and Perf environments — are their secrets identical to Dev and Stage respectively | Client |
| 10 | Confirm `Puma.getToken.password` variable group location | Client |
