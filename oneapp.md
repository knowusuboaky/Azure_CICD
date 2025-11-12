# 🚀 Deploy **Website + FastAPI** with App Service **Slots** (Zero-downtime)

## Architecture

* **Apps**

  * `website` (front-end) → slots: `production`, `staging`
  * `fastapi` (backend) → slots: `production`, `staging`
* **Plan tier**: use **Standard/Premium/Isolated** (slots + traffic routing require this)

> Your existing `azure.yaml` stays simple; slots are managed in the workflow.

---

## 1) `azure.yaml` (both services)

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
      # Staging example:    https://fastapi-staging.azurewebsites.net
      # Production example: https://fastapi.azurewebsites.net
      - name: API_BASE_URL
        value: ${API_BASE_URL}

      # Example: points to the LIVE host
      - name: WEBSITE_BASE_URL
        value: ${WEBSITE_BASE_URL}

    # Parent app name (no slot). e.g. azd env set webAppName website
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

      # Example (staging slot uses website-staging host)
      - name: WEBSITE_BASE_URL
        value: ${WEBSITE_BASE_URL}

    # Parent app name (no slot). e.g. azd env set apiAppName fastapi
    name: ${apiAppName}
```

---

## 2) One-time provision (creates parent apps; slots come later)

```bash
# Create parent apps (production slot is implicit)
azd env select website-prod && azd env set webAppName website && azd up
azd env select fastapi-prod && azd env set apiAppName fastapi && azd up
```

> Ensure the App Service Plans are **Standard/Premium+**.

---

## 3) GitHub Actions — Slots workflow

* Deploy both apps to their **staging** slots.
* After **Prod** approval: choose **ramp** (set traffic % to staging) or **swap** (staging → production).
* Independent ramp % for **website** and **fastapi**.

Save as **`.github/workflows/publish-slots.yml`**:

```yaml
name: Publish (Slots: staging → production)

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      mode:
        description: 'Promotion mode: ramp (canary) or swap (full)'
        required: true
        default: 'ramp'
        type: choice
        options: [ramp, swap]
      rampPercentWebsite:
        description: 'Traffic % to send to WEBSITE staging (0-100)'
        required: true
        default: '20'
      rampPercentApi:
        description: 'Traffic % to send to FASTAPI staging (0-100)'
        required: true
        default: '20'

env:
  # Parent app names (NO slot suffix)
  WEBSITE_APP: website
  FASTAPI_APP: fastapi

  # Resource groups (set as repo Variables or Secrets)
  WEBSITE_RG: ${{ vars.AZ_RG_WEBSITE }}
  FASTAPI_RG: ${{ vars.AZ_RG_FASTAPI }}

