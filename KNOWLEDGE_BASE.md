# 🌐 Ancestral Home Operations - Web Portal: Knowledge Base

**Document Version:** 1.1.0  
**Last Updated:** September 2026  
**Primary Repository:** `theancestralhomestay/AncestralHomeOperations`  
**Deployment Target:** Static Web Hosting / GitHub Pages  
**Dynamic Endpoints:**
- **PROD:** `https://script.google.com/macros/s/AKfycbzsSJKyUnW6FDAvHbiDbqRitg2SOPECjgBkk1_vGIZbYsgmf2LuFVQqV4dbyoV-3tg6mQ/exec`
- **UAT:** `https://script.google.com/macros/s/AKfycbyyLa9kOInMAxhZ-Cs0SWuliiyYfvWryEcyxJLyiE0dM0_E3_sWSpAkONIiaM70mGAZNA/exec`

---

## 1. System Overview & Ecosystem Architecture

The **Ancestral Home Operations Web Portal** serves as the lightweight, high-reliability web entryway, dynamic reverse proxy, and responsive wrapper for **The Ancestral Home Operations & Settlement Manager**. It embeds the Google Apps Script (GAS) Web Application inside a secure, full-viewport iframe while providing critical user-experience enhancements: dynamic multi-environment routing (`prod` vs `uat`), seamless loading transitions, device-aware layout handling, accessibility adaptations, and graceful error recovery.

### The Ancestral Homestay Platform Ecosystem

```mermaid
graph TD
    User["👤 End User (Browser / Mobile / Desktop)"]
    
    subgraph Client Layer
        WebPortal["🌐 AncestralHomeOperations (Web Portal / GitHub Pages)"]
        AppShell["📱 AncestralHomeAppShell (React Native / Expo Mobile App)"]
    end
    
    subgraph Core Application Layer
        GASProd["⚙️ AncestralHomestayApp (PROD GAS Web App)"]
        GASUat["🧪 AncestralHomestayApp (UAT GAS Web App)"]
    end
    
    subgraph Data & Cloud Services
        GSheetsProd[("📊 PROD Google Sheets Database")]
        GSheetsUat[("🧪 UAT Google Sheets Database")]
        GDrive["📁 Google Drive (Receipts & Files)"]
        Gmail["📧 Gmail / MailApp (OTP Auth)"]
    end

    User -->|Visits Web URL / ?env=uat| WebPortal
    User -->|Opens Android/iOS App| AppShell
    WebPortal -->|Default: PROD Embed| GASProd
    WebPortal -->|?env=uat: UAT Embed| GASUat
    AppShell -->|WebView Embed| WebPortal
    GASProd <-->|CRUD Data| GSheetsProd
    GASUat <-->|CRUD Data| GSheetsUat
    GASProd <-->|Store Files| GDrive
    GASProd <-->|Send OTPs| Gmail
```

---

## 2. Dynamic Multi-Environment Routing

The portal detects its target environment using **two mechanisms** (either is sufficient):

1. **Path-based detection (automatic):** If the page is served from a URL path containing `/uat` (e.g., the `/uat/` subdirectory deployment), UAT mode activates automatically.
2. **Query parameter (manual/legacy):** Appending `?env=uat` or `?env=test` to any URL activates UAT mode.

```javascript
var pathIsUAT = window.location.pathname.indexOf('/uat') !== -1;
var isUAT = pathIsUAT || envParam === 'uat' || envParam === 'test';
```

| Target Environment | Source Branch | Deployed Path | Embedded Apps Script | Visual Cues |
| :--- | :--- | :--- | :--- | :--- |
| **Production** | `main` | `/` (root) | `AKfycbzsSJKy...` (PROD) | Slate-900 background, Villa logo badge, Amber spinner (`#f59e0b`), standard title |
| **UAT / Staging** | `develop` | `/uat/` | `AKfycbyyLa9k...` (UAT) | 3px orange top bar (persistent), Orange spinner (`#f97316`), "UAT Mode" badge, title `(UAT)` |

---

## 3. Core Component & Behavioral Specifications

### A. Dynamic Viewport & Mobile Address Bar Handling
On mobile browsers (iOS Safari, Android Chrome), browser navigation toolbars dynamically shrink/expand during scrolling, causing CSS `100vh` to produce undesirable layout overflow or clipping.
- **Implementation:**
  ```javascript
  function setSize() {
    iframe.style.height = window.innerHeight + 'px';
  }
  window.addEventListener('resize', setSize);
  setSize();
  ```
- **Benefit:** Maintains exact full-height alignment with dynamic mobile browser viewports.

---

### B. Security & Iframe Sandboxing
The embedded iframe uses explicit HTML5 `sandbox` attributes to balance functionality with browser security constraints:

| Sandbox Attribute | Purpose |
| :--- | :--- |
| `allow-scripts` | Enables the Google Apps Script web application's JavaScript engine to execute. |
| `allow-forms` | Permits form submissions (e.g., OTP authentication, expense logging, income entry). |
| `allow-same-origin` | Allows the embedded document to retain its origin context for local storage/session storage and cookies. |
| `allow-popups` | Permits opening external links (e.g., viewing receipts in Google Drive, opening Google auth prompts). |
| `allowfullscreen` | Enables media/receipt previews to utilize full-screen mode when supported. |

---

### C. State Lifecycle & Timing Engine

