# Grounded web research: from snippets to sources

*November 2026 · SYNCRONIX Research Agent · claude-sonnet-5-5*

## The problem

The first production run of the SYNCRONIX Research Agent finished with **4 searches, 7 sources and 0 pages read**.
The report looked well sourced: every source traced back to a real search result. But nothing had been read; the
report was assembled from search-result snippets.

The run's stored trace showed why:

- The system prompt said to fetch pages "when a search snippet is not enough". The model judged snippets enough.
  This matches documented behaviour: without a specific page to open, Claude tends to answer open-ended
  questions from search results.
- The `web_search_20260209` tool's dynamic filtering ran code over search results three times. One query was issued
  twice.

## Changes

1. **A research protocol in the system prompt** — plan, search, *read 2–4 pages* (always, even when snippets look
   sufficient), cross-check key claims across independent sources, mark single-source claims, then write. The prompt
   explains *why* reading matters (snippets are ranked excerpts that can be stale or misleading) rather than only
   commanding it.
2. **A read gate** — if the model ends its turn having read fewer than two pages, it gets one follow-up instruction
   to read and rewrite. Bounded to once per run and skipped if the run has used half its cost ceiling.
3. **Passage citations for fetched pages** (`citations: { enabled: true }` on `web_fetch`; off by default).
4. **Grounding metrics** computed on every report: pages read, findings citing a page that was read, unsourced
   findings, single-source findings, passage citations. Sources are flagged as *read* vs. merely *seen*.
5. **Tool versions** — compared below.

## Measurements

Two tasks, run live against the same local build:
- **T1:** *How does Solana's Proof of History work, and is it being replaced?*
- **T2:** *What is the current Node.js Active LTS release line and when does it reach end of life?*

| Configuration | Task | Searches | Pages read | Findings citing a read page | Repeated calls | Cited passages | Cost |
|---|---|---|---|---|---|---|---|
| Original prompt (production) | similar to T1 | 4 | **0** | — | 1 search | — | $0.082 |
| New prompt + `*_20260209` tools | T1 | 5 | 3 | 5/5 | 3 searches, several fetches | 0 | $0.106 |
| New prompt + `*_20260209` tools | T2 | 2 | 2 | **1/4** | several fetches | 0 | $0.085 |
| New prompt + `web_search_20250305` / `web_fetch_20250910` | T1 | 2 | 2 | 6/8 | none | 17 | **$0.069** |
| New prompt + `web_search_20250305` / `web_fetch_20250910` | T2 | 1 | 2 | 5/5 | none | 2 | **$0.040** |

Cost is the list-price estimate from the API's reported usage (uncached input, cache reads and writes, output,
plus $0.01 per search).

## Findings

- **The prompt change alone made the agent read** (0 → 2–3 pages per run).
- **With dynamic filtering, the model re-issued identical searches and fetches from code**: the same page was fetched
  up to four times. Repeats count against the per-run `max_uses`, so in T2 the agent ran out of fetches before it
  could open the official release notes, and 3 of 4 findings rested on snippets. Search repeats are also billed.
- **With the direct-call tool versions** there were no repeats, runs were 35–53% cheaper, every finding was linked,
  and fetched pages came back as citable documents (17 and 2 passage citations versus none).
- Reports became more honest about their gaps, e.g. *"One page I tried to read failed to load; I relied on its
  search snippet"* and *"I did not open nodejs.org directly."* The latter led to one more prompt rule: when an
  official source exists, at least one page read must be that source.

## Limitations

- Two tasks, one model, single runs per configuration: the direction of the effects is clear, their size is not.
- The original production run used a similar but not identical task.
- The final prompt rule (read the official source when one exists) has not yet been measured live.
- Grounding metrics check that findings *link* to read pages, not that the page *supports* the claim.

## Next

A larger task set with repeated runs, an automated check that cited passages support their findings, and the
official-source rule measured live. The harness is in
[syncronix-benchmarks](https://github.com/SyncronixLabs/syncronix-benchmarks).
