# 🌐 The Ancestral Home - Operations Web Portal

A lightweight, responsive web portal wrapper that embeds and delivers **The Ancestral Home Operations & Settlement Manager** web application with full-screen dynamic sizing, loading state animations, accessibility adaptations, and resilient fallback error handling.

---

## 🚀 Overview

- **Source Code:** [`index.html`](index.html)
- **Embedded Web App:** Google Apps Script Web App (`/macros/s/.../exec`)
- **Hosting:** GitHub Pages (automated via GitHub Actions)

| Environment | Branch | URL |
| :--- | :--- | :--- |
| **Production** | `main` | `https://theancestralhomestay.github.io/AncestralHomeOperations/` |
| **UAT / Staging** | `develop` | `https://theancestralhomestay.github.io/AncestralHomeOperations/uat/` |

---

## 📚 Documentation & Architecture

For in-depth architecture diagrams, sandbox security policies, state lifecycles, and configuration guides, please refer to the persistent [Knowledge Base](KNOWLEDGE_BASE.md).

---

## 🔀 Branching & Environment Strategy

The project uses a **composite deployment model** — both environments are served from the same GitHub Pages site, each sourced from a different branch:

```
main (stable, reviewed)       →  deployed to /          (Production)
develop (work-in-progress)    →  deployed to /uat/      (UAT Sandbox)
feature/* branches            →  not deployed (develop via PRs)
```

### How It Works

1. The GitHub Actions workflow ([`deploy-pages.yml`](.github/workflows/deploy-pages.yml)) triggers on pushes to **both** `main` and `develop`
2. It checks out **both branches** and assembles a composite `_site/` directory:
   - `main` files → site root (`/`)
   - `develop` files → `/uat/` subdirectory
3. The composite site is deployed to GitHub Pages as a single artifact
4. **Production content always comes from `main`** — a push to `develop` can never modify production files

### Workflow

```
  ┌─────────────┐     push      ┌──────────────┐     deploy     ┌─────────────────────────┐
  │  develop    │───────────────▶│  GitHub       │──────────────▶│  /uat/index.html        │
  │  branch     │               │  Actions      │               │  (UAT sandbox)           │
  └─────────────┘               │  Workflow     │               ├─────────────────────────┤
  ┌─────────────┐     push      │               │     deploy     │  /index.html             │
  │  main       │───────────────▶│  (assembles   │──────────────▶│  (Production)            │
  │  branch     │               │   composite)  │               └─────────────────────────┘
  └─────────────┘               └──────────────┘
```

| Task | How |
| :--- | :--- |
| **Test a change to the portal** | Push to `develop` → verify at `/uat/` → production is untouched |
| **Promote to production** | Merge `develop` → `main` → both paths redeploy |
| **Hotfix production** | Push directly to `main` → root is updated immediately |

---

## 🧪 UAT Environment Detection

The portal auto-detects UAT mode via **two mechanisms** (both work, either is sufficient):

| Method | Example | When It Activates |
| :--- | :--- | :--- |
| **Path-based** (automatic) | `/AncestralHomeOperations/uat/` | When served from the `/uat/` subdirectory |
| **Query parameter** (manual) | `/AncestralHomeOperations/?env=uat` | When `?env=uat` or `?env=test` is appended to any URL |

### UAT Visual Indicators

When in UAT mode, the portal provides these visual cues:

- 🟠 **Persistent orange bar** — A thin 3px orange strip fixed at the very top of the viewport, visible at all times (even after the app loads inside the iframe)
- 🟠 **Orange loading spinner** — Replaces the default amber spinner during the loading phase
- 🏷️ **"UAT Mode" badge** — Shown on the loading screen
- 📄 **Page title** — Changes to `The Ancestral Home - Operations (UAT)`

Production mode has **none** of these — clean, default styling.

---

## ⚙️ GitHub Environment Setup

> **One-time setup required** for the composite deployment to work.

The `github-pages` environment must allow deployments from both `main` and `develop`:

1. Go to **Repository Settings → Environments → `github-pages`**
2. Under **Deployment branches and tags**, ensure both branches are listed:
   - `main` (should already be there)
   - `develop` (add this)

If `develop` is not in the allowed list, the UAT deployment will be rejected with:
> *"Branch 'develop' is not allowed to deploy to github-pages due to environment protection rules."*

---

## 🛠️ Local Development & Preview

To preview the portal locally, open [`index.html`](index.html) in any modern browser or run a local HTTP server:

```bash
# Python 3 local server
python3 -m http.server 8080

# Or using npx
npx serve .
```

Visit `http://localhost:8080` in your browser.

To test UAT mode locally, either:
- Append `?env=uat` → `http://localhost:8080?env=uat`
- Or serve from a `/uat/` path

---

## 📦 Deployment

Deployment is **fully automated** via GitHub Actions. There is no build step — the HTML/CSS/JS files are deployed directly as static assets.

| Trigger | What Happens |
| :--- | :--- |
| Push to `main` | Composite site redeploys (prod root + UAT `/uat/`) |
| Push to `develop` | Composite site redeploys (prod root + UAT `/uat/`) |
| Manual `workflow_dispatch` | Composite site redeploys from current `main` + `develop` |

### Updating Target Google Apps Script URLs

When a new deployment ID is created in `ancestralhomestayapp`:

1. Open [`index.html`](index.html)
2. Update `ENDPOINTS.prod` or `ENDPOINTS.uat` in the `ENDPOINTS` configuration block
