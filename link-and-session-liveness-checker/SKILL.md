---
name: link-and-session-liveness-checker
description: Turn a list of URLs and notes about saved login sessions into an ALIVE / DEAD / UNDETERMINED table without false positives or false negatives. Handles single-page apps that answer 200 for any path, 403/429/challenge pages (bot block, so UNDETERMINED, not dead), redirects to a login page (session expired versus link dead), soft-404 pages that say "not found" with status 200, HEAD versus GET differences, and the fact that a cookie's expires date is not the server-side session lifetime. Gives a safe curl or Playwright probe per case with explicit decision rules, and a rate-limit plan so the check does not hammer anyone's site. Use when you need to know which saved links and logins are still good before relying on them.
trigger: The user gives a list of URLs to check, or notes about saved login sessions (cookie file names, expiry timestamps, storage-state files), and asks which are still alive, dead, expired or blocked.
version: 1.0.0
compatible_agents: [claude-code, codex, openclaw]
---

# Link and Session Liveness Checker — a status code is a hint, a known marker is proof

"200 OK" does not mean the page exists. "404" does not always mean it is gone. A cookie that says it
expires next year may already be dead on the server, and one that expired yesterday may still work.
This Skill classifies each item from evidence and names the one probe that settles the doubtful ones.

It does not run requests itself: you run the probes it writes and paste back the results. It is for
links and accounts you own or are explicitly allowed to test. It does not help get around a bot
block, CAPTCHA or login wall; if the evidence shows a deliberate block, the item is UNDETERMINED
and the Skill stops there.

## Rules that apply throughout

- **Three verdicts only.** ALIVE (positive proof), DEAD (positive proof of absence or expiry),
  UNDETERMINED (everything else, including every block). Never round UNDETERMINED to DEAD.
- **Proof is a marker, not a status code.** ALIVE needs a string or element that only the real page
  (or only a logged-in user) shows. DEAD needs a real 404/410, a DNS failure, or a login-wall bounce
  on an item that previously worked.
- **A block is not a death.** 403, 429, 503 with a challenge, "verify you are human", or a
  `cf-mitigated` / `captcha` style header means the site refused the probe. Report UNDETERMINED
  and say "blocked, not dead".
- **Secrets stay on the user's machine.** Never ask for, accept or print a cookie value, token or
  password. Tell the user to redact values and paste only cookie **names, flags (Secure, HttpOnly,
  SameSite) and dates**. If a value was pasted, tell them to treat that session as exposed and log out
  or revoke it. Your scripts read cookie files from a path and never echo their contents.
- **Own or permitted targets only.** Ask once if ownership is unclear. No probing third-party
  accounts, no credential guessing.
- **Changing the User-Agent is a diagnostic aid only.** It can show whether a site treats a bare
  `curl` differently from a browser, but site rules vary, it is the user's responsibility to check
  them, and it may stop working at any time. Never use it, or any header trick, to get past a block
  the site put there on purpose; if a challenge appears, stop.
- **Do not hammer.** One request per URL per run; at least 2 seconds between requests to the same
  host and about 1 second across hosts; stop a host after its first 429 and honour `Retry-After`.
  No loops, no retries on blocks. Do not invent limits the site did not state.
- **Do not invent** markers, endpoints or header names. Use what the user pasted; otherwise write a
  placeholder like `<logged-in-only-url>` and say what to look up.

## Phase 1 — Inventory the evidence

For each URL or session, list (write "unknown" when missing):

1. **Item** and what it should be (article page, product page, dashboard, API endpoint).
2. **Observed**: status, redirect chain, `Content-Type`, key headers, first 200 characters of the body.
3. **Known marker**: text or element that must appear when it is healthy (a title, a price label).
   If none is given, ask for one; without it a 200 can only be UNDETERMINED for SPAs.
4. **For sessions**: cookie names, flags, `expires` dates, when last verified, which logged-in-only
   page or endpoint exists, whether the site rotates tokens on use.

If the user pasted only URLs with no results, go to Phase 3 and hand out probes first.

## Phase 2 — Classify with the case table

| Case | Signature | Verdict |
|---|---|---|
| Healthy | 2xx and the known marker is present | ALIVE |
| Real not-found | 404 or 410 with a not-found body, or DNS/connection refused on repeat | DEAD |
| Soft 404 | 200 but body says "not found", "no longer available", empty result list, or title equals a generic site title | DEAD if the marker is missing and the text is explicit, else UNDETERMINED |
| SPA catch-all | 200 and the same small HTML shell (`<div id="root">`) for any path, including a path you invented | UNDETERMINED until rendered content is checked (Case C) |
| Bot block | 403, 429, 503, challenge title, "Just a moment", captcha markup, `Retry-After` | UNDETERMINED (blocked) |
| Redirect to login on a public link | 30x to `/login` or `/signin` for a link meant to be public | UNDETERMINED: gated or moved, check manually |
| Redirect to login on a saved session | Authenticated request lands on login page | DEAD session (expired or revoked) |
| Redirect elsewhere | 301/302/308 to a new path with the marker present | ALIVE, note the new URL |
| HEAD disagrees | HEAD gives 405/403/404 but GET gives 200 | Trust GET with marker; HEAD is not evidence |
| Timeout or 5xx once | Single 500/502/504 or timeout | UNDETERMINED; re-check later, not now |

