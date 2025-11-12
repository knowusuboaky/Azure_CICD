# 🚀 Deploy Two Azure Web Apps Using Only `azure.yaml`

Deploy both **website** (frontend) and **fastapi** (backend) apps to **Azure App Service**
with a CI/CD flow that promotes from **Staging → Production (after reviewer approval)**.

---

## ⚙️ Environments Summary

| Environment    | URL examples                                                                       | Reviewer Required         | GitHub Environment |
| -------------- | ---------------------------------------------------------------------------------- | ------------------------- | ------------------ |
| **Staging**    | `https://website-stg.azurewebsites.net`, `https://fastapi-stg.azurewebsites.net`   | ❌ No                      | ✅ `Staging`        |
| **Production** | `https://website-prod.azurewebsites.net`, `https://fastapi-prod.azurewebsites.net` | ✅ Yes (reviewers approve) | ✅ `Prod`           |

---

## 🧩 Step 1 – Initialize & Create Environments

```bash
azd init

# Permanent envs (match your GitHub Environments)
azd env new website-stg
azd env new website-prod
azd env new fastapi-stg
azd env new fastapi-prod
```

If you already have App Service Plans:

```bash
azd env set AZURE_APPSERVICE_PLAN_NAME <plan-name>
```

---

## 🧾 Step 2 – `azure.yaml` (with inline comments)

```yaml
# azure.yaml
name: ai-platform

services:
  website:
    project: ./website
    language: js
    host: appservice
    startCommand: npx serve -s build -l $PORT
    healthCheckPath: /healthz
    appSettings:
      # Example (STAGING):    https://fastapi-stg.azurewebsites.net
      # Example (PRODUCTION): https://fastapi-prod.azurewebsites.net
      - name: API_BASE_URL
        value: ${API_BASE_URL}

      # Example (STAGING):    https://website-stg.azurewebsites.net
      # Example (PRODUCTION): https://website-prod.azurewebsites.net
      - name: WEBSITE_BASE_URL
        value: ${WEBSITE_BASE_URL}

    # Set via: azd env set webAppName <name>
    # STAGING → website-stg, PRODUCTION → website-prod
    name: ${webAppName}

  fastapi:
    project: ./api
    language: python
    host: appservice
    runtime: python:3.11
    startCommand: |
      pip install -r requirements.txt && \
      gunicorn -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:$PORT app:app
    healthCheckPath: /healthz
    appSettings:
      - name: COSMOS_DB_ENDPOINT
        value: ${COSMOS_DB_ENDPOINT}
      - name: COSMOS_DB_KEY
        value: ${COSMOS_DB_KEY}
      - name: REDIS_URL
        value: ${REDIS_URL}
      - name: OPENAI_ENDPOINT
        value: ${OPENAI_ENDPOINT}
      - name: OPENAI_API_KEY
        value: ${OPENAI_API_KEY}

      # Example (STAGING):    https://website-stg.azurewebsites.net
      # Example (PRODUCTION): https://website-prod.azurewebsites.net
      - name: WEBSITE_BASE_URL
        value: ${WEBSITE_BASE_URL}

    # Set via: azd env set apiAppName <name>
    # STAGING → fastapi-stg, PRODUCTION → fastapi-prod
    name: ${apiAppName}
```

---

## 🚀 Step 3 – Provision & Deploy (first run)

```bash
# Website
azd env select website-stg  && azd env set webAppName website-stg  && azd up
azd env select website-prod && azd env set webAppName website-prod && azd up

# FastAPI
azd env select fastapi-stg  && azd env set apiAppName fastapi-stg  && azd up
azd env select fastapi-prod && azd env set apiAppName fastapi-prod && azd up
```

---

## 🧠 Step 4 – GitHub Actions Workflow (`.github/workflows/publish.yml`)

