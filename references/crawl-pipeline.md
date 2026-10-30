# The crawl-export pipeline — reference for the redirect-verification skill

Load triggers: parsing a redirect request file and a crawl export; choosing the URL normalisation; deciding whether a broad locale or rewrite rule covers a specific request; tracing redirect chains; assigning verdicts; exporting verdict rows for a ticket; re-running on a fresh crawl

Reference file for the `redirect-verification` skill. The core `SKILL.md` (the north star, the claim-and-evidence rule, § Shared verification rules, the input ranking and the pre-flight) applies alongside this file.

## Contents

- The pipeline
  - Stage 1 — Parse both inputs with auto-detection
  - Stage 2 — URL normalisation layer (before any matching)
  - Stage 3 — Distinguish broad rules from specific URL rules (and the per-locale hypothesis rule)
  - Stage 4 — Chain analysis
  - Stage 5 — Categorisation framework
  - Stage 6 — Export categorised results for ticket feedback
- Iterative refinement

---

## The pipeline

Run these stages in order. The work is iterative: re-run the whole pipeline as fresh crawl data arrives, refining normalisation and rule matching each pass.

### Stage 1 — Parse both inputs with auto-detection

The request file and the crawl export almost never share a format. Before parsing:

- **Auto-detect the delimiter** (comma, semicolon, tab) per file independently. Crawl-tool exports and hand-authored request sheets frequently differ.
- **Auto-detect the encoding** (UTF-8, UTF-8-BOM, Latin-1, UTF-16). Exports from spreadsheet tools and crawl tools often disagree; a wrong encoding corrupts non-Latin characters and breaks matching downstream.
- Identify the relevant columns explicitly (source URL, destination/target URL, status code, and — in the crawl export — chain position / hop number). Do not assume column order; map by header.

Sanity-check row counts after parsing and report them, so a silently truncated or mis-delimited parse is caught early.

### Stage 2 — URL normalisation layer (before any matching)

Do **not** compare raw URL strings. Normalise both sides through the same function first. This is the single highest-leverage step.

- **URL-decode** — CRITICAL for non-Latin URLs. The request file may store a human-readable (decoded) form while the crawl export stores percent-encoded bytes (or vice versa). The failure shape: on a request file with non-Latin URLs, **a large share of the redirects that are actually correct match only after URL-decoding** — without this step each of them is falsely flagged as not implemented.
- **Trailing-slash normalisation** — decide one canonical form and apply it to both sides.
- **Protocol normalisation** — treat `http`/`https` consistently (or compare explicitly if the protocol is part of the requirement).
- **Lowercasing where appropriate** — host is case-insensitive; path case may be significant, so lowercase the host always and the path only when the site is known to be case-insensitive.

Keep the original (un-normalised) URL alongside the normalised key so the final export can show the human what was actually requested and observed.

### Stage 3 — Distinguish broad rules from specific URL rules

Requirements come in two shapes, and conflating them produces false negatives:

- **Specific URL rules** — one source URL → one target URL.
- **Broad locale / rewrite rules** — a pattern (e.g. a locale prefix change, a directory move, a parameter rewrite) that covers many URLs at once.

Analyse the requested redirects for patterns. A single broad rule can legitimately satisfy many requested specific redirects — so a specific request may be "implemented" by a broad rule even though no line-for-line specific redirect exists. Classify each requirement as broad or specific before matching, and when verifying a specific request, check whether a broad rule covers it.

#### On a multi-locale estate, a broad rule is a hypothesis until tested per locale

Locale is a dimension along which URL behaviour genuinely varies, so **a normalisation or redirect rule verified in one locale is a hypothesis about the others, not a finding**. Two properties that look global and are not:

