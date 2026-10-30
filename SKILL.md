---
name: redirect-verification
version: 1.1.0
core_max_lines: 164
description: Post-completion verification of a reported fix against its spec — establishing what is actually live and how it differs from what was requested. Redirects are the canonical case, cross-referenced against crawl data (e.g. a Screaming Frog "All Redirects" export) on destination, status code, chain depth and old-rule removal; the same discipline covers internal-link fix tickets, host migrations and bulk redirect jobs, which need targeted live requests rather than a crawl. Load this skill when the user mentions redirect verification, checking whether an implementation matches the spec, Screaming Frog redirects, redirect chain analysis, 301 vs 302 verification, "did the dev team implement this correctly", verifying a migration or job log against live state, a pre-switch check of a redirect mapping, or ticket QA after a fix is reported complete.
license: CC-BY-4.0
metadata:
  author: Eoghan Henn and contributors
  source: https://github.com/rebelytics/redirect-verification
---

# Redirect Verification

This skill codifies post-completion verification: a change has been reported as done, and the job is to establish what is actually live and how it differs from the spec. Redirect tickets checked against a crawl export are the canonical case, and the pipeline in `references/crawl-pipeline.md` is written for them — the normalisation steps, chain analysis, and categorisation framework that make verification reliable rather than guesswork. The same discipline covers the neighbouring cases (internal-link fix tickets, host migrations, bulk removal-and-redirect jobs), which are usually cheaper because they need targeted live requests rather than a crawl.

---

## North star: redirect verification is multi-dimensional

A redirect is not verified just because the browser ends up on the right page. The same final destination can hide a wrong status code, an extra chain hop, or an old rule that was never removed. Every requested redirect must be checked across four dimensions, not one:

1. **Final destination** — does it land on the URL the spec requires?
2. **Status code** — is each hop the status the spec calls for (e.g. a single 301, not a 302 or a 307)?
3. **Chain depth** — does it reach the destination directly, or via unnecessary intermediate hops?
4. **Old-rule removal** — was the previous redirect/rule for this URL actually removed, or is a stale rule overriding or shadowing the new one?

A redirect that reaches the correct final page via a `301 → 302` chain is **still wrong** if the spec calls for a single `301`. Simple final-destination checking misses this entire class of issues. Treat "lands on the right page" as necessary but not sufficient.

---

## When to use this skill

Trigger when:

- The client or implementing dev team reports a redirect ticket as done and the user wants to confirm it.
- The user has the original redirect request (one or more files) plus a fresh crawl export and wants a per-redirect verdict.
- A redirect implementation is being checked iteratively across successive crawls as fixes land.
- Any other reported fix needs checking against its spec after the fact — an internal-link ticket ("we replaced the old URLs everywhere"), a host migration, a bulk removal-and-redirect job — where the question is what is live now versus what was specified.
- A rule-based redirect mapping is about to go live and its would-be answers can be checked before the switch — see *The pre-switch variant* in `references/variants-and-residue.md`.

Every case needs the same two inputs, in the same relationship: **the claim** and **the evidence**.

