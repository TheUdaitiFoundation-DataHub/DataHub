# DataHub — deployment handoff for udaiti.org/datahub

DataHub is a self-contained static page: two files, no build step, no server code, no database, no third-party scripts. It has been tested under udaiti.org's current Content Security Policy (CSP) with zero violations, so **no CSP changes are needed** to publish it.

## Files

| File | Size | Purpose |
|---|---|---|
| `index.html` | ~714 KB | Page markup, styles, embedded fonts and images |
| `datahub.js` | ~249 KB | All page scripts (charts, maps, popups, carousel, search, tabs) |

SHA-256 checksums (to confirm the files arrived unchanged):

```
17b54d1eee5b79c8d8e85385ca614cbf2a34252b761df23effab1ffa91a1350e  index.html
ce14c95e4ff5bf01a3a3d80cb6d252e704715642d80dab68014478a2095aaf77  datahub.js
```

Check on macOS/Linux with `shasum -a 256 index.html datahub.js`.

## Steps

### 1. Add the files to the Next.js app

Place both files in the app's `public/` folder:

```
public/
  datahub/
    index.html
    datahub.js
```

`index.html` loads the script from the absolute path `/datahub/datahub.js`, so the folder name must be `datahub`.

### 2. Serve `/datahub` as the page

Next.js serves files in `public/` at their exact path (`/datahub/index.html`) but does not map `/datahub` to `index.html` on its own. Add rewrites in `next.config.js` (merge with any existing `rewrites`):

```js
async rewrites() {
  return [
    { source: '/datahub', destination: '/datahub/index.html' },
    { source: '/datahub/', destination: '/datahub/index.html' },
  ];
},
```

If Apache serves static files directly instead of passing them to Next.js, the equivalent is `DirectoryIndex index.html` for that path.

### 3. Keep the existing security headers — do not loosen them

- Do **not** add `'unsafe-inline'` to `script-src`. The page has no inline scripts and no inline event handlers.
- The page already works with the current rules: `script-src 'self'`, `style-src 'self' 'unsafe-inline'`, `font-src data:` (fonts are embedded), `img-src data:`.
- If the CSP is set in middleware with a `matcher` that skips static files, that is also fine. The page works with or without the CSP header.

### 4. Check the deployment (about 5 minutes)

Open `https://www.udaiti.org/datahub` with the browser developer tools Console open, and confirm:

1. **No red errors** in the Console, and no "Content Security Policy" messages.
2. **Home page:** the Featured Dashboards carousel shows bar charts and an India map. The Insights carousel scrolls.
3. **Data and Insights:** clicking any insight card opens a popup with a chart.
4. **Explore Dashboards:** the Entrepreneurship tab shows the State Startup Policy Index map, and hovering a state shows a tooltip.
5. **Search:** type `pay gap` in the header search and press Enter (or click the magnifier). A results page opens and the address bar shows `/datahub#search=pay%20gap`. Clicking a result opens the insight. The browser Back button returns to the results.
6. **Network tab:** `datahub.js` loads with status 200 (not 404), from `/datahub/datahub.js`.

If the page loads but nothing is interactive, `datahub.js` is not being served: check step 1's folder name and step 2's rewrite.

### 5. Recommended hardening (site-wide, not DataHub-specific)

- Remove version banners from responses: set `ServerTokens Prod` and `ServerSignature Off` in Apache, and `poweredByHeader: false` in `next.config.js`. Responses currently expose `Server: Apache/2.4.58 (Ubuntu)` and `X-Powered-By: Next.js`.
- Keep MFA on hosting, domain registrar and GitHub accounts.

## Things that are not in these files

- **Subscribe form:** the email sign-up boxes are not connected to anything yet. When connected, note that the site CSP has `form-action 'self'`. Either post to an endpoint on udaiti.org, or add the email provider's form URL to `form-action`. Use double opt-in and bot protection, and show a consent line (DPDP Act 2023).
- **External dashboards:** "Go to dashboard" buttons and insight popups open separate sites in a new tab (`theudaitifoundation-datahub.github.io/PLFS/`, `theudaitifoundation-datahub.github.io/Startup_Policy/`, `epfodata.z29.web.core.windows.net`, `cggdashboarddata.z29.web.core.windows.net`). They are links only, so no CSP changes are needed. Moving them to subdomains under udaiti.org is recommended, so the links can't be taken over if a storage account is ever deleted.
- **PLFS dashboard data:** the standalone PLFS dashboard (not part of this page) reads live data from a Google Apps Script / Google Sheet. Restrict editing of that sheet to named people, and keep the script read-only.
- **Placeholder links:** about 30 links (privacy policy, cookie notice, licence, footer items) are `href="#"` placeholders. Point them to real pages before launch.

## Search results URL

Search results live at `/datahub#search=<terms>` (a URL fragment, handled entirely in the browser). No server route is needed, and these links can be shared or bookmarked.

## Updating content later

Content edits arrive as a new pair of `index.html` + `datahub.js`. Replace both files together; never update one without the other.
