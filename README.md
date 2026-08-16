# Write-URL-query-strings-2-cookies

A custom [Google Tag Manager (GTM)](https://tagmanager.google.com/) **tag** template (`.tpl`) that captures any of ~28 pre-defined (plus custom) URL query string parameters — UTM parameters and a broad list of ad-platform click IDs — and writes each one to its own first-party cookie, using GTM's sandboxed cookie APIs so no extra Content Security Policy (CSP) `script-src`/SHA-256 allowances are needed beyond whatever your site already requires for `gtm.js`.

Per the repository's own description, this is the **current, consolidated evolution** of an idea the author first built years earlier (inspired by an Analytics Mania post on link decoration): rather than needing three separate tags per tracked parameter (as in this author's earlier `Cookie Creator`-based UTM/click-ID containers), this single tag template handles an entire configurable list of parameters at once. It's also the shared building block used by this author's [decorate-link-URL-for-improved-campaign-measurement](https://github.com/drewspen/decorate-link-URL-for-improved-campaign-measurement) container, where it's referenced by name.

## What it captures

The template ships with **28 pre-built checkboxes**, all checked (enabled) by default:

**Standard UTM parameters:** `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`, `utm_id`, `utm_source_platform`, `utm_creative_format`, `utm_marketing_tactic`

**Ad-platform click/attribution IDs:** `gclid`, `dclid`, `gclsrc`, `gad_source`, `gbraid`, `wbraid` (Google), `fbclid`, `igshid` (Meta/Instagram), `twclid` (X/Twitter), `msclkid` (Microsoft), `li_fat_id` (LinkedIn), `rdt_cid` (Reddit), `ttclid`, `ttcid` (TikTok), `srsltid` (Google Shopping/Merchant Center), `ScCID`, `ScCid` (Snapchat — note both a capital- and mixed-case variant are separately listed), `epic` (Epic Games), and a generic `cid`

Beyond the checkbox list, an **`addUrlQueryString`** table lets you type in any additional parameter names (validated against `^[a-zA-Z0-9._~-]{0,39}$`) not already covered — matching the repository description's point that you can "add new ones not listed."

## How it works

The template's sandboxed JavaScript does the following on each page load where the tag fires:

1. Builds the effective list of tracked parameter names — every checked checkbox, plus anything typed into the `addUrlQueryString` table.
2. Reads the page's full raw query string via `getUrl('query')`, optionally URI-decoding it first if `uriDecode` is set to `Yes`.
3. **Manually splits the query string** on `&` and then `=` to get each key/value pair (rather than using per-key lookups), keeping only pairs that split into exactly two pieces with both a non-empty key and value.
4. For each pair whose key is in the tracked-parameter list, checks whether a cookie of that same name (via `getCookieValues`) **already exists** — and **only writes the cookie if it doesn't**. If it does, or if the query parameter isn't present at all, nothing happens for that parameter.
5. Writes qualifying cookies via GTM's sandboxed `setCookie` API, with `domain`/`path` set from the `rootDomain` field, and (if `cookieDuration` is `persistentCookie`) a `max-age` computed from `persistentCookieExpires` days; if `cookieDuration` is `sessionCookie`, no `max-age` is set at all (browser-default session lifetime).

## ⚠️ Important nuance: getting both first-touch and current-touch value

Getting both first-touch and current-touch values: two tag instances + `cookieNameSuffix`
Because a single tag instance can only produce one duration behavior per parameter, the template's `cookieNameSuffix` field exists specifically so you can create **two separate tags** from this template — e.g., one configured with `cookieDuration: persistentCookie` and a suffix like `_1st`, another with `cookieDuration: sessionCookie` and no suffix — so both variants can write distinctly-named cookies (`utm_campaign_1st` and `utm_campaign`) without colliding. This matches the repository description's third listed feature directly.

## ⚠️ A parsing edge case: values containing a literal `=` are silently dropped

The manual `pair.split('=')` step only accepts pairs where the split produces **exactly two** elements (`pair.length === 2`). If a parameter's actual value itself contains an unencoded `=` character — which can happen with some base64-encoded click IDs (`gbraid`, `wbraid`, and `li_fat_id` are documented as base64url-encoded values in some implementations) if they aren't properly percent-encoded in the URL — that pair will silently fail this check and **not be captured at all**, with no error or console message. This is a subtle failure mode worth testing for if you notice a particular click ID never seems to get captured despite clearly being present in the URL.

## Template fields

| Field | Type | Default | Purpose |
|---|---|---|---|
| `rootDomain` | Select (variable reference) | `{{Root Domain}}` | The cookie `Domain` attribute — swap for `{{Page Hostname}}` or similar if targeting a single subdomain instead of the whole root domain. |
| `cookieDuration` | Radio | `sessionCookie` | `sessionCookie` (no explicit expiration) or `persistentCookie` (expires after `persistentCookieExpires` days). |
| `persistentCookieExpires` | Text (positive number) | `365` | Days until expiration, only used when `cookieDuration` is `persistentCookie`. |
| `uriDecode` | Radio | `No` | Whether to URI-decode the raw query string before parsing. |
| `addUrlQueryString` | Simple table | *(empty)* | Additional parameter names to track, beyond the 28 pre-built checkboxes. |
| `cookieNameSuffix` | Text (regex `^[a-zA-Z0-9_]{0,9}$`) | *(empty)* | Optional suffix appended to every cookie name this tag writes — lets a session-scoped and persistent-scoped tag instance coexist without name collisions. |
| `targetedUrlQueryString` (group of 28 checkboxes) | Checkbox group | All checked | Toggles which of the pre-built parameter names are actually tracked. |

## Requested permissions

- **`get_cookies`** — `cookieAccess: "any"` — needed to check whether a cookie already exists before deciding whether to write it.
- **`set_cookies`** — `allowedCookies` is **wildcarded** (`name`, `domain`, and `path` all set to `"*"`; `secure`/`session` both `"any"`) — meaning, unlike several of this author's other custom templates (which pin an explicit list of specific cookie names/domains), **this template's permission grant does not itself restrict which cookie names or domains it can write to**. The actual scope of what gets written is controlled entirely by the tag's own field configuration (the checkbox list and `addUrlQueryString` table), not by GTM's permission system. Worth keeping in mind if you're auditing this template purely from its Permissions tab — you won't see the specific cookie names there the way you would with this author's other, more narrowly-scoped templates.
- **`get_url`** — `urlParts: "any"`, `queriesAllowed: "any"` — needed to read the page's query string via `getUrl('query')`.
- **`logging`** — console logging, restricted to the `debug` environment — though the code requires `logToConsole` but never actually calls it, so no console output is currently produced regardless of environment.

## Earlier repositories this solution obsoletes

This template is a direct, consolidated successor to a family of this author's earlier GTM container exports, each of which implemented URL-query-string-to-cookie persistence using the older, per-parameter **`Cookie Creator`** template — requiring three separate tags (`Create`, `Create 1st`, `Rewrite 1st`) *for every individual parameter tracked*. This single tag template replaces all of that repetitive tag/trigger/variable scaffolding with one configurable tag. The specific earlier repositories superseded by this approach are:

- [gtm-persist-utm-url-query-strings-1st-party-cookie](https://github.com/drewspen/gtm-persist-utm-url-query-strings-1st-party-cookie) — the standard `utm_source`/`utm_medium`/`utm_campaign`/`utm_content`/`utm_term`/`utm_id` parameters, plus `utm_marketing_tactic`, `utm_source_platform`, and `utm_creative_format`.
- [gtm-persist-original-source-landing-fbclid](https://github.com/drewspen/gtm-persist-original-source-landing-fbclid) — the `fbclid` portion of this container (its site-referral-source and landing-page capture logic, which don't come from URL query parameters, aren't superseded by this template).
- [gtm-persist-various-social-media-marketing-url-query-strings](https://github.com/drewspen/gtm-persist-various-social-media-marketing-url-query-strings) — `fbclid`, `dclid`, `rdt_cid`, `srsltid`, `twclid`, `msclkid`, `li_fat_id`, `gclid`, and `twclid`.
- [gtm-persist-meta-facebook-url-query-strings](https://github.com/drewspen/gtm-persist-meta-facebook-url-query-strings) — `fbclid` plus Meta's ad-creative parameters (`ad_id`, `ad_name`, `adset_id`, `adset_name`, `campaign_id`, `campaign_name`, `placement`, `site_source_name`).
- [gtm-persist-google-ads-url-query-strings](https://github.com/drewspen/gtm-persist-google-ads-url-query-strings) — `gclid`, `dclid`, `srsltid`, `gad_campaignid`, and `gad_source`.

Between them, these five earlier containers required **93 tags, 93 triggers, and roughly 116 variables** combined (27+9+24+27+15 tags/triggers, plus their supporting per-parameter variables) to cover, in aggregate, a similar — and in some cases smaller — set of parameters than this one template now handles by itself with a single tag and a checkbox list. If you're currently running one or more of those older containers, migrating to this template lets you delete all of that per-parameter tag/trigger/variable scaffolding in favor of one (or two, if you need both session and persistent variants) tag instance — though note the semantic differences called out above (first-seen-only vs. first-touch-plus-current-touch, no expiration-renewal) before treating this as a drop-in behavioral replacement rather than just a configuration-surface simplification.

## Getting started

### Import into Google Tag Manager

1. In your GTM container, go to **Templates** → **Tag Templates** → **New**.
2. Click the **⋮** (more actions) menu → **Import**.
3. Select the `Write URL query strings 2 cookies.tpl` file from this repository.
4. Save the template.

### Create a tag from the template

1. Go to **Tags** → **New**.
2. Choose **Write URL query strings 2 cookies** as the tag type.
3. Set **rootDomain** to `{{Root Domain}}` (or your own equivalent variable) — or override with a specific hostname variable if you want subdomain-only scoping.
4. Choose **cookieDuration** — `sessionCookie` for current-visit-only capture, or `persistentCookie` (with a chosen `persistentCookieExpires`) for longer-lived, first-touch-style capture.
5. Uncheck any of the 28 pre-built parameters you don't want tracked, and/or add custom parameter names via `addUrlQueryString`.
6. If you plan to also run a second instance of this tag with a different `cookieDuration` for the same parameters, set a distinct `cookieNameSuffix` on at least one of the two tags.
7. Set the appropriate trigger — typically **All Pages** or GTM's **Initialization** trigger, so capture happens as early as possible on every page load.
8. Save, preview, and test the tag before publishing.

### Verify it's working

- Use GTM's **Preview/Debug** mode with a URL containing several tracked parameters (e.g., `?utm_source=test&gclid=abc123`) and confirm the corresponding cookies are set with the expected names, values, domain, and path.
- Reload the same URL with **different** parameter values and confirm the cookies **do not change** — this is expected, first-write-wins behavior, not a bug.
- If using two tag instances (session + persistent) with a `cookieNameSuffix`, confirm both cookies are set independently without overwriting each other.
- If a specific click ID never seems to get captured, check whether its value might contain an unencoded `=` character (see the parsing edge case above).

## Notes

- This template was created on 3/23/2026 — making it one of the more recent additions to this author's collection, and the same template referenced by name in the [decorate-link-URL-for-improved-campaign-measurement](https://github.com/drewspen/decorate-link-URL-for-improved-campaign-measurement) container.
- This is an unofficial, community-built template and is not published or endorsed by Google, Meta, Microsoft, LinkedIn, Reddit, X, Snapchat, TikTok, or Epic Games — despite covering parameters specific to all of them. Always review sandboxed template code and requested permissions — including the wildcarded cookie-write scope noted above — before relying on this in a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this template in a commercial or redistributed context.