```yaml
name: Publish

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      releaseType:
        description: 'Where to release (staging or prod)?'
        required: true
        default: 'prod'

env:
  WEBSITE_STG_APP: website-stg
  FASTAPI_STG_APP: fastapi-stg
  WEBSITE_PROD_APP: website-prod
  FASTAPI_PROD_APP: fastapi-prod
  AZ_RG_STG: ${{ vars.AZ_RG_STG }}
  AZ_RG_PROD: ${{ vars.AZ_RG_PROD }}

jobs:
  staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: Staging
      url: https://website-stg.azurewebsites.net/
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node & Build Website
        uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: |
          cd website
          npm ci && npm run build

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Configure WEBSITE app settings
        run: |
          az webapp config appsettings set \
            -g "${AZ_RG_STG}" -n "${WEBSITE_STG_APP}" \
            --settings \
              API_BASE_URL="https://${{ env.FASTAPI_STG_APP }}.azurewebsites.net" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_STG_APP }}.azurewebsites.net"

      - name: Deploy WEBSITE
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.WEBSITE_STG_APP }}
          publish-profile: ${{ secrets.WEBSITE_STG_PUBLISH_PROFILE }}
          package: website

      # ---------- FASTAPI ----------
      - name: Setup Python
        uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: |
          cd api
          python -m pip install --upgrade pip
          mkdir -p package
          pip install -r requirements.txt -t package
          cp -r . package
          cd package && zip -r ../fastapi.zip . && cd ..
      - name: Configure FASTAPI app settings
        run: |
          az webapp config appsettings set \
            -g "${AZ_RG_STG}" -n "${FASTAPI_STG_APP}" \
            --settings \
              COSMOS_DB_ENDPOINT="${{ secrets.STG_COSMOS_DB_ENDPOINT }}" \
              COSMOS_DB_KEY="${{ secrets.STG_COSMOS_DB_KEY }}" \
              REDIS_URL="${{ secrets.STG_REDIS_URL }}" \
              OPENAI_ENDPOINT="${{ secrets.STG_OPENAI_ENDPOINT }}" \
              OPENAI_API_KEY="${{ secrets.STG_OPENAI_API_KEY }}" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_STG_APP }}.azurewebsites.net"

          az webapp config set \
            -g "${AZ_RG_STG}" -n "${FASTAPI_STG_APP}" \
            --startup-file "gunicorn -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:\$PORT app:app"

      - name: Deploy FASTAPI
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.FASTAPI_STG_APP }}
          publish-profile: ${{ secrets.FASTAPI_STG_PUBLISH_PROFILE }}
          package: api/fastapi.zip

  prod:
    name: Deploy to Production
    needs: [staging]
    runs-on: ubuntu-latest
    environment:
      name: Prod   # Reviewer approval required
      url: https://website-prod.azurewebsites.net/
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node & Build Website
        uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: |
          cd website
          npm ci && npm run build

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Configure WEBSITE app settings (Prod)
        run: |
          az webapp config appsettings set \
            -g "${AZ_RG_PROD}" -n "${WEBSITE_PROD_APP}" \
            --settings \
              API_BASE_URL="https://${{ env.FASTAPI_PROD_APP }}.azurewebsites.net" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_PROD_APP }}.azurewebsites.net"

      - name: Deploy WEBSITE
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.WEBSITE_PROD_APP }}
          publish-profile: ${{ secrets.WEBSITE_PROD_PUBLISH_PROFILE }}
          package: website

      # ---------- FASTAPI ----------
      - name: Setup Python
        uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: |
          cd api
          python -m pip install --upgrade pip
          mkdir -p package
          pip install -r requirements.txt -t package
          cp -r . package
          cd package && zip -r ../fastapi.zip . && cd ..
      - name: Configure FASTAPI app settings (Prod)
        run: |
          az webapp config appsettings set \
            -g "${AZ_RG_PROD}" -n "${FASTAPI_PROD_APP}" \
            --settings \
              COSMOS_DB_ENDPOINT="${{ secrets.PROD_COSMOS_DB_ENDPOINT }}" \
              COSMOS_DB_KEY="${{ secrets.PROD_COSMOS_DB_KEY }}" \
              REDIS_URL="${{ secrets.PROD_REDIS_URL }}" \
              OPENAI_ENDPOINT="${{ secrets.PROD_OPENAI_ENDPOINT }}" \
              OPENAI_API_KEY="${{ secrets.PROD_OPENAI_API_KEY }}" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_PROD_APP }}.azurewebsites.net"

          az webapp config set \
            -g "${AZ_RG_PROD}" -n "${FASTAPI_PROD_APP}" \
            --startup-file "gunicorn -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:\$PORT app:app"

      - name: Deploy FASTAPI
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.FASTAPI_PROD_APP }}
          publish-profile: ${{ secrets.FASTAPI_PROD_PUBLISH_PROFILE }}
          package: api/fastapi.zip
```

---

## 🔐 Step 5 – GitHub Secrets & Variables

### ➤ Environment: **Staging**

**Settings → Environments → Staging → Secrets**

```
WEBSITE_STG_PUBLISH_PROFILE
FASTAPI_STG_PUBLISH_PROFILE
STG_COSMOS_DB_ENDPOINT
STG_COSMOS_DB_KEY
STG_REDIS_URL
STG_OPENAI_ENDPOINT
STG_OPENAI_API_KEY
```

### ➤ Environment: **Prod**

**Settings → Environments → Prod → Secrets**

```
WEBSITE_PROD_PUBLISH_PROFILE
FASTAPI_PROD_PUBLISH_PROFILE
PROD_COSMOS_DB_ENDPOINT
PROD_COSMOS_DB_KEY
PROD_REDIS_URL
PROD_OPENAI_ENDPOINT
PROD_OPENAI_API_KEY
```

### ➤ Repository-level **Secrets**

**Settings → Secrets and variables → Actions → Secrets**

```
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
```

### ➤ Repository-level **Variables**

**Settings → Secrets and variables → Actions → Variables**

```
AZ_RG_STG   = rg-ai-staging
AZ_RG_PROD  = rg-ai-production
```

---

✅ **Result**

* Push a release → Deploys automatically to **Staging**.
* Reviewer approves **Prod** → Deploys both apps to production.
* All app settings, secrets, and resource groups are cleanly separated by environment.

This setup gives you a **secure, reviewer-gated CI/CD pipeline** for dual App Services using only `azure.yaml` + GitHub Actions.