```mermaid
sequenceDiagram
    autonumber
    actor User as User Browser
    participant DOM as index.html Wrapper
    participant Frame as iframe (#frame)
    participant GAS as Google Apps Script Server

    User->>DOM: Load Web Portal (Optional ?env=uat)
    DOM->>DOM: Detect environment & configure ENDPOINTS
    DOM->>DOM: Render #loader spinner (hidden iframe)
    DOM->>DOM: Start ERROR_TIMEOUT timer (10,000ms)
    DOM->>Frame: Request resolved GAS Executable URL
    
    alt Load Event Fires (Success within 10s)
        Frame->>GAS: Fetch & render web app
        Frame-->>DOM: iframe 'load' event triggered
        DOM->>DOM: 150ms paint delay buffer
        DOM->>DOM: showIframe() (Hide #loader, display #frame)
    else Timeout Exceeded or Blocked (Failure)
        DOM->>DOM: Timer fires (loaded == false)
        DOM->>DOM: showFallback() (Display #fallback modal)
        User->>DOM: Click 'Open in new tab' or 'Refresh page'
    end
```

1. **Initial State:** `#loader` is displayed with a CSS spinning loader (blue for PROD, orange for UAT). `#frame` is hidden via `visibility: hidden`.
2. **Success State (`load` event):** Once the iframe fires its `load` event, a `150ms` delay buffer executes before setting `#frame` to `visibility: visible` and hiding `#loader`. This prevents visual flicker during sub-resource painting.
3. **Failure State (10s Timeout):** If cross-origin blocking, network failure, or timeout occurs, `#fallback` is displayed with:
   - **Direct Link Button:** Opens the resolved GAS URL directly in a new browser tab (`target="_blank"`).
   - **Refresh Page Button:** Triggers `window.location.reload()` for clean retry.
   - **Support Contact Link:** Directs users to `the.ancestral.home.stay@gmail.com`.

---

### D. Accessibility & Motion Preferences
Respects user OS/browser preferences for reduced motion:
```javascript
try {
  var prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)');
  if (prefersReduced && prefersReduced.matches) {
    var s = document.querySelector('.spinner');
    if (s) s.style.animation = 'none';
  }
} catch (e) {}
```

---

## 4. File Structure & Configuration

```
ancestralhomeoperations/
├── .git/                                     # Git version control metadata
├── .github/
│   └── workflows/
│       └── deploy-pages.yml                  # Composite UAT/Prod GitHub Pages deployment
├── index.html                                # Core single-file portal application
├── KNOWLEDGE_BASE.md                         # Architectural specification and operations manual
└── README.md                                 # Project documentation
```

### Key Configuration Variables (`index.html`)

```javascript
var ENDPOINTS = {
  prod: "https://script.google.com/macros/s/AKfycbzsSJKyUnW6FDAvHbiDbqRitg2SOPECjgBkk1_vGIZbYsgmf2LuFVQqV4dbyoV-3tg6mQ/exec",
  uat:  "https://script.google.com/macros/s/AKfycbyyLa9kOInMAxhZ-Cs0SWuliiyYfvWryEcyxJLyiE0dM0_E3_sWSpAkONIiaM70mGAZNA/exec"
};
```

---

## 5. Deployment & Maintenance Procedures

### Composite Deployment Model

The project uses a **composite GitHub Pages deployment** — both Production and UAT are served from the same site, each sourced from a different branch:

| Environment | Source Branch | Deployed Path | URL |
| :--- | :--- | :--- | :--- |
| **Production** | `main` | `/` (root) | `theancestralhomestay.github.io/AncestralHomeOperations/` |
| **UAT / Staging** | `develop` | `/uat/` | `theancestralhomestay.github.io/AncestralHomeOperations/uat/` |

The GitHub Actions workflow (`deploy-pages.yml`) handles this automatically:

1. **Triggers** on pushes to `main`, `develop`, or manual `workflow_dispatch`
2. **Checks out both branches** (`main` → `_source_prod/`, `develop` → `_source_uat/`)
3. **Assembles a composite `_site/` directory**: production files at root, UAT files in `/uat/`
4. **Deploys the composite site** to GitHub Pages
5. **Safety guarantee:** Production content always comes from `main` — a push to `develop` can never modify production files

### GitHub Environment Prerequisite

The `github-pages` environment must allow deployments from both `main` and `develop`. Configure this under **Repository Settings → Environments → `github-pages` → Deployment branches**.

### Deploying Updates

| Task | How |
| :--- | :--- |
| **Test a change** | Push to `develop` → verify at `/uat/` → production is untouched |
| **Promote to production** | Merge `develop` → `main` → both paths redeploy |
| **Hotfix production** | Push directly to `main` → root is updated immediately |

### Updating Target Google Apps Script URLs
When a new major version or new deployment ID is created in `ancestralhomestayapp`:
1. Open [`index.html`](index.html).
2. Update `ENDPOINTS.prod` or `ENDPOINTS.uat` in the `ENDPOINTS` dictionary at the top of the script.

---

## 6. Release & Change History

| Date | Version | Summary of Changes |
| :--- | :--- | :--- |
| **2026-09-24** | 1.3.0 | Composite UAT/Prod GitHub Pages deployment (`main` → root, `develop` → `/uat/`), path-based UAT auto-detection, persistent 3px orange UAT indicator bar |
| **2026-09-15** | 1.2.0 | Added parent-to-iframe message bridge (`INCOMING_SHARED_FILE` / `SHARED_FILES_RECEIVED`) with cold-start buffer for seamless native share intent support |
| **2026-09-03** | 1.1.0 | Added dynamic multi-environment routing (`?env=uat`), UAT badge/styling, and centralized `ENDPOINTS` configuration dictionary |
| **2026-09-03** | 1.0.0 | Initialized comprehensive architecture knowledge base and README |
| **2026-08-18** | 0.9.0 | Added loading spinner animation, 10s graceful timeout, and retry button |
| **2026-08-11** | 0.1.0 | Initial creation of responsive full-page iframe wrapper |
