# Variants and residue — reference for the redirect-verification skill

Load triggers: verifying an internal-link or template fix without a crawl; a pre-switch check of a rule-based redirect mapping; reviewing a fuzzy-matched redirect mapping before it is handed over; testing a country- or IP-based redirect; designing a spot-check sample of links; classifying residue before it is reported; splitting a missing-redirect count by target state; a section that has moved to a new host

Reference file for the `redirect-verification` skill. The core `SKILL.md` (the north star, the claim-and-evidence rule, § Shared verification rules, the input ranking and the pre-flight) applies alongside this file.

## Contents

- The no-crawl variant
- The pre-switch variant
- Reviewing a matched mapping before hand-over
- Geo-dependent redirects — establish each channel's vantage point
- Spot-check sampling — classify the emitter, then vary the axis it varies on
- Classify the residue before reporting it (residual links by target status; defect shape and scope; residue by pattern class; missing redirects by target state; the host-migration fourth check)

---

### The no-crawl variant

An internal-link or template fix rarely needs a crawl. Enumerate the claimed fixes from the ticket thread, fetch the affected server-rendered pages per locale with an explicit desktop UA, confirm each fetch's provenance (final URL, status, canonical), and grep for the old and the new `href`. A fix of a few parts across a dozen locales is on the order of ten requests — arithmetic that holds for component-emitted links only; a link authored into content needs the full-population sweep described under *Spot-check sampling* below. Include the newest locales specifically — they are the ones most likely to have been created after the fix and to have inherited the old template. Then classify every residue before reporting it (see *Classify the residue* below): a residual old-style link is not automatically a defect.

### The pre-switch variant

Before a rule-based mapping goes live — a domain retirement's "301 to an indexable equivalent, 410 otherwise" — verify what it *would* answer, so a defective rule is found before it has redirected anything.

- **The preview ships in the same change as the redirects.** A request header makes the old host report its would-be answer (status and target, e.g. in a response header) without changing behaviour otherwise. It costs no review cycle of its own, and the crawl runs while the switch waits on external blockers. Where production already emits the same mapping after the page has rendered (cross-domain canonicals), crawl that as the dry run as well.
- **Crawl the old host with the preview header, then list-crawl every would-be target on the new host.** The gate is 100% of targets answering 200, indexable and self-canonical; any other target is a rule defect, fixed before the switch.
- **Scope by template, because the rules work per template.** All URLs with clicks, plus a per-template sample weighted to templates whose controllers return 404 themselves — the cases a decision made before rendering cannot see.

### Reviewing a matched mapping before hand-over

A mapping built by fuzzy matching (path similarity, keyword overlap) is a claim like any other, and three checks run on it before it is handed to the implementing team.

- **Validate entity identity after matching.** Fuzzy matching optimises for surface similarity, so it pairs different entities of the same type — one product model with another, one company with a similarly named one — and pulls pages across sections on a shared keyword. Extract the primary entity (product, company, topic, standard or model number) from source and target, flag every mismatch for manual review, and use a lexicon of known entity names plus section-aware matching so a jobs page is never matched to a content page on a shared term.
- **Treat script variants of one language as a separate matching problem.** Simplified and traditional Chinese write the same term with different characters, so standard matching between the two locales finds no match rather than a wrong one, and the failure is silent. Convert between the scripts before matching, match on the language-agnostic part of the URL (standard numbers, product codes), or route the pair to manual review.
- **The least-specific targets are a review queue, not a result.** Every source mapped to a locale home page or a section fallback is reviewed by hand before hand-over; a few minutes per entry often finds a more specific target, and none stays a final state without a stated reason.

### Geo-dependent redirects — establish each channel's vantage point

Before testing a country- or IP-based switch, run a CDN trace endpoint (for example `https://www.cloudflare.com/cdn-cgi/trace`, which reports `loc=` and `colo=`) from every available channel — the sandbox shell, the user's browser — to learn which country each one is seen from, and test from the channel that reports the country in question. A cloud sandbox has reported a US location, which made it a US-visitor vantage point with no VPN — Last verified 2026-10-08 · Re-test: `curl -s https://www.cloudflare.com/cdn-cgi/trace`. The egress location is a measurement, not a property, so re-run the trace every time. Only cookieless behaviour is testable this way; a cookie-based switch is not.

---

### Spot-check sampling — classify the emitter, then vary the axis it varies on

Shared rule — see § Shared verification rules (emitter classification). The instance here: classify **what emits the URL** — a shared component (theme, template, header, footer, nav, breadcrumb) or a link a person typed into an individual document's body — before any sample is designed.

For a **component-emitted** link, design the sample around the emitter, not around page types: a link emitted by a global header or footer returns the same answer on every page type of a locale, so a locale × page-type grid has one real axis (locale) and one decorative one. One page per locale is sufficient; the emitters to enumerate are header, footer, mobile nav, breadcrumb and any in-content module.

