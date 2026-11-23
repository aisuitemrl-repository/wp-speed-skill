# WordPress speed work: run log

Append-only. Add an entry for every step, including failed attempts and corrections. Never edit past entries; add a correction entry instead.

Entry format:
- **Date / site / step**
- **Action** (what was done, and by whom: Claude via REST, or user via Chrome/wp-admin)
- **Result** (verified how)
- **Undo** (how to reverse it)
- **Lesson** (if any)

---

## 2026-11-23 · philipsaccessories.in (WooCommerce, XStore theme, Elementor, Nestify host)

### Access
- **Site blocked by the egress proxy.** Claude could not reach the host until it was added to the environment allowlist. Done on the user's personal Default environment (`add_allowed_hosts`). **Lesson:** every new site needs its host added before anything works. Check with a plain `curl` first; a 403 CONNECT or `ENOTFOUND` means the host is not allowed.
- **WordPress Application Password.** Saved for the host via `request_credential` (basic, username + Application Password). The first two saves returned 401 on `/wp-json/wp/v2/users/me?context=edit`; a third save worked (user account `claude-optimizer`, role Administrator). **Lesson:** verify access with the `users/me` call before any other step. On 401, ask for the credential again. The cause of the first 401s was not confirmed.
- **Undo:** revoke the Application Password and delete the `claude-optimizer` user when the work ends.

### Measurement
- **PageSpeed Insights API:** quota exhausted (limit 0 per day). Not usable.
- **Lighthouse (self-run):** works. Install `lighthouse@12` into a scratch folder with npm, then run with the preinstalled Chromium: `--chrome-path=/opt/pw-browsers/chromium-1243/chrome-linux64/chrome --chrome-flags="--headless --no-sandbox --disable-gpu"`. Use `--preset=desktop` for desktop.
- **Baseline (Lighthouse, homepage, before changes):**
  - Mobile: performance 84, LCP 2.5 s, TBT 330 ms, CLS 0, render-blocking 880 ms, unused CSS 141 KiB, 2,789 DOM elements, main-thread work 5.3 s.
  - Desktop: performance 92, LCP 0.7 s, CLS 0.141.
  - LCP element on both: a **heading (text)**, not an image.
- **Admin-view baseline (logged in, Query Monitor, second load):**
  - Homepage: 10.8 s, 685 queries, 1.0 s DB.
  - Product page (DLC3104U, in stock): 18.6 s, 477 queries, 1.1 s DB. Top callers: `get_option` (151 queries, 0.58 s), `wp_prime_option_caches`, `WP_Post::get_instance`.
  - Object cache: not in use. Query Monitor shows "Not available" because its `db.php` symlink could not be created.
- **Lesson:** logged-in admin timings are the worst case. Logged-out visitors are served from the Nestify CDN cache, so check that separately.

### Plugins and images
- **Converter for Media (`webp-converter-for-media` 6.7.2)** installed and activated. It showed "rewrites_not_executed": Nginx rewrite rules are needed. **Deactivated** again. **Lesson:** on Nginx hosts, check whether a WebP plugin needs server rules before installing it.
- **Smush Pro (3.16.9, was inactive)** activated. User chose CDN off (images stay on our server) and Local WebP as the method.
- **Smush background processing** failed with a loopback error ("Your site seems to have an issue with loopback requests"). Retry did not help. **Nestify fixed it on their side.** Bulk conversion was running at the time of the last check; no WebP copies were served yet. **Pending:** re-check once Smush reports the run as complete, and confirm Local WebP is on.
- **Lesson:** Smush loopback problems on Nestify are fixed by Nestify support, not in wp-admin.

### Redis
- `redis-cache` 3.0.0 is installed and active.
- Redis Object Cache page: **Connection reachable**, filesystem writable, **object cache not enabled** (drop-in not installed).
- **Key prefix:** not shown in Overview or Diagnostics. Diagnostics shows `WP_CACHE_KEY_SALT`, which is a salt, not `WP_REDIS_PREFIX`. User confirmed the prefix is different per site, as on the earlier site.
- **Status:** enabling the cache is **not done**. Waiting for the baseline measurements and the user's go-ahead.

### Plugin and theme updates (available, nothing applied)
- 15 plugins and 1 theme (XStore 9.7.6 → 9.7.12).
- Flagged "Not tested" for WordPress 7.1.5: Combo Offers WooCommerce, Core Web Vitals Booster, PRO Elements, XStore Core.
- CTX Feed needs its own sequence (CTX Feed first, then CTX Feed Pro, then regenerate feeds).
- Contact Form 7 warns about the site's PHP version.
- **Core Web Vitals Booster 1.2.2** is installed. Purpose and settings not yet checked. Check before changing anything else.
- **Pending:** nothing applied. Plan: backup first, then Redis update, then cache, then the rest one at a time with a check after each.

