# ApeRate — Security Review

**Scope:** `index.html` (entire file — the whole app is this one static, client-only page)
**Method:** Manual review (no backend, no git history to diff — reviewed the full file as-is)
**Date:** 2026-07-04

## Summary

ApeRate is a fully client-side SPA with no server, no auth, and no secrets — that rules out
whole classes of bugs (SQLi, SSRF, auth bypass, session handling). React's default text
escaping means comment text and ape metadata cannot be used for stored/reflected XSS. No
`dangerouslySetInnerHTML`, `innerHTML`, `eval`, or `Function()` calls exist anywhere in the file.

The real findings are all supply-chain / hardening items: this page trusts four third-party
CDN origins with no integrity checking, and a couple of data-flow patterns are safe only
because today's data is a hardcoded config — they'd need attention before ever accepting
dynamic or user-submitted ape entries.

## Findings

### Medium

1. **No Subresource Integrity (SRI) on any third-party `<script>`** — `index.html:15, 103-105`
   Tailwind (`cdn.tailwindcss.com`), React, ReactDOM, and Babel Standalone are all loaded with
   no `integrity` hash. If any of the three CDNs (unpkg, Tailwind, jsDelivr-backed fonts) is
   compromised or MITM'd, arbitrary JS runs with full page access.
   **Fix:** add `integrity="sha384-…"` + `crossorigin="anonymous"` to each `<script>` tag.
   (`openssl dgst -sha384 -binary file.js | openssl base64 -A`)

2. **Tailwind Play CDN used for the styling engine** — `index.html:15`
   `cdn.tailwindcss.com` prints its own console warning: "should not be used in production."
   It ships a JS build step that runs in-browser on every load — another full-trust script
   dependency, and slower than shipping static CSS.
   **Fix:** for anything beyond a quick demo, precompile Tailwind's CSS via the CLI/PostCSS
   and drop the CDN script entirely.

3. **Babel Standalone pinned only to a floating major version** — `index.html:105`
   `@babel/standalone@7` resolves to *whatever the latest 7.x.x is* — unlike React/ReactDOM,
   which are pinned to exact patch versions (`@18.3.1`). A future 7.x release (even a
   legitimate one) can change transform behavior without warning, and there's no way to
   audit what actually shipped.
   **Fix:** pin to an exact version, e.g. `@babel/standalone@7.26.4`.

### Low

4. **No Content-Security-Policy** — nothing in `<head>` (`index.html:1-89`)
   No CSP meta tag restricting `script-src`/`img-src`/etc. Given the app already depends on
   inline `<script>` blocks and inline styles, a strict CSP is hard to add without
   `unsafe-inline`, but even a `script-src` allow-list of the four origins in use would add
   defense-in-depth if a dependency is ever compromised.

5. **`imageUrl`/`sourceUrl` flow into `src`/`href` with no scheme validation** —
   `index.html:175-177` (`withParams`, `heroSrc`, `thumbSrc`) and `index.html:809-820`
   (`<a href={ape.sourceUrl}>`)
   Safe today only because `APE_DATABASE` is a hardcoded config array. The schema is
   explicitly designed to be "incredibly easy to extend" — the moment ape entries come from
   anything less trusted (an admin form, CSV import, query param, CMS), an attacker-supplied
   `sourceUrl` of `javascript:...` would sit directly in an `<a href>` with no allow-listing.
   **Fix:** if/when this data ever becomes dynamic, validate `imageUrl`/`sourceUrl` are
   `http(s)://` before use, and consider host allow-listing for `imageUrl`.

6. **Parsed `localStorage` state isn't validated** — `index.html:208-225` (`usePersistentState`)
   `JSON.parse` output is trusted as-is with no shape/type check (e.g., `ratings` values
   aren't confirmed to be integers 1–10). Not exploitable for XSS (React still escapes
   render output), but any code sharing the origin — or manual tampering via devtools — can
   corrupt state and crash the UI.
   **Fix:** validate parsed values match expected shape before using them; fall back to
   `initialValue` on mismatch.

7. **No clickjacking protection** (`X-Frame-Options` / CSP `frame-ancestors`)
   Can't be fixed from inside the HTML file — these are HTTP response headers. Flagging so
   whoever deploys this sets them at the server/hosting layer.

## Explicitly checked, no issue found

- No `dangerouslySetInnerHTML`, `innerHTML`, `eval`, or `Function()` usage anywhere.
- Comment text and ape metadata render through JSX (`{value}`), which auto-escapes — no
  stored/reflected XSS via the comment feed.
- The one `target="_blank"` link already carries `rel="noopener noreferrer"`.
- No hardcoded secrets, API keys, or credentials.
- No server-side component, so SQLi/SSRF/auth-bypass classes don't apply.
- No prototype-pollution vector in the `{...prev, [id]: value}` spreads — computed property
  keys (as opposed to literal `__proto__`) don't trigger the special `[[Prototype]]` setter
  per spec, so a crafted `id` of `"__proto__"` would just become a normal own property.

## Priority order

1. Pin Babel to an exact version (quick, no behavior change).
2. Add SRI hashes to all four third-party `<script>` tags.
3. Replace the Tailwind Play CDN with a precompiled stylesheet before any real deployment.
4. Add scheme validation to `imageUrl`/`sourceUrl` before `APE_DATABASE` ever accepts
   non-hardcoded entries.
5. Validate parsed `localStorage` shape; set deployment-layer headers (CSP/X-Frame-Options).
