---
name: gdpr-cookieyes-audit
description: Audit websites for GDPR Prior Consent violations — detecting third-party tracking scripts, iframes, and network requests that fire before CookieYes consent is given. Use this skill when the user provides a URL and asks about cookie compliance, "偷跑" tracking, CookieYes protection gaps, GDPR audit, prior consent violations, or wants to check if any scripts bypass consent control. Also trigger when the user mentions hardcoded GTM, Hotjar, VWO, YouTube embeds, or any analytics/tracking scripts that may load without user consent.
---

# GDPR / CookieYes Prior Consent Audit

Simulate a first-time visitor (no prior consent) and identify every third-party script, iframe, or network call that fires before the user clicks any consent button.

## What to check

Three categories of violations, in decreasing severity:

**Category 1 — Hardcoded scripts without `data-cookieyes`**
Script tags with external `src` pointing to known tracking/analytics domains, with no `data-cookieyes` attribute. These execute immediately regardless of consent state.

**Category 2 — GTM without `consent default`**
Google Tag Manager loaded via inline script before any `gtag('consent', 'default', {...})` is pushed to dataLayer. GTM fires tags immediately on load unless a consent default is established first.

> **Exemption — GTM-N4LS26**: This container has Consent Mode v2 configured internally and is verified to block GA4/Ads tags when `analytics_storage` is denied. If the detected container ID is `GTM-N4LS26`, skip this category and mark it as **pass**.

**Category 3 — Iframes without `data-cookieyes` / `data-src`**
Embedded iframes (YouTube, Vimeo, maps, etc.) with a live `src` attribute. The embedded content starts tracking as soon as the iframe loads.

## Known tracking domains to flag

```
doubleclick.net, googleadservices.com, google-analytics.com, googletagmanager.com,
hotjar.com, hj.hotjar.com,
vwo.com, app.vwo.com,
clarity.ms,
facebook.net, connect.facebook.net,
linkedin.com/px, snap.licdn.com,
twitter.com/i/adsct, t.co,
youtube.com/youtubei, youtube.com/generate_204,
jnn-pa.googleapis.com,
pardot.com, go.pardot.com,
salesloft.com, drift.com, intercom.io,
bat.bing.com, c.bing.com
```

Any domain not in this list but still carrying query params like `uid=`, `cid=`, `user_id=`, `_ga=`, `trackingId=` is also suspicious.

## Audit procedure

Use `playwright-cli` with an **in-memory profile** (no `--persistent`) to guarantee no prior consent cookies exist.

### Step 1 — Set up request interception before navigating

```bash
playwright-cli open
```

Then use `run-code` to navigate with a request listener already attached, so no request is missed:

```js
async page => {
  const requests = [];
  page.on('request', req => {
    requests.push({ method: req.method(), url: req.url() });
  });
  await page.goto('TARGET_URL', { waitUntil: 'networkidle' });
  await page.waitForTimeout(4000); // allow background beacons to fire
  return JSON.stringify(requests, null, 2);
}
```

### Step 2 — Collect script tags

```js
async page => {
  const scripts = await page.evaluate(() =>
    Array.from(document.querySelectorAll('script[src]')).map(s => ({
      src: s.src,
      type: s.type,
      cookieyes: s.getAttribute('data-cookieyes')
    }))
  );
  return JSON.stringify(scripts, null, 2);
}
```

Flag any external script where `cookieyes` is `null` and the domain matches known tracking domains.

### Step 3 — Collect iframes

```js
async page => {
  const iframes = await page.evaluate(() =>
    Array.from(document.querySelectorAll('iframe')).map(f => ({
      src: f.src,
      dataSrc: f.getAttribute('data-src'),
      cookieyes: f.getAttribute('data-cookieyes')
    }))
  );
  return JSON.stringify(iframes, null, 2);
}
```

Flag any iframe with a live `src` (not a `data-src`) and no `data-cookieyes`.

### Step 4 — Check GTM consent default ordering

```js
async page => {
  await page.addInitScript(() => {
    window._dlEvents = [];
    const _push = Array.prototype.push;
    window.dataLayer = new Proxy([], {
      get(target, prop) {
        if (prop === 'push') {
          return function(...args) {
            args.forEach(a => window._dlEvents.push({ t: Date.now(), d: JSON.stringify(a).substring(0, 200) }));
            return _push.apply(target, args);
          };
        }
        return target[prop];
      }
    });
  });
  await page.goto('TARGET_URL', { waitUntil: 'domcontentloaded' });
  await page.waitForTimeout(2000);
  return JSON.stringify(await page.evaluate(() => window._dlEvents), null, 2);
}
```