jobs:
  # ==================== DEPLOY TO STAGING SLOTS ====================
  stage_deploy:
    name: Deploy website+fastapi → staging slots
    runs-on: ubuntu-latest
    environment:
      name: Staging
      url: https://website-staging.azurewebsites.net/
    steps:
      - uses: actions/checkout@v4

      # ---------- WEBSITE build ----------
      - name: Setup Node & Build (website)
        uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: |
          cd website
          npm ci
          npm run build

      # ---------- Azure login ----------
      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # ---------- Ensure slots exist (idempotent) ----------
      - name: Ensure WEBSITE staging slot exists
        run: |
          if ! az webapp deployment slot list -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" --query "[?name=='staging']" -o tsv | grep -q staging; then
            az webapp deployment slot create -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" --slot staging --configuration-source "${WEBSITE_APP}"
          fi
      - name: Ensure FASTAPI staging slot exists
        run: |
          if ! az webapp deployment slot list -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" --query "[?name=='staging']" -o tsv | grep -q staging; then
            az webapp deployment slot create -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" --slot staging --configuration-source "${FASTAPI_APP}"
          fi

      # ---------- Configure slot app settings (STAGING) ----------
      - name: Configure WEBSITE (staging) app settings
        run: |
          az webapp config appsettings set -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" -s staging \
            --settings \
              API_BASE_URL="https://${{ env.FASTAPI_APP }}-staging.azurewebsites.net" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_APP }}-staging.azurewebsites.net"

      - name: Configure FASTAPI (staging) app settings
        run: |
          az webapp config appsettings set -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" -s staging \
            --settings \
              COSMOS_DB_ENDPOINT="${{ secrets.STG_COSMOS_DB_ENDPOINT }}" \
              COSMOS_DB_KEY="${{ secrets.STG_COSMOS_DB_KEY }}" \
              REDIS_URL="${{ secrets.STG_REDIS_URL }}" \
              OPENAI_ENDPOINT="${{ secrets.STG_OPENAI_ENDPOINT }}" \
              OPENAI_API_KEY="${{ secrets.STG_OPENAI_API_KEY }}" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_APP }}-staging.azurewebsites.net"
          az webapp config set -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" -s staging \
            --startup-file "gunicorn -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:\$PORT app:app"

      # ---------- Deploy to STAGING slots ----------
      - name: Deploy WEBSITE → staging slot
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.WEBSITE_APP }}
          slot-name: staging
          publish-profile: ${{ secrets.WEBSITE_STAGING_SLOT_PUBLISH_PROFILE }}
          package: website

      - name: Package FASTAPI
        uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: |
          cd api
          python -m pip install --upgrade pip
          mkdir -p package
          pip install -r requirements.txt -t package
          cp -r . package
          cd package && zip -r ../fastapi.zip . && cd ..
      - name: Deploy FASTAPI → staging slot
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.FASTAPI_APP }}
          slot-name: staging
          publish-profile: ${{ secrets.FASTAPI_STAGING_SLOT_PUBLISH_PROFILE }}
          package: api/fastapi.zip

  # ==================== PROMOTE (RAMP or SWAP) ====================
  promote:
    name: Promote (ramp % or full swap)
    needs: [stage_deploy]
    runs-on: ubuntu-latest
    environment:
      name: Prod   # reviewers required here
      url: https://website.azurewebsites.net/
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # ---------- Option A: RAMP (canary) ----------
      - name: Ramp WEBSITE traffic to staging %
        if: ${{ inputs.mode == 'ramp' }}
        run: |
          PCT=${{ inputs.rampPercentWebsite }}
          if [ "$PCT" -lt 0 ] || [ "$PCT" -gt 100 ]; then echo "Invalid rampPercentWebsite"; exit 1; fi
          PROD=$((100 - PCT))
          az webapp traffic-routing set -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" --distribution staging=${PCT} production=${PROD}
      - name: Ramp FASTAPI traffic to staging %
        if: ${{ inputs.mode == 'ramp' }}
        run: |
          PCT=${{ inputs.rampPercentApi }}
          if [ "$PCT" -lt 0 ] || [ "$PCT" -gt 100 ]; then echo "Invalid rampPercentApi"; exit 1; fi
          PROD=$((100 - PCT))
          az webapp traffic-routing set -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" --distribution staging=${PCT} production=${PROD}
      - name: Hint: finalize ramp
        if: ${{ inputs.mode == 'ramp' }}
        run: |
          echo "Ramping complete. Monitor, then set staging=0 for both apps or run in swap mode."

      # ---------- Option B: SWAP (zero-downtime) ----------
      # 1) Prepare production slot settings (API first, then website)
      - name: Set FASTAPI production slot settings (pre-swap)
        if: ${{ inputs.mode == 'swap' }}
        run: |
          az webapp config appsettings set -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" \
            --settings \
              COSMOS_DB_ENDPOINT="${{ secrets.PROD_COSMOS_DB_ENDPOINT }}" \
              COSMOS_DB_KEY="${{ secrets.PROD_COSMOS_DB_KEY }}" \
              REDIS_URL="${{ secrets.PROD_REDIS_URL }}" \
              OPENAI_ENDPOINT="${{ secrets.PROD_OPENAI_ENDPOINT }}" \
              OPENAI_API_KEY="${{ secrets.PROD_OPENAI_API_KEY }}" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_APP }}.azurewebsites.net"
          az webapp config set -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" \
            --startup-file "gunicorn -k uvicorn.workers.UvicornWorker -w 2 -b 0.0.0.0:\$PORT app:app"

      - name: Set WEBSITE production slot settings (pre-swap)
        if: ${{ inputs.mode == 'swap' }}
        run: |
          az webapp config appsettings set -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" \
            --settings \
              API_BASE_URL="https://${{ env.FASTAPI_APP }}.azurewebsites.net" \
              WEBSITE_BASE_URL="https://${{ env.WEBSITE_APP }}.azurewebsites.net"

      # 2) Clear traffic rules before swapping (best practice)
      - name: Clear traffic routing for both apps
        if: ${{ inputs.mode == 'swap' }}
        run: |
          az webapp traffic-routing clear -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" || true
          az webapp traffic-routing clear -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" || true

      # 3) Swap API first, then Website
      - name: Swap FASTAPI: staging → production
        if: ${{ inputs.mode == 'swap' }}
        run: |
          az webapp deployment slot swap -g "${FASTAPI_RG}" -n "${FASTAPI_APP}" --slot staging --target-slot production
      - name: Swap WEBSITE: staging → production
        if: ${{ inputs.mode == 'swap' }}
        run: |
          az webapp deployment slot swap -g "${WEBSITE_RG}" -n "${WEBSITE_APP}" --slot staging --target-slot production
```

---

## 4) GitHub Secrets & Variables (grouped)

### **Staging** environment → *Secrets*

* `WEBSITE_STAGING_SLOT_PUBLISH_PROFILE`  *(Publish profile for `website` → `staging` slot)*
* `FASTAPI_STAGING_SLOT_PUBLISH_PROFILE`  *(Publish profile for `fastapi` → `staging` slot)*
* `STG_COSMOS_DB_ENDPOINT`, `STG_COSMOS_DB_KEY`, `STG_REDIS_URL`, `STG_OPENAI_ENDPOINT`, `STG_OPENAI_API_KEY`

### **Prod** environment → *Secrets*

* `PROD_COSMOS_DB_ENDPOINT`, `PROD_COSMOS_DB_KEY`, `PROD_REDIS_URL`, `PROD_OPENAI_ENDPOINT`, `PROD_OPENAI_API_KEY`
* *(Optional)* `WEBSITE_PROD_PUBLISH_PROFILE`, `FASTAPI_PROD_PUBLISH_PROFILE` (only if you ever deploy straight to prod slots)

### **Repository-level Secrets**

* `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`

### **Repository-level Variables** (or Secrets)

* `AZ_RG_WEBSITE`  (resource group of `website`)
* `AZ_RG_FASTAPI`  (resource group of `fastapi` — can be same as above)

---

## 5) Operating tips

* **Ramping**: re-run the workflow with `mode=ramp` and new percentages
  (e.g., Website 10% → 25% → 50% → 100%; API 10% → 50% → 100%).
  When done, set both to **0% staging** or run **swap** to finalize.
* **Rollback**: if a swap fails, swap back; if ramping, set `staging=0`.
* **Health checks**: keep `/healthz` fast; App Service pre-warms slots.

---

If you want, I can add a **progressive ramp** job that automatically steps traffic over time (e.g., 10% → 25% → 50% → 100% with sleeps) — just say the increments you prefer.
