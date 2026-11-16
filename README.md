# redirect-verification

Open-source skill for **post-completion verification of a reported fix against its spec** — establishing what is actually live and how it differs from what was requested. Redirects are the canonical case, checked against crawl data on destination, status code, chain depth and old-rule removal; the same discipline covers internal-link fix tickets, host migrations and bulk redirect jobs. Load it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.).

Part of the [Open Skill Library](https://github.com/rebelytics/one-skill-to-rule-them-all/blob/main/OPEN-SKILL-LIBRARY.md), the open-source agent skills maintained with Task Observer.

## When it triggers

Redirect verification, checking whether an implementation matches the spec, Screaming Frog redirects, redirect chain analysis, 301 vs 302 verification, "did the dev team implement this correctly", verifying a migration or job log against live state, a pre-switch check of a redirect mapping, or ticket QA after a fix is reported complete.

## What this repo contains

- `SKILL.md` — the north star, the shared verification rules (deployment gate, emitter and residue classification, expected alternate URL, shape and scope of a defect), the input ranking, the pre-flight, and the pointer list for the reference files.
- `references/crawl-pipeline.md` — the six-stage crawl-export pipeline, from parsing and URL normalisation to verdicts and the ticket-feedback export.
- `references/variants-and-residue.md` — the no-crawl and pre-switch variants, fuzzy-matched mappings, geo-dependent redirects, spot-check sampling and residue classification.
- `references/ceilings.txt` — line ceilings for the reference files, read by the skill validator.
- `LICENSE` — CC BY 4.0.

## Install

Install the whole skill directory — `SKILL.md` and `references/` must travel together — into your MCP-capable agent's skills directory, following the client-specific path, and restart the client.

## Contributing

Open an issue or a PR at [github.com/rebelytics/redirect-verification](https://github.com/rebelytics/redirect-verification).

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)

## License

CC BY 4.0 — the licence text is in [LICENSE](./LICENSE): the full legal code, unmodified, as Creative Commons publishes it; the [human-readable summary](https://creativecommons.org/licenses/by/4.0/) says what it allows. Use it, fork it, adapt it, monetise it. Keep the attribution.

Copyright (c) 2026 Eoghan Henn / rebelytics.com