- **Canonical direction.** The same page can canonicalise in opposite directions depending on language — one locale treating the hyphenated slug as canonical while another treats the unhyphenated one as canonical, or a country version canonicalising to an international version rather than to itself. A blanket find-and-replace or slug rewrite derived from one locale will invert the relationship everywhere it was not tested, and the damage is silent because both spellings resolve.
- **Parameter inertness.** A parameter that is decorative cruft in most locales can change rendered language, currency or content in one of them. Classify parameters empirically — fetch with and without, compare title and a body hash — never by the parameter's name. Note that a body-hash difference is not automatically a real difference: rotating carousels, randomised modules and timestamps produce hash churn on identical pages, so inspect what actually differs before treating it as evidence.
- **The expected target of a rebuilt cross-locale URL.** Shared rule — see `SKILL.md` § Shared verification rules (expected alternate URL). The instance here: when the fix under verification rebuilds hreflang alternates, switcher entries or a cross-domain canonical, each locale's expected target is checked against that rule — per locale, like every other broad rule here — not against what the site linked before the fix.

Two rules follow. **Before applying** any blanket rewrite or redirect rule: gather per-locale evidence, sampling one URL per locale rather than one URL overall, and state which locales were tested. **When verifying** a shipped rule: sample every locale, not one — a rule can be correctly implemented in the locale that was specced and wrong or absent in the rest, and a single-locale check reports that as fully correct.

Also disambiguate the word "canonical" with the requester before acting on it. "Use the canonical URL" can mean the parameter-free URL or the URL in the `rel=canonical` tag, and on a multi-locale estate these are routinely different URLs pointing at different locales. Quantify how many rows the two readings differ on; that number usually settles the question faster than the discussion does.

### Stage 4 — Chain analysis

For each requested redirect, trace the full chain in the crawl export, not just the first hop or the final landing URL:

- Record **every hop**: source → intermediate(s) → final destination, with the **status code at each hop**.
- Compare **chain depth** against the requirement. The spec almost always wants the shortest correct chain (typically a single hop).
- Compare the **status code at each hop** against the requirement. A `301` requirement satisfied by a `302` (or by `301 → 302`) is wrong even if the destination is right.
- Confirm the **final destination** matches (using normalised keys from Stage 2).
- Check whether **old rules were removed**: a stale prior redirect that still fires — overriding the new rule, or inserting an extra hop — is a defect even when the chain eventually reaches the right place.

### Stage 5 — Categorisation framework

Assign every requested redirect exactly one verdict from these actionable buckets:

| Verdict | Meaning |
|---|---|
| **Correct** | Right destination, right status, minimal chain, no stale override. |
| **Wrong target** | Lands somewhere other than the requested destination. |
| **Wrong status code** | Right destination but at least one hop has the wrong status (e.g. 302 where 301 was required). |
| **Unnecessary / extra chain hop** | Reaches the right destination but via one or more avoidable intermediate hops. |
| **Old rule not removed (override)** | A prior rule still fires and shadows or interferes with the new redirect. |
| **Partial chain** | The chain is incomplete or breaks before reaching the requested destination. |
| **Not implemented** | No redirect found for the source URL (and no broad rule covers it). |

If a redirect has multiple defects, record the most actionable/severe verdict as primary and note the others in the evidence so the fix is complete in one pass.

### Stage 6 — Export categorised results for ticket feedback

Produce **one row per requested redirect** so the output can be pasted straight into the ticket. Each row should carry:

- The original requested source and target (un-normalised, human-readable).
- The required status / chain expectation.
- The observed chain (each hop + status) and final destination.
- The **verdict** (from Stage 5).
- **Evidence** — a short, specific reason for the verdict (e.g. "302 on hop 1; spec requires 301" or "matched broad locale rule, OK").

This gives the implementing dev team a precise, line-by-line list of exactly what is wrong and why, instead of a vague "redirects aren't right".

---

## Iterative refinement

This is not a one-shot check. As the dev team ships fixes, request a fresh crawl and re-run the pipeline. Across passes:

- Tighten the normalisation layer when a new false negative surfaces (a new encoding quirk, a new slash/protocol edge case).
- Re-confirm previously **Correct** rows have not regressed.
- Track the shrinking count per verdict bucket so progress is visible.