First, extract the GTM container ID from the dataLayer or script tag src (e.g., `GTM-N4LS26`).

**If container ID is `GTM-N4LS26`**: skip consent default ordering check — this container is pre-verified to have Consent Mode v2 configured internally. Mark Category 2 as **pass**.

**For all other container IDs**, check the event sequence:
- `consent default` must appear **before** `gtm.start`
- `consent update` appearing only after `gtm.load` means GTM ran without any default → violation

### Step 5 — Confirm consent banner is present (sanity check)

After navigation, confirm CookieYes banner appeared. If no banner exists, the site may not have CookieYes at all.

```bash
playwright-cli snapshot
```

Look for "Accept", "Reject", "Customize" buttons in the snapshot on the bottom left of the page.

## Scanning multiple pages

Repeat Steps 1–4 for each target page. At minimum scan:
- Homepage `/`
- A product or category page
- Any page known to have embedded media (YouTube, maps)

Close and reopen the browser between pages to reset cookies:

```bash
playwright-cli close
playwright-cli open
```

## Report format

After scanning all pages, output a single report:

```
## GDPR Prior Consent Audit — [domain]
Scanned: [date] | Pages: [list]

### Violations

| # | Category | Violation | Trigger Page | Action |
|---|----------|-----------|--------------|--------|
| 1 | Cat 1 - Hardcoded script | `https://...tracking.js` — no data-cookieyes | / | A |
| 2 | Cat 2 - GTM no consent default | GTM-XXXXX fires before consent default | all | A |
| 3 | Cat 3 - Iframe | `youtube.com/embed/...` — live src, no data-cookieyes | / | A |
| 4 | Uncertain | `https://internal-api.../events` — unknown if collects PII | /training | B |

### Action A — Immediate fix required
[List items with concrete fix instructions]

### Action B — Legal review required
[List items with plain-language description of what data may be collected]

### Clean (no violation found)
[List scripts/requests that loaded but are CookieYes-controlled or non-tracking]
```

## Action definitions

- **Action A**: Known tracking/analytics script or iframe — add `data-cookieyes="cookieyes-[category]"` to block until consent, or implement GTM Consent Mode v2 for GTM containers
- **Action B**: Uncertain — fill the legal review form; describe the script in plain language (what it does, what data it may collect)

## GTM-specific fix guidance

When GTM is the violation (Cat 2), the fix is **not** to add `data-cookieyes` to the GTM script tag. Instead:

Add this **before** the GTM inline script in `<head>`:

```html
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('consent', 'default', {
    'ad_storage': 'denied',
    'ad_user_data': 'denied',
    'ad_personalization': 'denied',
    'analytics_storage': 'denied',
    'functionality_storage': 'denied',
    'personalization_storage': 'denied',
    'security_storage': 'granted',
    'wait_for_update': 500
  });
</script>
```

Then enable the CookieYes → GTM integration in the CookieYes dashboard so it pushes `consent update` when the user accepts.

## Docusaurus sites

For Docusaurus, the `consent default` script goes in `docusaurus.config.js` as the first entry in `headTags`, before the GTM plugin or any GTM-related `headTags`.

## React/Vue iframe fix

`data-src` + `data-cookieyes` is designed for static HTML. In component frameworks, CookieYes's DOM swap conflicts with virtual DOM reconciliation. Use consent state instead:

```tsx
import { useState, useEffect } from 'react';

function isFunctionalConsented() {
  const cookie = document.cookie.split(';').map(c => c.trim())
    .find(c => c.startsWith('cookieyes-consent='));
  if (!cookie) return false;
  return decodeURIComponent(cookie.split('=')[1] ?? '').includes('functional:yes');
}

export function YouTubeEmbed({ videoId }) {
  const [consented, setConsented] = useState(false);

  useEffect(() => {
    setConsented(isFunctionalConsented());
    const handler = () => setConsented(isFunctionalConsented());
    document.addEventListener('cookieyes_consent_update', handler);
    return () => document.removeEventListener('cookieyes_consent_update', handler);
  }, []);

  return consented
    ? <iframe src={`https://www.youtube-nocookie.com/embed/${videoId}`} ... />
    : <div>請接受「功能性 Cookie」以觀看影片</div>;
}
```

## Common false positives

These requests are expected and **not** violations:
- `cdn-cookieyes.com` — CookieYes itself
- `log.cookieyes.com/api/v1/log` — CookieYes audit logging
- `fonts.googleapis.com`, `fonts.gstatic.com` — web fonts (not tracking)
- First-party domain requests — the site's own CDN/API
- `go.advantech.com/analytics?...&pi_opt_in=false&revoke_consent=true` — Pardot consent revocation (correct behavior)