For a **content-emitted** link, enumerate the full population from the sitemap and fetch every page, scoping the match to the body container (the article or main element) so component instances do not contaminate the count — that scoping is also what separates the two populations in one pass. Cost the full sweep rather than dismissing it: roughly 1,500 pages fetch in about fifty seconds as a detached async loop from a tab already on the target origin (same-origin `fetch`, concurrency around 12, progress counter and hits kept in a `window` variable so a timed-out poll is re-read rather than restarted). Report per-locale counts. The failure shape: a small sample of a migrated help centre reports a rate several times higher than the full sweep finds; the sweep also turns up articles carrying the link in translations but not in the source-language version, and one regional locale with none of its articles affected while its sibling locale, over the same article set, has a substantial share affected.

Never spot-check by counting occurrences of the old URL or slug in a page's source. Resolve each match's host and enclosing component before counting it as a defect — cross-domain hreflang annotations and language-switcher links legitimately contain other locales' URLs, and a raw occurrence count reports those as false regressions. Establish the enclosing component from the parsed DOM, not from where the match sits in the serialised HTML: proximity to a footer marker in the source is a correlate of containment, not containment, and attributes a link to the wrong emitter.

This skill pairs naturally with a crawl-export-to-developer-instructions style workflow: that workflow produces the redirect request from crawl data; this one verifies the request was honoured. They form the two ends of the same loop.

---

## Classify the residue before reporting it

A verification's value lies in what it says about the leftovers, and an unclassified residue mixes real defects with instances that were never in scope and with defects that belong to a different ticket.

**A residual old-style link is only a defect if its target still misbehaves.** Resolve the residual URL before counting it against the fix:

| Target status | Class |
|---|---|
| **200** | Never in scope — a live URL that merely resembles the old pattern. Not a defect. |
| **301 / 302** | Unfixed instance — the link still points at a redirecting URL, which is what the fix was for. |
| **404 / 410** | New defect class — a broken link, and a different ticket from the one being verified. |

This step is what keeps false positives out of the report. Slugs that look untranslated or legacy in a couple of locales are routinely live 200 URLs with no connection to the issue at all.

**A confirmed defect has a shape and a scope.** Shared rule — see § Shared verification rules (Rule 4, defect shape and scope). The instance here: every residue the table above classes as a defect — an unfixed instance or a new defect class — is scoped against the rows this verification found correct before it is named in the report.

**Classify the residue by pattern class before scoping a fix.** Shared rule — see § Shared verification rules (emitter classification). The instance here: the grouping axes are the failing URL's structure, template, locale and section, and a truncated crawl is re-run larger before the residue is sized. The failure shape: the first residue found — a page whose alternates all resolve to the home page — is analysed and fixed on a small crawl sample; the same crawl run to its full size surfaces a second, larger class (section-specific pages rebuilt without their section) with the same visible failure and a different mechanism, which the instance-level fix does not touch.

**A missing redirect splits by whether its intended target still exists.** For every source URL that does not redirect, additionally fetch the target the spec named. "Redirect missing, target live" and "redirect missing, target gone" need different fixes and usually different owners — one is a routing job, the other is a content question — and a single "missing redirects" count conflates them. The failure shape: most of the non-redirecting sources turn out to have targets that are still live, so the single count hides that the bulk of the fix is a routing job.

**A redirect map does not carry over to a newly launched domain or locale.** When an old URL is still linked on a domain launched after the map was built, verify the redirect exists on that domain, not on the established ones. The failure shape: an old article URL 301s correctly on the established domains and 404s on the newest one, where a related-article link still points at it.

**A host migration needs a fourth check that redirects, canonicals and sitemaps cannot make.** When a section moves to a new host, all the standard checks can pass — path-preserving 301 from the old host, self-canonicals on the new host in every locale, a regenerated sitemap containing only new-host URLs, both hosts' robots.txt pointing at the new sitemap — while the migrated pages themselves still link to the old host on every page. So, after those pass:

- Grep the migrated pages' **own raw HTML** for anchors whose `href` host is the old host.
- Sample both hub/index and detail templates, in **every** locale. Theme-level hardcoded absolute URLs are per-template, so a single-page or single-locale spot check finds them or misses them by luck.
- Classify each hit. An `<a href>` is a real internal link and a finding; a config or JSON value (a vendor theme's brand-URL variable, for example) is not a link and is a false positive to discount.

Such links redirect, so nothing appears broken — but the site is handing search engines the old host on every page of the migrated section, which is precisely what any pending "wait for the old URLs to drop out of the index" step depends on not happening. Redirects, canonicals and sitemaps describe what the new host *says about itself*; they say nothing about what it still *links to*.
