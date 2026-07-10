# Azure App Portal

One place to see **every Azure Static Web App** in the subscription — with
human-readable display names and descriptions — so auto-generated hostnames
like `purple-ground-0f377120f` never get lost or forgotten.

This is a **standalone** app: its own repo, its own Static Web App, its own URL.
It is fully separate from the job-application platform.

## How it works

```
┌─────────────────────┐         ┌──────────────────────────────┐
│  azure-app-portal    │  GET/   │  job-platform-api (Functions) │
│  (this repo)         │  POST   │  /api/apps                    │
│  static UI, no creds │ ──────▶ │  service principal → Azure ARM │
└─────────────────────┘         └──────────────────────────────┘
```

- **`index.html`** — the entire portal UI (self-contained HTML/CSS/JS). It calls
  a **scan API** for the data. It holds **no credentials**.
- The **scan API** (`/api/apps`) lives on the `job-platform-api` Function App,
  which already holds the Azure service principal. It lists every
  `Microsoft.Web/staticSites` resource, enriches each with its creation time,
  and merges in curated display names / descriptions stored in Azure Table
  storage. `GET` returns the list; `POST` saves a name/description per app.

### Why the scan API is reused rather than duplicated

Scanning Azure requires a server-side identity (a service-principal secret) to
mint an ARM token — a static site can't do that safely in the browser. The
`job-platform-api` Function App already has that identity and the right
permissions, so the portal calls it instead of standing up a second copy of the
same credential. The **UI is standalone**; the **scan is a shared service**.

The API base URL is configurable — set `?api=https://your-host` on the URL, or
use the "change" link in the footer (stored per-browser in `localStorage`).

## Deploy

Pushed to `main` → GitHub Actions (`.github/workflows/azure-static-web-apps.yml`)
deploys the static site to the `app-portal-web` Static Web App.

**No per-app deploy-token secret is needed.** Following the same pattern as
`executive-engine-deploy.yml`, the workflow logs in with the shared service
principal and fetches this app's own deploy token at runtime:

```yaml
azure/login  ← AZURE_CLIENT_ID / AZURE_CLIENT_SECRET / AZURE_TENANT_ID / AZURE_SUBSCRIPTION_ID
az staticwebapp secrets list --name app-portal-web  → the deploy token
```

So the only requirement is that those four service-principal secrets are
available to this repo (as org secrets, or added to the repo). SWA deploy tokens
are per-app, which is why a shared `AZURE_STATIC_WEB_APPS_API_TOKEN` can't be
reused here — it belongs to a different Static Web App.

### First-time provisioning (already done for `app-portal-web`)

```bash
az staticwebapp create \
  --name app-portal-web \
  --resource-group EnterpriseDS_ResourceGRP \
  --location eastus2 \
  --sku Free
```

## Features

- Live scan of all Static Web Apps, newest first
- **NEW** badge for apps created in the last 48 hours
- Search across name / description / hostname / tags
- Inline editing of display name + description (persisted server-side)
- Direct **Open** links to each live app and its source repo
- Hide retired apps from the default view
- Light / dark theme (follows the OS)