- **The claim** — the spec the fix was meant to satisfy: the redirect request file (source URL → target URL, expected status, and any locale/rewrite rules), the ticket's enumerated list of fixes, or the recommendation a migration job was built from. Enumerate the claimed fixes explicitly before checking anything; a claim you did not write down is a claim you will not verify.
- **The evidence** — observed behaviour. A crawl export (a Screaming Frog "All Redirects" / "Redirect Chains" export is the canonical form, but any crawl that records each hop's source, destination, and status code works), or targeted live requests. Neither a ticket comment nor a job's own output log is evidence — see *Shared verification rules* (Rule 1, the deployment gate) and *Rank the inputs* below.

Use a scripting/Python capability for the matching — this is a data-join problem, and doing it by eye does not scale past a handful of rows and silently misses chain and status issues.

The crawl-export case runs as a six-stage pipeline — parse both inputs, normalise URLs, separate broad rules from specific ones, trace every chain, assign one verdict per request, export one row per request. **Load `references/crawl-pipeline.md` before Stage 1 and again on every re-run with a fresh crawl.** The no-crawl and pre-switch variants, geo-dependent redirects, spot-check sampling and residue classification are in `references/variants-and-residue.md`; its load triggers are listed under *Reference files* below.

## Reference files — load on demand

Each pointer names its trigger. These loads are mandatory steps: when an episode fires, load the file before doing the work — never improvise the episode from this core file. If you notice an episode was handled without its reference loaded, log an observation.

- `references/crawl-pipeline.md` — the six-stage crawl-export pipeline (parsing with auto-detection, the URL normalisation layer, broad versus specific rules and the per-locale hypothesis rule, chain analysis, the verdict table, the ticket-feedback export) and iterative refinement across crawls. **Load before parsing a redirect request file and a crawl export, before matching a specific request against a broad rule, before assigning verdicts, and on every re-run with a fresh crawl.**
- `references/variants-and-residue.md` — the no-crawl variant (internal-link and template fixes), the pre-switch variant (a rule-based mapping checked before it goes live), the review of a fuzzy-matched mapping before hand-over, geo-dependent redirects, spot-check sampling by emitter class, and residue classification including the host-migration fourth check. **Load before verifying a fix without a crawl, before a pre-switch check of a redirect mapping, before handing over a fuzzy-matched redirect mapping, before testing a country- or IP-based redirect, before designing any sample of links, before reporting any residue or missing-redirect count, and when a section has moved to a new host.**

---

## Shared verification rules

Four rules sit underneath every verification of a claim against live state, whatever the subject — redirects, structured data, translations, breadcrumbs, hreflang. The sibling audit skills point here instead of carrying their own copies, and each keeps only its own instance: what the population is, what counts as residue, which verdicts the gate guards. Change a rule here, never in a sibling.

### Rule 1 — The deployment gate: whether it shipped, then when

The gate is two ordered questions, and the first one is unconditional. It applies whatever the evidence channel is: crawl export, cached report, a previously fetched sample, or live checks made a minute ago. Cache-busted live evidence does not exempt it.

#### Question 1 — was the change deployed at all?

Live state is the *observation*; deployment is the *claim being tested*. No amount of evidence about current state answers whether the change was ever requested, built and shipped — so a live-check verification needs this question exactly as much as a crawl-based one does. A live fetch establishes current state, not that a deployment occurred: the redirect, block, annotation or corrected text may have been there all along, or been produced by something unrelated. The question is easy to skip precisely because live checks are immune to staleness: the freshness worry that motivates Question 2 is genuinely satisfied, and the whole gate then reads as spent. It is not; only its second half is.

Resolve it from the same evidence classes as the date — a deploy log, a release, a commit, the requester's first-hand account, or the client confirming the work was commissioned at all. If it is still unresolved when the report goes out:

- Say so in the report's **opening line**.
- Report live state plainly and offer **no causal reading of it**. "Shipped for section X only", "partially rolled back", "the job ran and deployed for one language version" each presuppose an affirmative answer, and each is wrong in a different way if nothing was ever deployed — the same working redirects would then be pre-existing alias behaviour rather than a partial rollout.
- **Hold the remediation lists**, and any state change that records a fix as verified, until it closes. Action lists built on an unresolved deployment question are premature, and the reader cannot tell.

#### Question 2 — when did it ship, and how does the evidence date against it?

This matters only once Question 1 is yes, and only for dated evidence. A verification then compares two dated things — the fix and the evidence — and each date needs its own provenance. A ticket comment saying "this is done" dates the report, not the change: the dates a thread actually offers (when someone was asked, when someone replied, when a status was relayed, when the ticket was closed) are systematically *later* than when the code shipped, because reporting lags doing and the summarising comment is the most prominent, most quotable line. Sourcing the deployment date from that comment makes clean evidence look contaminated, or stale evidence look valid; the resulting "wait for another crawl" recommendation is an invisible cost nobody ever disputes. Never discard or accept evidence against a fix on the strength of a completion comment's date.

Run the dating in two steps, naming the source of each date in the finding:

1. **Pin the deployment date from evidence, not prose.** Prefer a deploy log, a commit or release timestamp, a per-URL crawl timestamp showing the changed state, or a monitoring series. Where only ticket comments exist, take the EARLIEST comment demonstrating the change on production — typically a developer pasting a live URL — not the latest comment declaring completion. If the date cannot be pinned, say so and give the range rather than promoting a comment date to a fact. Where the person asking has first-hand knowledge of the work, their account of when it shipped outranks any inference from the thread.
2. **Compare the evidence's start AND end against it.** Three outcomes: evidence gathered entirely after the fix — valid, findings are genuine residue; entirely before — no evidence, wait; genuinely straddling — findings are undated, so restrict verdicts to URLs whose fetch timestamp postdates the deployment, or fall back to live checks and defer the sweep. A live fetch made after the pinned date needs no further dating, which is what makes it the fallback.

#### The two questions guard different verdicts

A positive verdict that attributes the live state to the reported change ("fixed", "verified", "implemented") needs Question 1. A negative verdict ("not fixed", "not implemented") additionally needs Question 2: it is only meaningful if the evidence postdates the deployment. When the date is not yet established, treat a negative verdict as provisional and re-check once it is, rather than counting a failed attempt against a fix that had not yet shipped.

Corollary: a live re-check of a deterministic rule is stronger and cheaper than any crawl — a rule that fires correctly on constructed inputs proves the general case. The crawl's job is to size the residue, not to establish that the rule exists.

### Rule 2 — Classify the emitter before sampling, and the residue before scoping a fix

Before designing any sample or sizing any population, classify **what emits** the thing being checked — the link, string, block or property — because the emitter class decides whether a sample is valid at all:

- **Component-emitted** — generated from a rule by a shared template, component or serialiser (theme, header, footer, nav, breadcrumb, template boilerplate, a structured-data serialiser). Every instance comes from one rule, so it varies only along the axes the emitter varies on — typically locale and template — and is identical on every page of a locale and template by construction. Sample: one page per locale × emitting template answers for the rest.
- **Content-emitted (authored)** — typed by a person into an individual document, or into an individual translation of that document: body copy, an in-article link, a description, a step list, a pasted identifier. There is no rule to infer: the affected set varies per document and per translation, so a sample measures only itself. Do not sample; enumerate the whole population — the full page-type class from the sitemap, in every locale, not one page's set of language versions — and report per-locale counts. This holds for verification too: a fix to authored values repairs the instances it touched, and the unfixed instances are not predictable from the pages that were checked.

A sample is only valid where the thing sampled is generated by a rule. Three consequences:

- **Translations of one document are independent artefacts.** A defect in one language version can be absent from the source and from a sibling translation, and the reverse. Never infer a translation's state from its source or from another translation. A per-locale sweep of one page's language versions, applied to an authored value, is the misclassification that lets "fixed for all languages" pass on the one page that was checked.
- **One page can carry both classes of the same thing.** A footer link and an in-body link to the same retired host are two populations, and the sampling answer for one is wrong for the other.
- **A component sample varies only the axes the emitter varies on.** An axis it does not vary on (page type, for a global header) is decorative; spend the saved effort enumerating every component that emits the thing, because a second, untouched emitter is where the residual hides. A fix report that names a component ("we fixed the footer links") is scoping information: check the unnamed emitters first.

**The residue is classified by pattern class over the largest sample available — before any fix is scoped.** A fix designed on the first instance found is a fix for an instance: the first failing row is one member of one class, and it says nothing about whether a second class with the same visible shape and a different mechanism sits in the rows not yet looked at. Group the whole residue by the template that emits it and by pattern (the failing URL's structure, locale, section) over every row in hand, and enlarge the sample when the crawl or sweep was truncated — the residue in a truncated sample is not the residue. The residue's aggregate by pattern, not its first member, defines the fix's scope.

### Rule 3 — The expected alternate URL is the language's own variant

When a fix or an audit concerns URLs that point across locales — hreflang alternates, language or country switcher entries, a cross-domain canonical — and a page has per-language URL variants (a translated slug per language, each live with a self-canonical), the correct alternate on each domain is **that language's own variant**. Write that expectation down before any check or fix is designed. Two things masquerade as the spec, and both must be rejected:

- **The path the site links today — under-reaches.** A switcher template that appends one shared path to every host is evidence about the link template, not about the URL. Taking the currently linked path as the expected alternate names the same untranslated path on every domain.
- **The path a consistent rule would produce — over-reaches.** Applying "the language's variant" mechanically fails where one locale's established, indexed and linked URL is itself a valid variant of the language's slug: swapping it for the "consistent" spelling changes a live URL for no benefit and adds a redirect chain.

The rule that survives both: **the language's own variant is the spec, except that an established, indexed URL is kept wherever it is a valid variant** — decided by data (index status, inlinks, self-canonical), not by naming the language. Establish the per-language variants from the route table or the live self-canonicals, never from the switcher, so observed alternates are compared against the spec rather than against the link template.

### Rule 4 — A confirmed defect has a shape and a scope: test the shape against the rows that passed

A confirmed defect is one member of a class, and the class has a **shape** — the structural condition the failing rows share — and a **scope** — which members of the population it actually covers. The failing rows establish only the shape; only the rows that passed establish the scope. Once a residue is confirmed a defect, establish its scope before naming it.

The shape — "uses the country code where the vendor expects a language code" — describes the failing instances accurately and, in doing so, asserts a cause: that whoever built them did not know the two vocabularies differ. Test that cause before it becomes the headline. Enumerate every member of the population that carries the same structural condition (the same differing-code condition, the same template, the same parameter), failing and passing, in one table, and look among the passing ones for the condition done right. A passing row that matches the shape means the test — the shape read as a cause — is wrong, not the row:

- **A matching passing row exists** — the finding is a per-instance misconfiguration scoped to the failing rows: a different fix, usually with a different owner, from a systemic one.
- **None exists** — the finding is a systemic rule, and the shape may name it.

The rows that passed are not the background to the finding; they are the evidence that decides what the finding is. The per-row table that decides this is routinely already in hand and never read as a distribution. Name the finding by its scope (which domains, templates or rows) rather than its shape unless the shape has survived this counterexample check, and where a working counterexample exists, report it alongside the failures — it is the clause that stops the reader widening the fix.

Failure shape: several country domains link a vendor-hosted support path keyed on the country code where the vendor expects the language code, so each falls back to the vendor's default language. Read from the failing domains alone, the finding becomes "the estate confuses country and language codes", and the fix becomes an estate-wide code-mapping audit. A domain in the same table with a differing pair keyed correctly falsifies that cause: the defect is the failing domains' configuration, and the fix is a per-domain config change.

## Rank the inputs — a log is testimony, not observation

A job, migration or deploy log is a claim about a *run*. It ranks with a ticket comment, not with a crawl or a live check. This is counter-intuitive when the log is per-URL: hundreds of lines each naming a source URL and the target it was redirected to read like the strongest possible evidence, a machine reporting success case by case. They are not evidence of live state — they record what the job computed, in whichever environment it ran in. The failure shape: a removal-and-redirect job's log reports most rows as handled and a few as errors; live-checking every row finds that only a minority of the "handled" redirects resolve, while some of the rows the same log reported as *unhandled* do redirect.

Two rules follow:

- **Never treat a per-row success line as per-row deployment evidence.** Every row is unverified until it has been checked live.
- **When checking every row is cheap, check every row rather than sampling.** The failure boundary is routinely an axis nobody thought to sample along: in that failure shape the working redirects are confined to a single path segment — a fact invisible in the log — and a per-locale sample still misses it, because the axis is section, not locale.

---

## Pre-flight verification (run before declaring done)

Before reporting results as final, re-read the categorisation rules (`references/crawl-pipeline.md`, Stage 5) and confirm:

1. **Every** requested redirect has exactly one verdict — none silently dropped during the join.
2. Verdicts were assigned using **all four dimensions** — final destination AND status code at each hop AND chain depth AND old-rule removal — not final destination alone.
3. The URL-decode (and other normalisation) step was applied to **both** sides before matching, so no correct redirect is falsely flagged as "not implemented".
4. Broad locale/rewrite rules were considered when marking specific requests as "not implemented".
5. On a multi-locale estate, every broad rule was evidenced **per locale** — canonical direction and parameter behaviour confirmed in each, not extrapolated from one — and the verification sample covers every locale the rule touches. The report names the locales tested. Where the fix rebuilds cross-locale URLs, the expected alternate per domain is the language's own variant, established from the route table or live self-canonicals rather than from what the site currently links, and no established, indexed URL was proposed for replacement merely for consistency.
6. Every verdict row carries concrete evidence a developer could act on.
7. **Whether** the change was deployed at all is resolved — or the report says in its opening line that it is not, offers no causal reading of the observed pattern, and withholds the remediation lists. This applies to live-check verifications exactly as it does to crawl-based ones.
8. Any comparison of dated evidence against a reported fix sourced the deployment date from evidence (deploy log, release timestamp, earliest comment demonstrating the change live) — never from a completion comment — and the report names the source of each date.
9. No job, migration or deploy log was treated as evidence of live state. Where one was the input, every row was checked live rather than sampled.
10. Every residue was classified before being reported: residual links by their target's status (200 = never in scope, 3xx = unfixed instance, 4xx = new defect class), and missing redirects by whether the intended target is still live or gone. Every confirmed defect was scoped against the passing rows before it was named — a working counterexample with the same structural condition is reported alongside the failures, and the heading names the scope, not an unchecked shape. The residue was grouped by pattern class over the largest sample available before any fix was scoped; no fix was designed on the first instance found.
11. On a host migration, the migrated pages' own HTML was grepped for anchors pointing at the old host — hub and detail templates, in every locale — with `<a href>` hits separated from config/JSON values.
12. The emitter was classified before any sample was designed. Component-emitted links were sampled along the axes the emitters vary on (locale × emitting component, not page type); content-emitted (authored) links were enumerated in full from the sitemap with the match scoped to the body container and counts reported per locale. No residual was asserted or cleared by raw occurrence counts of the old URL.
13. In the pre-switch variant, every would-be target was list-crawled on the new host and the gate held at 100% (200, indexable, self-canonical), over all URLs with clicks plus a per-template sample weighted to templates whose controllers 404.

If any check fails, fix it and re-run before sharing the output.

## Feedback on this skill

If the user has methodology feedback, offer to draft a report for github.com/rebelytics/redirect-verification, running the feedback pre-flight first; if the problem is the agent not following the skill's rules, acknowledge and correct it instead.

Redirect Verification: Eoghan Henn and contributors | CC BY 4.0
