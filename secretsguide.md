## 🟩 1. WEBSITE_STAGING_SLOT_PUBLISH_PROFILE

### 🔹 Location in Portal

**For Web App (Website) → Staging Slot**

1. Go to **Azure Portal → App Services**.
2. Open your **website app** (e.g., `website-stg`).
3. In the **left menu**, scroll to **“Deployment Center”** → or click **“Overview”**.
4. On the **top bar**, click **“Get publish profile.”**
   → This downloads an `.PublishSettings` XML file.
5. Open it in any text editor. Copy the entire XML — that’s your **publish profile**.

   * You’ll later add it to GitHub as a secret named:
     `WEBSITE_STAGING_SLOT_PUBLISH_PROFILE`.

---

## 🟩 2. FASTAPI_STAGING_SLOT_PUBLISH_PROFILE

### 🔹 Location in Portal

**For FastAPI App → Staging Slot**
Repeat the same steps:

1. **App Services →** your FastAPI app (e.g., `fastapi-stg`).
2. **Get Publish Profile** → Download the `.PublishSettings`.
3. Save or copy the XML contents.
4. Add to GitHub Actions secrets:
   `FASTAPI_STAGING_SLOT_PUBLISH_PROFILE`.

---

## 🟦 3. STG_COSMOS_DB_ENDPOINT & STG_COSMOS_DB_KEY

### 🔹 Location in Portal

1. Go to **Azure Portal → Azure Cosmos DB Accounts**.
2. Open your **Cosmos DB instance** (e.g., `cosmos-stg`).
3. In the left menu → click **Keys**.
4. Copy:

   * **URI** → use as `STG_COSMOS_DB_ENDPOINT`
   * **Primary Key** → use as `STG_COSMOS_DB_KEY`

---

## 🟥 4. STG_REDIS_URL

### 🔹 Location in Portal

1. Go to **Azure Cache for Redis**.
2. Open your **Redis instance** (e.g., `redis-stg`).
3. In the left menu → **Access keys**.
4. Copy:

   * **Primary connection string (SSL)** → use as `STG_REDIS_URL`.

---

## 🟨 5. STG_OPENAI_ENDPOINT & STG_OPENAI_API_KEY

### 🔹 Location in Portal

1. Go to **Azure OpenAI** resource.
2. In the left menu → **Keys and Endpoint**.
3. Copy:

   * **Endpoint URL** → `STG_OPENAI_ENDPOINT`
   * **Key 1** (or Key 2) → `STG_OPENAI_API_KEY`.

---

## 💡 Summary Table

| Variable                             | Azure Service         | Portal Path                             |
| ------------------------------------ | --------------------- | --------------------------------------- |
| WEBSITE_STAGING_SLOT_PUBLISH_PROFILE | App Service (Web)     | App → Overview → Get Publish Profile    |
| FASTAPI_STAGING_SLOT_PUBLISH_PROFILE | App Service (FastAPI) | App → Overview → Get Publish Profile    |
| STG_COSMOS_DB_ENDPOINT               | Cosmos DB             | Keys → URI                              |
| STG_COSMOS_DB_KEY                    | Cosmos DB             | Keys → Primary Key                      |
| STG_REDIS_URL                        | Azure Cache for Redis | Access Keys → Primary Connection String |
| STG_OPENAI_ENDPOINT                  | Azure OpenAI          | Keys and Endpoint → Endpoint            |
| STG_OPENAI_API_KEY                   | Azure OpenAI          | Keys and Endpoint → Key 1               |

---
