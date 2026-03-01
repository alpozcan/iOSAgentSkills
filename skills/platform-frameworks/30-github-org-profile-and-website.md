---
title: "GitHub Organisation Profile & Static Website"
description: "Setting up a GitHub org profile README, static marketing website on GitHub Pages with custom domain, DNS configuration, and SSL — no build system, no dependencies."
---

# GitHub Organisation Profile & Static Website

## Context

Every app shipped via a GitHub organisation needs two public-facing surfaces: (1) a GitHub org profile README that visitors see when landing on `github.com/YourOrg`, and (2) a marketing website hosted on GitHub Pages with a custom domain. Both should be dark-themed, Apple-native in feel, and require zero build tooling — just static HTML/CSS/JS files served as-is.

## Pattern

### Repository Structure

Three repositories under one GitHub organisation:

```
YourOrg/
├── .github/              # Org profile repo (special GitHub repo)
│   ├── README.md         # Simple intro shown on org overview
│   ├── profile/
│   │   └── README.md     # Rich HTML profile displayed on org page
│   └── assets/
│       └── demo.gif      # Hero image/animation
│
├── YourApp/              # Main app source code
│   └── ...
│
└── wwwyourapp/           # Marketing website (GitHub Pages)
    ├── index.html         # Landing page (all CSS/JS inline)
    ├── support.html       # FAQ + contact
    ├── privacy.html       # Privacy policy
    ├── 404.html           # Error page
    ├── CNAME              # Custom domain (e.g., yourapp.com)
    ├── .nojekyll          # Disable Jekyll processing
    └── assets/
        ├── app-icon.png   # App icon for nav/hero
        └── screenshots/   # App screenshots for gallery
```

### Deployment Checklist

Setting up a new GitHub org with profile README, marketing website, and custom domain from scratch:

**Step 1 — Create the three repos under your GitHub org:**

```bash
gh repo create YourOrg/.github --private --description "Org profile"
gh repo create YourOrg/wwwyourapp --private --description "Marketing website"
# YourOrg/YourApp already exists
```

**Step 2 — Make both public-facing repos public:**

GitHub Pages requires a public repo on the free plan. The `.github` repo must also be public for the org profile README to render.

```bash
gh repo edit YourOrg/.github --visibility public --accept-visibility-change-consequences
gh repo edit YourOrg/wwwyourapp --visibility public --accept-visibility-change-consequences
```

> **Critical:** The `.github` repo MUST be public. If it is private, GitHub silently hides the org profile README — you'll see the default "We think you're gonna like it here" onboarding page instead.

**Step 3 — Add required files to the website repo:**

```bash
# In wwwyourapp/
echo "www.yourapp.com" > CNAME
touch .nojekyll
# Add index.html, privacy.html, support.html, 404.html, assets/
git add -A && git commit -m "feat: initial website" && git push
```

**Step 4 — Enable GitHub Pages via API:**

```bash
gh api repos/YourOrg/wwwyourapp/pages -X POST \
  -f "source[branch]=main" \
  -f "source[path]=/"
```

Or manually: repo **Settings → Pages → Source → Deploy from a branch → main → / (root)**.

**Step 5 — Configure DNS at your registrar (e.g., Namecheap):**

Go to your registrar's **Advanced DNS** settings for `yourapp.com`. Remove any parking page records, then add:

| Type | Host | Value |
|------|------|-------|
| CNAME | `www` | `yourorg.github.io` |
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |

Also disable **Namecheap Parking Page** and any **URL Redirect** settings.

**Step 6 — Verify DNS propagation:**

```bash
dig www.yourapp.com CNAME +short   # expect: yourorg.github.io.
dig yourapp.com A +short           # expect: 185.199.108-111.153
```

DNS propagation takes 10-30 minutes. During this time the site may return 404.

**Step 7 — Enforce HTTPS (after SSL cert is issued, ~15 min after DNS):**

```bash
gh api repos/YourOrg/wwwyourapp/pages -X PUT \
  -F "https_enforced=true" \
  -F "source[branch]=main" \
  -F "source[path]=/"
```

If you get "The certificate does not exist yet", wait a few more minutes and retry. You can also enable it in the repo **Settings → Pages → Enforce HTTPS** checkbox.

**Step 8 — (Optional) Verify the custom domain for the org:**

In org **Settings → Pages → Add a domain**, add `yourapp.com` and follow the DNS TXT verification. This prevents other GitHub users from claiming your domain.

### GitHub Org Profile README

The `.github` repository is a special GitHub repo. Its `profile/README.md` renders on the org's main page.

**profile/README.md** — rich HTML layout for the org page:

