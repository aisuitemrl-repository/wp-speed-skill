# WordPress speed optimization: WooCommerce / Elementor / XStore on Nestify

**Status: DRAFT v0.1.** Built from the philipsaccessories.in work on 2026-11-23. Not yet proven end-to-end. Each step marked *(unverified)* still needs confirmation on a real site. Refine after every site, and record the changes in RUNLOG.md first.

## When to use
A WordPress site on Nestify hosting (same server as the other sites in the portfolio) where the user wants faster load times, better Core Web Vitals, or fewer slow database or PHP calls.

## Ground rules
1. **Measure before changing.** Record a baseline first (Lighthouse, admin timings, visitor cache check).
2. **Ask before any live change.** List the proposed changes and wait for the user to approve each one. Changing settings on the live site needs approval, even when the user has said "do it".
3. **Backup before changes.** Ask the user for a Nestify backup first.
4. **One change at a time,** each followed by a check and a re-measure. If a page breaks, undo that one change.
5. **Log every step** in RUNLOG.md, including failed attempts and corrections. Don't reconstruct the history later.
6. **Never write secrets into the log or chat.** Application Passwords, licence keys and Redis prefixes stay out. Describe them by shape or by a fingerprint.

## Phase 0: Access
1. **Allow the host.** The egress proxy blocks unlisted hosts. Check with `curl -I https://<site>/`. A `403` on CONNECT or `ENOTFOUND` means the host isn't allowed. Add it to the environment allowlist (`update_environment`, exact host name only, no wildcards).
2. **Create a helper user** in WordPress: a dedicated Administrator account (not the owner's login), with an Application Password.
3. **Save the credential** with `request_credential` (host = the site's exact hostname, suggested_type = basic). Never ask for it in chat.
4. **Verify:** `GET /wp-json/wp/v2/users/me?context=edit` must return the helper user's `roles` including `administrator`. A `401 rest_not_logged_in` means re-enter the credential. The cause of the first 401s on philipsaccessories.in was not confirmed.
5. **Revoke at the end:** delete the Application Password and the helper user when the work is complete.

## Phase 1: Measure (no changes)
1. **Lighthouse (self-run).** `npm install lighthouse@12` in a scratch folder, then:
   ```
   npx lighthouse https://<site>/ --preset=desktop  (omit for mobile)
     --chrome-path=/opt/pw-browsers/chromium-1243/chrome-linux64/chrome
     --chrome-flags="--headless --no-sandbox --disable-gpu"
     --only-categories=performance,accessibility,best-practices,seo
     --output=json --output=html --output-path=<dir>/<name>
   ```
   The PageSpeed Insights API may be out of quota; don't rely on it.
2. **Read the key audits:** `largest-contentful-paint-element`, `render-blocking-resources`, `unused-css-rules`, `unused-javascript`, `dom-size`, `mainthread-work-breakdown`, `server-response-time`, `uses-responsive-images`, `modern-image-formats`.
3. **Admin-view timings (logged in, Query Monitor).** Homepage, one in-stock product, shop, cart, checkout. Read the admin-bar figures on a second load: time, queries, DB time. Query Monitor reports Object Cache status. Record these as the worst case.
4. **Visitor cache check:** a logged-out `GET` should show the `nestify-cdn-cache` header (`HIT` or `MISS`). Note it for each page.
5. **Plugins inventory** (REST `GET /wp-json/wp/v2/plugins`). Flag these:
   - Active licence-cracking or admin tools (for example `yith-activator`). Check where they came from.
   - Duplicate feed, form or import plugins.
   - Cache plugins whose caches are switched off.
   - Performance plugins already installed (for example Core Web Vitals Booster, a custom `pagespeed` plugin). Read their settings before changing anything, since they may already be doing the work.
6. **Updates list** (REST or Dashboard > Updates). Note any flagged "Not tested" for the current WordPress version.

**Interpretation**
- High DB time and query count → object cache (Redis) will help.
- Little DB time but a long total → PHP is the bottleneck: page builder, theme, plugin code, or HTTP calls.
- Largest element is **text**, not an image → image preloading won't help. Look at render-blocking CSS and web fonts.
- Largest element is an **image** → preload it (with `fetchpriority="high"`) and fix image format and size.

## Phase 2: Fixes (each needs approval)
Ordered by typical risk, lowest first.

1. **Images to WebP (Smush Pro, Local WebP).** *Verified on philipsaccessories.in, pending re-check.*
   - Smush Pro → Local WebP on. Keep CDN off if the owner doesn't want images on a third-party server.
   - Run Bulk Smush. If background processing shows a **loopback error**, the host (Nestify) must fix it. This is not a setting in wp-admin.
   - Before enabling a WebP plugin, check the server type. On Nginx hosts, the plugin may need rewrite rules (`rewrites_not_executed`). Converter for Media failed for this reason.
   - Verify: a `.webp` copy exists, and an `Accept: image/webp` request returns `image/webp`.
2. **Redis object cache.** *Partly verified.*
   - Install and activate the Redis Object Cache plugin. It's already installed on philipsaccessories.in.
   - Settings > Redis: confirm "Reachable". Then click **Enable Object Cache**.
   - Before enabling, confirm the key prefix is unique to this site (Nestify sets `WP_REDIS_PREFIX`). The Diagnostics panel may not show it, so check with the host.
   - Expected effect: fewer repeated queries (`get_option`, `wp_prime_option_caches`). PHP time is unchanged.
   - Undo: disable the object cache, or deactivate the plugin.
3. **Unused Elementor widgets off.** Element Manager (if available) → scan → switch off zero-use widgets. The screen may redirect in some setups; use the user's own browser if so. Undo: Element Manager > Enable All.
4. **Idle plugins off.** Deactivate, don't delete. Check each one first with the front end (search for `wpcf7-form`, `wpforms-form`, and similar).
5. **Plugin and theme updates.** One at a time, with a backup, starting with minor and low-risk releases. Leave page builders, theme cores and feed plugins for last. Test checkout after WooCommerce updates.
6. **Render-blocking CSS and fonts** *(unverified on this site).*
   - Elementor → Settings → Advanced: check the Google Fonts loading option. If the site has no swap setting, skip it.
   - Autoptimize, Aggregate and Optimize CSS only. Keep Defer and Inline off at first. **Note:** it does not remove unused CSS.
   - Unused CSS per page needs Asset CleanUp (manual, per page) or a paid tool.
7. **Nestify panel (user action):** Browser Cache on. Flush the page cache once after the changes, then avoid frequent full purges.

## Phase 3: Verify and report
- Re-run the Phase 1 measurements. Compare with the baseline, and only claim a gain larger than the normal variation (about ±0.5 s on page time).
- Test the key pages: homepage, one in-stock product (Add to Cart, gallery), shop, cart and checkout, and any form pages. Check Query Monitor for new PHP errors.
- Before reporting a break, confirm the cause. Common false alarms:
  - No Add to Cart button usually means out of stock.
  - Empty-looking widgets are often lazy-loaded or animated.
  - `admin-ajax.php` returning 0 bytes for cookieless fetches is a test artifact on Nestify.
- Report a before/after table, each change with its undo, and the items only the owner can do (licence activation, replacing cracked plugins, content fixes).

## Browser checks (Claude in Chrome, when connected)
- The Chrome tools are only available through the desktop app's Remote Control folder. In a cloud session they may not be connected.
- Run read-only checks one prompt at a time: Redis status, Query Monitor on the homepage and a product page, the element scan, and the updates list.
- The Chrome tool output filters text that looks like query strings or cookies. Ask for counts and shapes, not raw URLs or secrets.

## Known pitfalls
- **Redis plugin active but object cache off.** Active plugin ≠ working cache. Check the Overview status.
- **Query Monitor "Object Cache: Not available"** and an empty Components column are both caused by the missing `db.php` drop-in. They clear once the cache is enabled.
- **Admin-view timings are the worst case.** Visitors get the CDN cache. Report both.
- **Element Manager may redirect** back to Elementor settings. Don't loop on it.
- **Smush background processing** needs the host's loopback to work. Ask the host early.
- **Autoptimize** minifies and aggregates. It does not remove unused rules.
- **Inactive plugins** cost nothing at runtime. Removing them is housekeeping only.
- **Hero preload** helps only when the largest element is an image.

## Per-site checklist
- [ ] Host allowed in the environment allowlist
- [ ] Helper user created, credential verified (`users/me`)
- [ ] Nestify backup taken (user)
- [ ] Baseline: Lighthouse mobile and desktop saved
- [ ] Baseline: admin timings (home, product, cart) saved
- [ ] Baseline: visitor cache headers recorded
- [ ] Plugin inventory and update list recorded
- [ ] Performance plugins already present: settings read
- [ ] Largest element identified (text or image)
- [ ] Each fix approved, applied, checked, and logged
- [ ] Re-measure and report
- [ ] Helper user and Application Password revoked

## Changelog
- v0.1 (2026-11-23): draft from philipsaccessories.in. Smush loopback fixed by Nestify. Redis reachable, object cache off. Updates not applied. Smush conversion in progress.
