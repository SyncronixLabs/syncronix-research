<p align="center">
  <img src="https://raw.githubusercontent.com/SyncronixLabs/SyncronixLabs/main/assets/syncronix-banner.jpg" alt="SYNCRONIX" width="100%">
</p>

<h1 align="center">SYNCRONIX Research</h1>

<p align="center">
  Engineering notes from building SYNCRONIX — each with the measurements behind it and its limitations stated.<br>
  <a href="https://syncronixai.xyz">Website</a> · <a href="https://github.com/SyncronixLabs/syncronix-docs">Docs</a> · <a href="https://github.com/SyncronixLabs/syncronix-benchmarks">Benchmarks</a>
</p>

---

## Notes

| Date | Note | Question |
|---|---|---|
| Nov 2026 | 🔬 [**Grounded web research**](notes/2026-11-grounded-web-research.md) | Why did a research agent produce a "sourced" report after reading zero pages, and what made it read and cite sources at lower cost? |
| Nov 2026 | ⚖️ [**Atomic budget admission**](notes/2026-11-atomic-budget-admission.md) | How do you open a paid LLM agent to the public with a hard daily budget that holds under concurrent requests? |

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

## Standards for these notes

| Rule | Why |
|---|---|
| Every number comes from a recorded run or test | No estimates presented as measurements |
| Sample sizes and conditions are stated | Readers can judge how far a result generalises |
| Limitations get their own section | What was *not* measured matters as much as what was |
| Live and planned work are kept apart | Robinhood Chain data is live; wallet transfers and execution are not |

Raw data and the harness that produced it: [syncronix-benchmarks](https://github.com/SyncronixLabs/syncronix-benchmarks).

<sub>Licensed under [CC BY 4.0](LICENSE).</sub>