```markdown
<p align="center">
  <img src="https://github.com/YourOrg/.github/raw/main/assets/screenshot-main.png" alt="Main window" width="720">
</p>

<h1 align="center">yourapp</h1>

<p align="center">
  <strong>Tagline — key differentiator.</strong>
</p>

<p align="center">
  <a href="https://yourapp.com">Website</a> •
  <a href="https://yourapp.com/privacy">Privacy</a> •
  <a href="https://yourapp.com/support">Support</a>
</p>
```

**Adding screenshots:** Place in `.github/assets/` — one full-width hero + two side-by-side detail shots:

```markdown
<p align="center">
  <img src="https://github.com/YourOrg/.github/raw/main/assets/screenshot-main.png" alt="Main" width="720">
</p>

<p align="center">
  <img src="https://github.com/YourOrg/.github/raw/main/assets/screenshot-detail.png" alt="Detail" width="360">
  <img src="https://github.com/YourOrg/.github/raw/main/assets/screenshot-code.png" alt="Code" width="360">
</p>
```

### Static Website (GitHub Pages)

A single-page dark marketing site with embedded CSS/JS — no build system.

**Design system (CSS custom properties):**

```css
:root {
  --accent: #7c5bf0;              /* Your app's brand color */
  --accent-dim: #6344d0;
  --accent-glow: rgba(124, 91, 240, 0.15);
  --bg: #09090b;
  --bg-card: #111113;
  --bg-subtle: #18181b;
  --text: #fafafa;
  --text-secondary: #a1a1aa;
  --text-dim: #71717a;
  --border: rgba(255, 255, 255, 0.06);
  --border-hover: rgba(255, 255, 255, 0.1);
  --radius: 16px;
}
```

**Page structure:**

1. **Fixed nav** — frosted glass (`backdrop-filter: blur(20px)`), brand + links + CTA button
2. **Hero** — app icon with glow animation, headline with gradient text, subtitle, action buttons
3. **Screenshot gallery** — macOS/iOS window chrome frames with 3 key screenshots
4. **How It Works** — 4-column pipeline grid with numbered steps
5. **Features** — 3-column card grid with icon, title, description
6. **Requirements** — 2-column cards (minimum vs recommended)
7. **CTA** — centered box with radial glow, headline, download button
8. **Footer** — brand, links, copyright

**Supporting pages** (same nav/footer, narrower `720px` content column):
- `support.html` — download card (accent gradient border), contact card, FAQ items, system requirements
- `privacy.html` — data collection, screen recording/camera, on-device AI, data storage, analytics & crash reporting, third-party services
- `404.html` — minimal centered error with home link

**SEO & Social Meta:**

```html
<meta property="og:title" content="YourApp — Tagline">
<meta property="og:description" content="Description">
<meta property="og:image" content="https://www.yourapp.com/assets/app-icon.png">
<meta property="og:url" content="https://www.yourapp.com">
<meta property="og:type" content="website">
<meta name="twitter:card" content="summary_large_image">
```

## Edge Cases

- **`.github` repo must be public** — if private, the org profile README is silently hidden. GitHub shows the default onboarding page with no error message.
- **GitHub image caching:** Append `?v=2` query param to bust cache when updating images.
- **CNAME propagation:** After adding the CNAME file and DNS records, GitHub Pages may take 10-30 minutes to issue the SSL certificate. The "Enforce HTTPS" API call will fail until then.
- **`.nojekyll` is required:** Without it, GitHub Pages runs Jekyll which breaks inline `{{ }}` patterns.
- **Font licensing:** Use system fonts (`-apple-system, BlinkMacSystemFont, 'SF Pro Display'`) — no external fonts needed.
- **Namecheap specifics:** Disable "Parking Page" and "URL Redirect" in Advanced DNS before adding records, or they'll conflict.
- **Screenshot file size:** 3-4 key screenshots max for the website, 2-3 for the README.
- **Privacy page per-app analytics:** Each app's privacy page must reflect its own analytics behavior. Don't copy-paste between apps.

## Why This Matters

- **Zero dependencies:** No Node.js, no bundler, no CI pipeline. Edit HTML, push, done.
- **Instant load:** Inline CSS/JS means one HTTP request per page.
- **GitHub-native hosting:** Free, with automatic SSL, CDN, and custom domain support.
- **Brand consistency:** The org profile and website share the same voice, colour palette, and messaging.
- **Privacy-first messaging:** App users care about privacy — both the website and README should emphasise on-device processing.

## Anti-Patterns

- Don't use a static site generator for a 4-page marketing site — the complexity isn't justified
- Don't embed Google Fonts — use system fonts for faster loading and native feel
- Don't forget `.nojekyll` — your site will break silently on GitHub Pages
- Don't keep the `.github` repo private — the org profile README won't render
- Don't use `<center>` tags — they're deprecated; use `<p align="center">` for GitHub Markdown
