# Security policy

## Reporting a vulnerability

Please report security problems **privately** through GitHub: open the **Security** tab of this repository and choose
**Report a vulnerability**. Do not open a public issue, pull request or discussion for a security problem.

Please include:

- what is affected: the production service at syncronixai.xyz, its public API, or code in this repository;
- steps to reproduce;
- the impact you expect.

Reports are read by the maintainer of the SyncronixLabs account. There is no bug bounty.

## Scope

| In scope | Out of scope |
|---|---|
| syncronixai.xyz and its public API | Denial of service and load testing |
| Code and documentation in SyncronixLabs repositories | Social engineering, physical attacks |
| Wallet-transfer previews that could mislead a signer | Findings in third-party services (Anthropic, Blockscout, Robinhood Chain RPC, wallets): report those to their owners |

Please test only with your own wallets and your own daily run, and never send other people's funds or data.

## How SYNCRONIX is secured

The controls, known limitations and threat model are documented in
[security.md](https://github.com/SyncronixLabs/syncronix-docs/blob/main/security.md). In short:

- no wallet keys on the server;
- read-only chain access;
- same-site checks on every `POST` endpoint;
- an atomic daily budget for every model call.
