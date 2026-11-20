<p align="center">
  <img src="https://raw.githubusercontent.com/SyncronixLabs/SyncronixLabs/main/assets/syncronix-banner.jpg" alt="SYNCRONIX" width="100%">
</p>

<h1 align="center">SYNCRONIX Research</h1>

<p align="center">
  Engineering notes from building SYNCRONIX — each with the measurements behind it and its limitations stated.<br>
  <a href="https://syncronixai.xyz">Website</a> · <a href="https://github.com/SyncronixLabs/syncronix-docs">Docs</a> · <a href="https://github.com/SyncronixLabs/syncronix-benchmarks">Benchmarks</a> · <a href="https://github.com/SyncronixLabs/syncronix-docs/blob/main/whitepaper.md">Whitepaper</a> · <a href="CHANGELOG.md">Changelog</a>
</p>

---

## Notes

| Date | Note | Question |
|---|---|---|
| Nov 2026 | 🔬 [**Grounded web research**](notes/2026-11-grounded-web-research.md) | Why did a research agent produce a "sourced" report after reading zero pages, and what made it read and cite sources at lower cost? |
| Nov 2026 | ⚖️ [**Atomic budget admission**](notes/2026-11-atomic-budget-admission.md) | How do you open a paid LLM agent to the public with a hard daily budget that holds under concurrent requests? |
| Nov 2026 | 🔎 [**Proving a wallet's first funder**](notes/2026-11-wallet-funding-trace.md) | How do you show who funded a Robinhood Chain address, with evidence, keeping verified transfers apart from inferred links, on a free indexer allowance? |

## Key findings

<details open>
<summary><b>Grounded web research</b> — from snippets to sources</summary>

The first production run read **0 pages** while listing 7 sources. A research protocol that requires reading,
a bounded read gate, passage citations and direct-call tool versions changed that:

| Configuration | Pages read | Findings citing a read page | Repeated calls | Cost |
|---|---|---|---|---|
| Original prompt (production) | 0 | — | 1 search | $0.082 |
| Protocol + dynamic-filtering tools | 2–3 | 5/5, 1/4 | searches and fetches | $0.085–$0.106 |
| **Protocol + direct-call tools (current)** | **2** | **6/8, 5/5** | **none** | **$0.040–$0.069** |

Two tasks, single runs per configuration: the direction is clear, the effect size is not.
[Read the note →](notes/2026-11-grounded-web-research.md)

</details>

<details>
<summary><b>Atomic budget admission</b> — a hard budget under concurrency</summary>

Every run is admitted in one PostgreSQL transaction under an advisory lock, holding a $0.20 reserve while
active; settlement replaces the reserve with the real cost in one statement.

| Scenario (real PostgreSQL, production code) | Result |
|---|---|
| 25 simultaneous requests, 1 slot | exactly 1 admitted |
| 12 simultaneous requests, room for 2 reserves | exactly 2 admitted; spend + reserves ≤ budget |
| 6 simultaneous requests from one visitor | exactly 1 admitted |
| Run stuck past its time limit | closed by the next admission, charged its full reserve |

[Read the note →](notes/2026-11-atomic-budget-admission.md)

</details>

<details>
<summary><b>Proving a wallet's first funder</b> — verified transfers vs. inferred links</summary>

The first funder is the earliest of the address's first top-level inflow and every internal transfer before it; it
is marked *proven* only when the indexer's history up to that block is complete. Shared funders are reported as
*inferred*, never as proof of common ownership. Measured on production (19 Nov 2026, 5 addresses, no AI calls):

| Measure | Result |
|---|---|
| Activity, first request / cached | 4.0–5.8 s / 0.40–0.67 s |
| Funding trace, uncached | 1.7–7.6 s, 4–5 hops |
| First funder proven | testnet 1/1 · mainnet 0/4 (indexer still processing internal transactions) |
| Newest transaction matches the official RPC | 5/5 |
| Explorer links on Robinscan / testnet explorer | 379/379 |
| Opt-in AI analysis, measured | $0.0013–$0.0014 per brief (estimate shown first) |

[Read the note →](notes/2026-11-wallet-funding-trace.md)

</details>

## Standards for these notes

| Rule | Why |
|---|---|
| Every number comes from a recorded run or test | No estimates presented as measurements |
| Sample sizes and conditions are stated | Readers can judge how far a result generalises |
| Limitations get their own section | What was *not* measured matters as much as what was |
| Live and planned work are kept apart | Robinhood Chain data, Wallet Intelligence and wallet-approved transfers are live; autonomous AI execution is not |
| Verified and inferred are kept apart | A transfer with a link is verified; a pattern across transfers is inferred and labelled so |

Raw data and the harness that produced it: [syncronix-benchmarks](https://github.com/SyncronixLabs/syncronix-benchmarks).
The whole system in one document: [technical whitepaper](https://github.com/SyncronixLabs/syncronix-docs/blob/main/whitepaper.md).
History: [CHANGELOG.md](CHANGELOG.md).

<sub>Licensed under [CC BY 4.0](LICENSE).</sub>