## Phase 3 — Probes, one per case

All snippets use placeholders. Run one request per item and paste back the output.

**Case A — plain link, GET not HEAD (status, final URL, size, marker):**
```bash
curl -sS -L --max-time 20 --max-redirs 5 -o /tmp/body.html \
  -w 'status=%{http_code} final=%{url_effective} bytes=%{size_download} type=%{content_type}\n' \
  '<url>'
grep -c -i '<known-marker>' /tmp/body.html
```
Rule: status 2xx and marker count at least 1 is ALIVE. 2xx with marker 0 and small bytes is a soft
404 or SPA shell, so go to Case C. Status 403/429 or a challenge title is UNDETERMINED (blocked).

**Case B — header-only view, to see blocks and redirects without a body:**
```bash
curl -sS -I --max-time 20 '<url>'; echo '--- GET headers ---'; curl -sS -D - -o /dev/null --max-time 20 '<url>'
```
Rule: if HEAD and GET differ, believe GET. Note `Server`, `Retry-After`, and any challenge-style header.

**Case C — SPA that answers 200 for everything (control path test):**
```bash
curl -sS -o /tmp/real.html -w 'real=%{http_code} %{size_download}\n' '<url>'
curl -sS -o /tmp/fake.html -w 'fake=%{http_code} %{size_download}\n' '<same-origin>/this-path-should-not-exist-7f3a'
cmp -s /tmp/real.html /tmp/fake.html && echo IDENTICAL || echo DIFFERENT
```
Rule: IDENTICAL means the server cannot tell you anything; render it (Case D). DIFFERENT plus the
marker is ALIVE. Make the fake request once per host, not once per URL.

**Case D — render the page and look for the marker (Playwright, no stealth, no evasion):**
```js
const { chromium } = require('playwright');
(async () => {
  const b = await chromium.launch(); const p = await b.newPage();
  const r = await p.goto(process.argv[2], { waitUntil: 'networkidle', timeout: 30000 });
  const text = await p.locator('body').innerText();
  console.log(JSON.stringify({ status: r && r.status(), url: p.url(), title: await p.title(),
    marker: text.includes(process.argv[3]), head: text.slice(0, 200) }));
  await b.close();
})();
```
Usage: `node probe.js '<url>' '<known-marker>'`. Rule: marker true is ALIVE; marker false with explicit
"not found" text is DEAD; a challenge title or captcha text is UNDETERMINED (blocked), do not retry.

**Case E — saved session: one authenticated request to a logged-in-only endpoint:**
```bash
curl -sS -L --max-time 20 --max-redirs 3 -b '<cookie-file-path>' -o /tmp/me.html \
  -w 'status=%{http_code} final=%{url_effective}\n' '<logged-in-only-url>'
grep -c -i '<logged-in-only-marker>' /tmp/me.html
```
Rule: marker present and final URL not a login page is ALIVE. Final URL is the login page, or 401,
is DEAD (expired or revoked). 403 or a challenge is UNDETERMINED. Choose a marker that exists only
when logged in (logout link, account name area, balance), never a word like "Log in" that footers
show to everyone. Some sites rotate tokens on use, so check the user's notes before testing and
save the cookie file afterwards if the site returned new cookies (`-c <cookie-file-path>`).

**Case F — cookie dates are not session lifetime:**
Read only names, flags and dates. Then apply: a far-future `expires` is often a tracking cookie, not
the login; a session-only cookie (no expires) can still live for days; a cookie past its date is
usually dead but only Case E proves it; some tokens carry their own expiry inside, which the user can
check locally without sharing the value. A live session can outlive or die before its cookie date.
Report the cookie date as "claimed", the Case E result as "observed", and trust observed.

## Phase 4 — Run order and pacing

1. Group by host. Run each host sequentially with at least 2 seconds between requests.
2. Cheapest probe first (Case A); escalate to C, D or E only for items that stayed unclear.
3. After any 429 or challenge, stop that host for this run and mark remaining items on it
   UNDETERMINED (not checked). Suggest waiting for the `Retry-After` time or running later.
4. Re-run schedule suggestion: sessions daily at most, links weekly; more often only if the site
   documents that it is fine.

## Phase 5 — Verdict table

Output one row per item, then a short action list.

| Item | Verdict | Evidence (status, marker, final URL) | Confidence | Next step |
|---|---|---|---|---|

Confidence is confirmed (marker or authenticated probe seen), likely, or unconfirmed. Next steps:
ALIVE: none, or update stored URL after a redirect. DEAD link: replace or remove. DEAD session:
log in again yourself in a normal browser and re-export your own cookies. UNDETERMINED (blocked):
open it manually in your browser; do not retry in a loop.

## Output format

Headings in order: `## Evidence`, `## Verdicts` (the table), `## Probes to run` (only for items
still unclear, with the snippets above filled in), `## Notes` (pacing, rotation, anything blocked).
Keep it under about 500 words plus snippets. If an item has no marker and no rendered content, say
exactly what is missing instead of guessing.
