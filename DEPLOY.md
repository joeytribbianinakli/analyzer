# Deploy Stock Analyzer

This package separates the app into two parts:

- `site/` — static HTML, CSS, and JavaScript for GitHub Pages.
- `worker/` — a stateless Cloudflare Worker that fetches SEC EDGAR, Finnhub, and Yahoo Finance data.

No Cloudflare KV, D1, or other server-side database is used. Analysis responses are cached for about 24 hours in the browser's IndexedDB. The watchlist and portfolio are stored in that browser's local storage, so they do not automatically sync between devices or browsers.

## Prerequisites

- A GitHub account.
- A Cloudflare account.
- Node.js 18 or newer (or Bun) for Wrangler.
- Optional: a Finnhub API key. The app works without one.

## 1. Deploy the Cloudflare Worker

Open a terminal in the `worker/` folder.

1. Install Wrangler:

   ```bash
   npm install
   ```

2. Sign in to Cloudflare:

   ```bash
   npx wrangler login
   ```

3. Open `wrangler.toml` and set:

   - `ALLOWED_ORIGIN` to the exact origin of your GitHub Pages site. Use only the scheme and host, with no repository path and no trailing slash. For a standard user site, that is in the form `https://YOUR-USERNAME.github.io`.
   - `SEC_USER_AGENT` to a descriptive app name followed by your real contact email. The SEC asks automated clients to identify themselves.

4. Optional: add Finnhub as the first fallback after SEC EDGAR:

   ```bash
   npx wrangler secret put FINNHUB_API_KEY
   ```

   Paste the key into Wrangler's secure prompt. Do not add the key to `worker.js`, `wrangler.toml`, GitHub, or the website files.

5. Deploy:

   ```bash
   npx wrangler deploy
   ```

6. Copy the HTTPS Worker URL shown by Wrangler. Opening its `/status` route should return JSON with `ok: true` and whether Finnhub is configured.

### CORS for local testing

A page opened directly from disk has the opaque `null` origin and is intentionally not allowed by the provided production CORS setting. Test through a local web server instead, and temporarily add its origin as a comma-separated value:

```toml
ALLOWED_ORIGIN = "https://YOUR-USERNAME.github.io,http://localhost:8080"
```

Remove the localhost origin and redeploy when testing is complete.

## 2. Publish the static site on GitHub Pages

1. Create a new GitHub repository.
2. Copy the **contents** of `site/` into the repository root. The repository root should contain `index.html`, `.nojekyll`, and `assets/`.
3. Commit and push the files to the repository's default branch.
4. In the repository, open **Settings → Pages**.
5. Under **Build and deployment**, select **Deploy from a branch**.
6. Select the default branch and the `/ (root)` folder, then save.
7. Wait for GitHub Pages to publish the site and open the URL it provides.

The generated site uses relative asset paths, so it works for both a user site and a project site.

## 3. Connect the website to the Worker

1. Open the deployed Stock Analyzer site.
2. Select **Settings** in the top navigation.
3. Paste the HTTPS Worker URL from `wrangler deploy`.
4. Choose **Save Worker URL**.
5. Search a ticker to verify the connection.

The URL is stored only in that browser. Repeat this step on each device or browser you use.

## Data-source order

For reported fundamentals, the Worker keeps the original app's priority:

1. SEC EDGAR — primary source for U.S. issuers.
2. Finnhub — first fallback when `FINNHUB_API_KEY` is configured.
3. Yahoo Finance — second fallback.

Yahoo Finance is also used for market price history, dividends, analyst estimates, and ownership data when available. Upstream providers can change or temporarily reject requests; the interface reports unavailable fields rather than inventing values.

## Browser storage and privacy

- Analysis cache: IndexedDB, approximately 24 hours per ticker.
- Watchlist: local storage.
- Portfolio: local storage.
- Finnhub key: Cloudflare Worker secret only.
- Cloudflare storage: none. No KV namespace is declared or used.

To clear local app data, use the browser's site-data controls for the GitHub Pages site. A hard refresh does not clear IndexedDB or local storage.

## Update or rebuild the frontend

The deployable files are already in `site/`. The editable React source is included in `source/`.

```bash
cd source
bun install
bun run build
```

Copy the rebuilt contents of `source/dist/` into `site/`, commit, and push. To rebuild the Worker after editing `source/worker/worker-source.ts`:

```bash
cd source
bun run build:worker
cp worker/worker.js ../worker/worker.js
cd ../worker
npx wrangler deploy
```

## Troubleshooting

- **"Set your Cloudflare Worker URL"** — open Settings in the app and save the deployed Worker URL.
- **CORS error** — make sure `ALLOWED_ORIGIN` exactly matches the GitHub Pages origin, then redeploy the Worker.
- **Finnhub says not configured** — run `npx wrangler secret put FINNHUB_API_KEY`, then redeploy.
- **A ticker lacks fundamentals** — it may not report to SEC EDGAR, or an upstream fallback may not provide the required statements.
- **Old data remains visible** — use the app's Refresh button to bypass the 24-hour IndexedDB cache for that ticker.
