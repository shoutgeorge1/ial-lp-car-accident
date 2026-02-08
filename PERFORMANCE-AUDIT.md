# Performance Audit — Insider Accident Lawyers

**URL:** https://insideraccidentlawyers.vercel.app/  
**Audit:** Lighthouse 12 (mobile, simulated 4G)  
**Date:** Feb 2026  
**Note:** Page sometimes times out waiting for third-party requests (e.g. Facebook); results can be incomplete.

---

## Summary

| Metric | Value | Target | Verdict |
|--------|--------|--------|---------|
| **Performance (mobile)** | ~45–55 | 90+ | ❌ Poor |
| **First Contentful Paint (FCP)** | **3.2 s** | &lt; 1.8 s | ❌ Slow |
| **Largest Contentful Paint (LCP)** | **7.5 s** | &lt; 2.5 s | ❌ Very slow |
| **Speed Index** | **5.4 s** | &lt; 3.4 s | ❌ Slow |
| **Total Blocking Time (TBT)** | **400 ms** | &lt; 200 ms | ⚠️ High |
| **Time to Interactive (TTI)** | **7.6 s** | &lt; 3.8 s | ❌ Slow |
| **Cumulative Layout Shift (CLS)** | **0** | &lt; 0.1 | ✅ Excellent |
| **Server response (TTFB)** | **~63 ms** | &lt; 600 ms | ✅ Good |

---

## What’s Hurting Performance

### 1. Render‑blocking Google Fonts (~2 s impact)

- **Issue:** The Google Fonts stylesheet is loaded with a blocking `<link rel="stylesheet">` in `<head>`. The browser waits for it (and the font files it references) before first paint.
- **Evidence:** Critical request chain is **HTML → fonts.googleapis.com CSS → font files** (Inter + Poppins). “Eliminate render-blocking resources” reports **~1,980 ms** potential savings.
- **Fix:** Load the font CSS asynchronously (e.g. `<link rel="preload" as="style" href="..." onload="this.rel='stylesheet'">` and a `<noscript>` fallback), or self-host and preload only the critical font file. You already use `display=swap`, which is good once the CSS is non-blocking.

### 2. LCP element and hero images

- **Issue:** LCP is likely the hero/attorney image. Even with preloads for `attorney-mobile.webp` and `attorney-hero.webp`, the LCP is still **7.5 s** — third-party and font work delay the main thread and compete for bandwidth.
- **Fix:** Keep the existing preloads. Ensure the LCP image has `fetchpriority="high"` and that no other heavy resources block the main thread before it paints. Reducing render-blocking (fonts) and deferring third-party JS (below) will help LCP.

### 3. Third‑party scripts (main thread + network)

- **Issue:** GTM, gtag (GA), CallRail, and Facebook (fbevents) load during page load. They add **~1.1 s+** of script execution and compete for bandwidth. Long tasks (e.g. **357 ms** from gtag) contribute to TBT and delayed interactivity.
- **Evidence:** “Minimize main-thread work” ~**10.9 s** total; “JavaScript execution time” lists gtag, GTM, fbevents, CallRail as top contributors.
- **Fix:** Defer loading of GTM/CallRail/Facebook until after first paint or after a short delay (e.g. `requestIdleCallback` or 2–3 s). Load GTM (and thus GA/CallRail/FB) from a small inline script at the end of `<body>` or after a timeout so the critical path is **HTML → (optional non-blocking fonts) → hero image → your CSS**.

### 4. Image delivery and weight

- **Issue:** Many images have **natural dimensions much larger than display size** (e.g. 1200×1200 or 701×701 shown at 88×88). Several large assets: **caoc.png** (~165 KB), **hero-bg.jpg** (~112 KB), **yelp-logo** duplicates (~74 KB each), **trust-badge-4.webp** (~46 KB), etc. Total transfer **~1.28 MB**.
- **Fix:** Serve responsive images (`srcset`/sizes) for hero and cards; use smaller variants for small viewports. Convert remaining PNGs/JPGs to WebP/AVIF where possible. Lazy-load offscreen images (e.g. **stop-sign.jpg**, **gold-badge.jpg** — Lighthouse “Defer offscreen images” ~45 KB savings).

### 5. Cache policy for static assets

- **Issue:** “Serve static assets with an efficient cache policy” fails for **4 resources** (e.g. some third-party or short TTL). **~120 KB** could benefit from longer cache.
- **Fix:** Ensure your own static assets (e.g. under `/images/`) are served with a long cache TTL (e.g. 1 year) and use cache-busting via filename or query param. Third-party assets are not under your control.

### 6. Minor: Unminified inlined CSS

- **Issue:** Inlined CSS has **~4 KB** potential savings if minified.
- **Fix:** Optional: run the inlined styles through a minifier in your build step. Lower priority than fonts and third-party.

---

## Quick wins (in order)

1. **Make Google Fonts non‑blocking** — Load the font stylesheet asynchronously (or self-host and preload one critical font). Largest single improvement for FCP/LCP.
2. **Defer third‑party JS** — Load GTM (and thus GA, CallRail, FB) after first paint or after 2–3 s so the critical path is shorter.
3. **Keep LCP preloads and add `fetchpriority="high"`** — You already preload the hero images; add `fetchpriority="high"` on the LCP `<img>`.
4. **Lazy-load offscreen images** — Add `loading="lazy"` to images below the fold (e.g. stop-sign, gold-badge) to save bandwidth and a bit of LCP.
5. **Resize/optimize images** — Use `srcset`/sizes and smaller file sizes for small viewports; convert large PNGs/JPGs to WebP where possible.

---

## What’s already good

- **HTTPS** — All good.
- **Viewport** — Correct.
- **CLS** — 0 (no layout shift).
- **Preconnect** — You preconnect to `fonts.googleapis.com` and `fonts.gstatic.com`.
- **LCP image preload** — You preload `attorney-mobile.webp` and `attorney-hero.webp` with media queries.
- **Favicon** — Fixed (no 404).
- **Server response** — ~63 ms TTFB.

---

## How to re-run the audit

```bash
npx lighthouse "https://insideraccidentlawyers.vercel.app/" --only-categories=performance --output=json --output-path=./lighthouse-perf-audit.json --chrome-flags="--headless --no-sandbox"
```

Then open the JSON or use the HTML report to inspect metrics and opportunities in detail.
