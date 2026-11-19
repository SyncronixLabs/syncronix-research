# Proving a wallet's first funder on Robinhood Chain

*November 2026 · SYNCRONIX*

## The question

Who sent a Robinhood Chain address its first ETH, who funded *that* address, and which other wallets share the same
funders? The answer has to be shown with evidence, separate what is a real transfer from what is only a pattern, run
within a free indexer allowance, and use no AI unless a visitor explicitly asks for a written summary.

This note covers how SYNCRONIX answers it in production (release `e73d8c5`), what went wrong on the way, and what is
still uncertain.

## Choosing an indexed-history source

Address history (transactions, and especially **internal** ETH transfers made by contracts) cannot be read from a
standard RPC endpoint. We checked the options for mainnet (chain 4663) in November 2026 with documented, permitted
test requests:

| Option | Free access for chain 4663 | Internal transfers | Result |
|---|---|---|---|
| Etherscan V2 API | No: requests without a paid plan were refused for this chain | Yes | Not used for data. Its explorer, **Robinscan** (`robin.etherscan.io`), is used for every public link |
| Blockscout hosted API (PRO) | Yes, with a free API key: 100,000 credits per day, 5 requests per second | Yes, with keyset pagination | **Used for mainnet data** |
| Blockscout public explorer API, testnet | Yes, no key | Yes | Used for testnet data |
| Blockscout mainnet explorer UI | Served behind a bot challenge to non-browser clients | — | Not used; the challenge was not bypassed |

The key stays on the server and is only added to upstream requests that match a fixed path allowlist.

## Proving the first inflow

"The oldest incoming transfer" is not enough: a wallet can first be funded by a contract (an internal transfer) that
never appears in its list of transactions. The rule we settled on:

1. **Bx** = the earliest block in which the address sent a transaction or received a top-level ETH transfer.
2. Read all internal ETH transfers to the address up to Bx, oldest first.
3. The earliest inflow of either kind is the first funder.
4. Mark it **proven** only if the indexer returned the internal-transfer history up to Bx completely. Otherwise show
   it as **earliest found**, with the reason.

Each funder is then confirmed against the official RPC: for a top-level transfer, sender, recipient and value must
match (`rpc`); for an internal transfer, the transaction must exist and have succeeded (`indexer+rpc_tx`). The same
rule runs on each funder in turn, up to 5 hops, and every trace ends with an explicit stop reason.

## What broke, and the fix

| Problem found with real data | Fix |
|---|---|
| Jumping to the oldest entry of the indexer's **balance history** took over 60 seconds for long-lived addresses | Dropped it. Keyset jumps on internal transactions (by block, index, transaction index) are fast enough to use on every lookup |
| A bridged deposit from Ethereum appeared as an ordinary transfer from an unknown L2 address | Arbitrum system transaction types (≥ 100) are labelled as bridge deposits; the L1 sender is recovered by undoing Arbitrum's address aliasing (subtracting `0x1111000000000000000000000000000000001111` modulo 2¹⁶⁰) |
| A tiny first inflow displayed as `0 ETH` | Inflows from one sender in one block are merged, internal transfers within one transaction are summed, and small amounts keep 9 decimal places |
| On mainnet, the indexer reports that *some internal transactions in this block range have not yet been processed* | Treated as incomplete data: no mainnet first funder is marked proven while that message is present. The UI and the AI fact sheet both say so |
| Indexer calls cost more credits than we first assumed | Measured from the `x-credits-remaining` header: **20–60 credits per call, about 30 on average**. The daily cap was set to 3,000 calls, and reads stop early when fewer than 2,000 credits remain |

## Verified versus inferred

Two claims look alike on a graph but are very different:

- *"Address B sent address A 0.03 ETH in transaction T"* is **verified**: one transfer, one link, checked on the RPC.
- *"A and C were funded by the same wallet"* is **inferred**: it is built from two verified transfers, but says
  nothing certain about who controls A or C.

So the trace keeps them apart everywhere: separate edge types in the graph (solid vs. dashed), separate sections in
the response and the UI, and separate sections in the AI brief. A first funder that paid **20 or more distinct
wallets** in its 50 most recent outgoing transactions is marked as a likely distribution service, because sharing it
is especially weak evidence. Owners are never named.

## Opt-in AI analysis

The AI brief is a summary of the deterministic data, not a source of facts:

- The server builds a fact sheet in code (about 29–38 lines and 15–23 links for the measured addresses) and makes one
  call to `claude-haiku-5-5`.
- Links and addresses in the answer that are not in the fact sheet are removed in code before display.
- The call is admitted exactly like a research run: per-visitor daily limit, shared daily budget with a reserve,
  one run in progress. A cost estimate is shown first, and nothing is sent until the visitor clicks.

**What went wrong first:** the first real call returned an **empty** answer. With adaptive thinking enabled, thinking
tokens count toward `max_tokens`, and a cap of 1,100 tokens was used up entirely by thinking. Fix: effort `low`, a
3,000-token cap, an estimate recalibrated so its maximum is a true ceiling, and an empty or cut-off answer is recorded
as failed with its real cost instead of being shown or replayed.

## Measurements

Production, 19 Nov 2026, 5 addresses (4 mainnet, 1 testnet), read-only public API, no AI calls. Raw data:
[syncronix-benchmarks](https://github.com/SyncronixLabs/syncronix-benchmarks/blob/main/results/2026-11-19-wallet-intel.json).

| Measure | Result |
|---|---|
| Activity, first request | 4.0–5.8 s (6 indexer calls each) |
| Activity, repeat within 60 s | 0.40–0.67 s (cached) |
| Funding trace, uncached | 1.7–7.6 s; upstream hops already traced for another address are reused from cache |
| Hops traced | 4–5; stop reasons: 4 × `contract`, 1 × `max_hops` |
| First funder proven | Testnet 1/1 · mainnet 0/4 (indexer still processing internal transactions) |
| Newest transaction matches the official RPC | 5/5 |
| Links pointing to the expected explorer | 379/379 |
| Responses containing an API key | 0 |
| AI estimate (no call made) | $0.00114–$0.00124, maximum $0.00202–$0.00215 |

AI analyses actually run during release testing (local build, real data):

| Run | Input tokens | Output tokens | Cost | Time | Links kept |
|---|---|---|---|---|---|
| Before the fix | 5,481 | 1,100 (all thinking) | $0.00110 | 5.9 s | 0 (empty answer) |
| After the fix | 4,670 | 1,909 | $0.00142 | 6.9 s | 12 |
| After the fix | 4,863 | 1,625 | $0.00130 | 6.1 s | 11 |

The two successful runs cost slightly more than the *typical* estimate shown to visitors and well under its stated
maximum.

## Limitations

- Five addresses and two successful AI runs: the latency and cost ranges are indicative, not statistically
  established.
- Several of the measured mainnet addresses share upstream funders, so their traces benefited from each other's
  cache. A cold trace for an unrelated address can take longer.
- Mainnet proof depends on Blockscout finishing its internal-transaction processing; until then, mainnet first
  funders are *earliest found*.
- Activity covers the most recent 150 transactions and 100 internal transfers of busy addresses, not their full
  history.
- Traces follow ETH only and stop at contracts, bridges and system addresses by design.
