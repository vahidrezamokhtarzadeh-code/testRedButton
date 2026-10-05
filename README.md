# Cloudflare Pages Deployment Guide

This guide walks you through deploying the Big Red Button site to Cloudflare
Pages and embedding it inside an iframe on your own site.

**Total time**: ~5 minutes
**Cost**: Free (Cloudflare Pages has a generous free tier)
**Result**: A permanent `https://your-project.pages.dev` URL that loads fast for
Iran users (Cloudflare has a Tehran edge node).

---

## Step 1 — Download the deploy package

From the chat preview panel, download the folder `cloudflare-pages-deploy/`
which contains:

```
cloudflare-pages-deploy/
├── index.html      ← the page (big red button + click counter)
├── _headers        ← iframe lock-down config (EDIT THIS — see Step 4)
└── _redirects     ← routing safety (don't touch)
```

Total size: ~4 KB.

---

## Step 2 — Edit `_headers` to lock iframe embedding to YOUR site

Open `_headers` in any text editor. You'll see:

```
/*
  Content-Security-Policy: frame-ancestors 'self' https://your-site.com;
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
```

Replace `https://your-site.com` with the **origin of the site that will host
the iframe**. Examples:

| Your site | What to put in `_headers` |
|---|---|
| `https://example.com` | `frame-ancestors 'self' https://example.com;` |
| `https://www.example.com` and `https://example.com` | `frame-ancestors 'self' https://example.com https://www.example.com;` |
| Any subdomain of example.com | `frame-ancestors 'self' https://*.example.com;` |
| Don't care, allow anyone | `frame-ancestors *;` (NOT recommended for production) |
| Run locally only | `frame-ancestors 'self' http://localhost:*;` |

**Why this matters**: without a `frame-ancestors` rule, anyone can embed your
page in their site. With this rule, only your site can.

---

## Step 3 — Push the folder to a GitHub repo (recommended path)

Cloudflare Pages has two deployment methods — Direct Upload or Git-connected.
The Git path is better because it auto-redeploys on every commit.

1. Go to https://github.com/new and create a new public or private repo
   (e.g. `tempbigredbut`)
2. Drag-drop the 3 files (`index.html`, `_headers`, `_redirects`) into the
   repo root at https://github.com/<you>/tempbigredbut/upload/main
3. Commit.

---

## Step 4 — Connect the repo to Cloudflare Pages

1. Go to https://dash.cloudflare.com/?to=/:account/pages
2. Click **Create application** → **Pages** → **Connect to Git**
3. Authorize Cloudflare to access your GitHub (one-time)
4. Select your `tempbigredbut` repo
5. Configure the build:
   - **Framework preset**: `None` (this is plain HTML, no framework)
   - **Build command**: *(leave empty)*
   - **Build output directory**: `/` (root — since the HTML is in the repo root)
   - **Root directory**: *(leave empty)*
6. Click **Save and Deploy**

Cloudflare will deploy in ~30 seconds and give you a URL like
`https://tempbigredbut.pages.dev` (the project name you chose).

---

## Step 5 — Verify the deployment

```bash
# Should return 200 OK
curl -sI https://tempbigredbut.pages.dev/ | head -3

# Should show your CSP frame-ancestors rule
curl -sI https://tempbigredbut.pages.dev/ | grep -i frame-ancestors
```

Visit `https://tempbigredbut.pages.dev/` in your browser — you should see
the big red button.

---

## Step 6 — Embed the iframe on your own site

Add this snippet to any HTML page on your site:

```html
<iframe
  src="https://tempbigredbut.pages.dev/"
  width="800"
  height="600"
  title="Big Red Button Demo"
  style="border:0; max-width:100%;"
  loading="lazy"
  referrerpolicy="strict-origin-when-cross-origin"
  allow="fullscreen"
></iframe>
```

**Tuning tips**:
- For a card-style embed: `width="640" height="480" style="border:1px solid #e5e7eb; border-radius:12px; overflow:hidden;"`
- For a full-width hero embed: `width="100%" height="600"`
- For mobile: wrap in a container with `aspect-ratio: 4/3; max-width: 100vw;`
- If you don't want the iframe to scroll inside its frame, set `scrolling="no"`
  (but our page is one viewport, so scrolling shouldn't trigger anyway).

---

## Step 7 — (Optional) Add a custom domain

For a customer-facing demo, `tempbigredbut.pages.dev` is fine. If you want
a branded URL like `button.yourcompany.com`:

1. In Cloudflare Pages → your project → **Custom domains** → **Set up a custom domain**
2. Add `button.yourcompany.com`
3. If your domain is also on Cloudflare, DNS is automatic. If not, you'll need
   to add a CNAME record at your DNS provider pointing to
   `tempbigredbut.pages.dev`.
4. Update the iframe `src` to `https://button.yourcompany.com/`.

---

## Alternative: Direct Upload (no GitHub required)

If you don't want to use Git:

1. Zip the contents of `cloudflare-pages-deploy/` (not the folder itself — the
   files inside should be at the zip root).
2. Go to Cloudflare Pages → **Create application** → **Pages** → **Direct Upload**
3. Drag-drop the zip file
4. Click **Deploy**

You'll get a `*.pages.dev` URL. Re-uploading replaces the deployment.

---

## Verifying the iframe lock-down works

After deployment, run this from a different origin (e.g. your local machine)
to confirm the lock-down is in effect:

```bash
curl -sI -H "Referer: https://evil.example.com/" https://tempbigredbut.pages.dev/ \
  | grep -i "frame-ancestors"
```

You should see your rule. Browsers will refuse to render the iframe from any
origin not in the list.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| iframe shows blank white | CSP `frame-ancestors` is blocking your origin | Add your origin to `_headers` and redeploy |
| iframe shows Cloudflare error 1020 | Your site is on Cloudflare's bot-protection blocklist | Disable Bot Fight Mode for this site in Cloudflare dashboard |
| Page loads but button doesn't work | JavaScript disabled on parent page (rare) | The button is plain DOM, no framework — should work everywhere |
| Slow load in Iran | Cloudflare Tehran edge sometimes routes via Frankfurt | Add Iran to the AS path allowlist; or also deploy on BunnyCDN as backup |
| `_headers` rules not applied | Wrong file location or syntax | `_headers` must be at the **root** of the project, not inside `cloudflare-pages-deploy/` |

---

## Why this beats the z.ai preview for customer-facing use

| Aspect | z.ai preview | Cloudflare Pages |
|---|---|---|
| Lifetime | Session-bound, no SLA | Indefinite, free forever |
| Iran latency | 250-450 ms (Aliyun HK) | 80-180 ms (Tehran edge) |
| Custom domain | No | Yes (free) |
| iframe lock-down | Hard (no header control) | One file (`_headers`) |
| Deploy time | Re-init workspace | 30 seconds via Git push |
| Reliability for 7-day customer demo | Risky | Production-grade |
