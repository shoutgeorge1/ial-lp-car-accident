# Landing Page Audit: Speed & Technical

**URL:** https://insideraccidentlawyers.vercel.app/  
**Audit:** Lighthouse (mobile simulation) · Feb 2026

---

## Scores (mobile)

| Category        | Score | Note                    |
|----------------|-------|-------------------------|
| **Performance**   | **52/100** | Below target (aim 90+)   |
| **Best practices**| **57/100** | Console errors, 3rd‑party |
| **CLS**           | **0.002**  | ✅ Excellent (no layout shift) |

---

## Speed metrics

| Metric | Your page | Rough target | Verdict |
|--------|-----------|--------------|--------|
| **First Contentful Paint (FCP)** | 3.8 s | &lt; 1.8 s | ❌ Slow |
| **Largest Contentful Paint (LCP)** | 9.8 s | &lt; 2.5 s | ❌ Very slow |
| **Speed Index** | 8.6 s | &lt; 3.4 s | ❌ Slow |
| **Total Blocking Time (TBT)** | 330 ms | &lt; 200 ms | ⚠️ High |
| **Time to Interactive (TTI)** | 9.9 s | &lt; 3.8 s | ❌ Slow |

Lighthouse also reported: *“The page loaded too slowly to finish within the time limit.”* So real users on slow 4G will see an even slower experience.

---

## What’s hurting speed

### 1. **Render‑blocking Google Fonts**
- **Poppins** and **Inter** are loaded from Google Fonts in the `<head>` with a blocking `<link rel="stylesheet">`.
- That blocks first paint until the CSS (and then the font files) are fetched.
- **Fix:** Load fonts with `display=swap` (you already have it) and add `preload` for the main font, or self‑host and inline only critical font CSS. Better: load the font CSS asynchronously or after first paint.

### 2. **Hero / LCP image**
- LCP is likely the hero background or attorney image.
- **hero-bg.jpg** and **attorney-hero.webp** / **attorney-mobile.webp** are big; they’re requested early and can dominate LCP.
- **Fix:** Preload the LCP image:  
  `<link rel="preload" as="image" href="/images/la/attorney-mobile.webp" media="(max-width: 768px)">`  
  (and equivalent for desktop). Use responsive `srcset`/sizes and keep hero images under ~100–150 KB where possible.

### 3. **Very large HTML document**
- **Main document:** ~21 KB transfer, ~116 KB uncompressed (single huge HTML with all CSS inlined).
- **DOM size:** 498 elements — fine, but the single file means the parser does a lot before first paint.
- **Fix:** Already using one HTML file; keep it. Ensure images and scripts don’t block parsing.

### 4. **Third‑party scripts (main thread + network)**
- **GTM** (gtag + GTM-WS8XT5FC), **CallRail**, **Facebook (fbevents)** load during page load.
- They add ~12+ s of main‑thread work and compete for bandwidth with your content.
- **Fix:** Load GTM/CallRail/Facebook after user interaction or after a short delay (e.g. after `requestIdleCallback` or 2–3 s). Keep critical path: HTML → fonts → hero image → your CSS.

### 5. **Images not optimized for display size**
- Lighthouse “image delivery” flags: many images have **natural dimensions much larger than display size** (e.g. 1200×1200 shown at 88×88, 701×701 at 88×88).
- **Fix:** Serve responsive variants (e.g. 200w, 400w for small cards) or use a CDN that resizes. Convert remaining PNGs/JPGs to WebP/AVIF where you can.

### 6. **Missing favicon**
- **404** on `/favicon.ico` — small cost but triggers a console error and an extra request.
- **Fix:** Add a `favicon.ico` (or `<link rel="icon" href="/favicon.ico">` to an existing icon).

---

## Technical / best‑practices issues

| Issue | Impact | Fix |
|-------|--------|-----|
| **Console errors** | Best‑practices score | 1) Favicon 404 — add file. 2) Google Analytics “Attestation check… failed” — known GA/Privacy Sandbox; monitor for GA updates. |
| **Deprecations** | Future breakage | Attribution Reporting / Topics from **Facebook (fbevents.js)** — update FB script when they fix; not under your direct control. |
| **Third‑party cookies** | Future blocking | 2 cookies (e.g. Google Ads, GA). Plan for cookie‑less tracking (e.g. GA4, server‑side or consent‑based). |
| **HTTPS** | ✅ | All good. |
| **Viewport** | ✅ | Correct. |
| **Server response (TTFB)** | ✅ | ~53 ms — good. |

---

## Quick wins (in order)

1. **Preload LCP image**  
   Add in `<head>`:  
   `<link rel="preload" as="image" href="/images/la/attorney-mobile.webp" media="(max-width: 768px)">`  
   and same for desktop hero image.

2. **Defer non‑critical JS**  
   Load GTM (and thus GA, CallRail, FB) after first paint: e.g. move GTM snippet to end of body or load it in a `setTimeout`/`requestIdleCallback` after 1–2 s.

3. **Make Google Fonts non‑blocking**  
   Use a small inline script to load the font stylesheet asynchronously, or self‑host and preload only one or two weights.

4. **Add favicon**  
   Add `/favicon.ico` (or link to existing icon) to remove 404 and console error.

5. **Resize/serve responsive images**  
   Use `srcset` for hero and attorney images; serve smaller files for small viewports (e.g. 400w/800w). Convert any remaining large PNGs/JPGs to WebP.

---

## Summary

- **Speed:** Performance **52** — main issues are **slow LCP (9.8 s)** and **slow FCP (3.8 s)** from render‑blocking fonts, heavy hero images, and third‑party scripts.
- **Technical:** **57** best practices — favicon 404, console errors, third‑party scripts and cookies; **CLS is excellent (0.002)**.
- **Biggest levers:** Preload LCP image, defer GTM/CallRail/FB until after first paint, and make font loading non‑blocking. Then add favicon and responsive images for a solid next step.