### Chrome browser route
- The Claude in Chrome tools are not connected to this cloud session. The user has to connect through the desktop app (Remote Control folder), and the folder was reported "not_available" even after rename and restart. Not resolved at the time of writing.
- **Workaround that worked:** user runs read-only checks in Chrome from a prompt list, one prompt at a time, and pastes results back. Claude does the analysis and proposes changes.
- **Lesson:** Chrome-only checks can't see HTTP headers or the page source, so use REST and `curl` for those.

### Element Manager
- Opening Elementor > Element Manager redirected back to Elementor > Settings > General. Skipped; the user can check it in their own browser if needed.

### Corrections (wrong claims made earlier, now withdrawn)
- Removing inactive plugins does **not** speed up the site. It's only housekeeping.
- The `mrlightglobal.in` reference is **menu links** to another store, not a third-party script.
- **Autoptimize does not remove unused CSS.** "Optimize CSS Code" minifies, and "Aggregate CSS-files" combines files. Neither removes rules. Unused CSS needs per-page control (Asset CleanUp) or a paid tool.
- The **hero-image preload** used on the flashlights site does not apply here: the LCP element is text, not an image.

### Reference site: philipsflashlights.in (for comparison only)
- Plugins include a custom `pagespeed` plugin (1.3), Smush Pro 3.21.1, Redis Cache 3.0.1, Query Monitor, Insert Headers and Footers.
- Homepage preloads the hero WebP with `fetchpriority="high"`. Images are served from the site's own uploads.
- Earlier optimization steps are not recorded anywhere that Claude can read. The user says they were done mainly in Chrome, in a session named "Session Overflow Issues" that Claude cannot access.
- **Lesson:** record every step as we go. Don't depend on a later reconstruction.

---

## Open items (as of this entry)
1. Smush bulk conversion: re-check WebP serving once Smush reports complete.
2. Confirm Local WebP is on.
3. Core Web Vitals Booster: read its settings.
4. Enable Redis object cache (needs user go-ahead).
5. Plugin and theme updates, one at a time, after a Nestify backup.
6. Element Manager widget scan (optional).
7. Re-run Lighthouse and the admin baselines after each change.

---

## 2026-11-23 13:06 UTC · philipsaccessories.in · scheduled WebP re-check

**Action:** Read-only re-test by Claude (scheduled task `trig_01Q8K1aQog3TFaCjEPWNCFhL`). Nothing changed on the live site.

**Result:**
- Test image `SWV5531-01-2048x2048.jpg`: 111,817 B, `image/jpeg` for both plain and WebP-capable requests. Unchanged.
- Test image `Charger-1024x402.png`: 58,556 B, `image/png` for both requests. Unchanged.
- No `.webp` copy exists for the test image (both `.jpg.webp` and `.webp` naming return 404).
- Homepage: HTTP 200, 617,893 B, 1.34 s (earlier 1.10 to 2.23 s; within normal variation).
- Smush Pro (`wp-smush-pro/wp-smush`): active, version 3.16.9.

**Interpretation:** Smush bulk conversion has not produced serving-ready WebP files yet, or Local WebP is not enabled, or the run is still in progress. The REST API cannot show Smush's bulk progress or the Local WebP toggle, so this can't be told apart from here.

**Pending (needs user, in wp-admin):** confirm Smush > Bulk Smush status and whether Local WebP is on. Nothing is applied until approved.

**Next:** scheduled re-check in 60 minutes.

---

## 2026-11-23 14:14 UTC · philipsaccessories.in · second scheduled WebP re-check

**Action:** Read-only re-test by Claude (scheduled task `trig_01N4v7LCjBMxgn84TN3vjqeN`). Nothing changed on the live site.

**Result:**
- `SWV5531-01-2048x2048.jpg`: 111,817 B, `image/jpeg` for both plain and WebP-capable requests. Unchanged since 11:x UTC.
- No `.webp` copy exists (both `.jpg.webp` and `.webp` naming return 404).
- Homepage: HTTP 200, 0.60 s.

**Interpretation:** Still no served WebP after two checks. The run is either not finished or Local WebP is not enabled. Neither can be checked through the REST API.

**Decision:** Per the task, stopped re-checking. No further automatic checks are scheduled for WebP.

**Pending (user, in wp-admin):**
1. Smush > Bulk Smush: is the run finished, and how many images are done?
2. Smush > Local WebP: is it on?
3. If it's on and the run is done but the images are still not served, that points to the host's rewrite rules for Nginx (see the Converter for Media error earlier). Ask Nestify.
