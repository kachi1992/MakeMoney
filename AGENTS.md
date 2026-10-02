# Implementation instructions

## Scope and truthfulness

Read README and docs/00, 08, 12, 13, 14 before implementation. This is a design baseline, not implemented software. Do not mark a phase complete without executable artifacts, reproducible tests and explicit evidence. Code comments and identifiers should be English; user-facing copy and product documentation primarily Chinese.

## Architecture boundaries

- iOS talks only to our authenticated backend. No SEC, market-data, LLM or brokerage secrets in app bundles.
- Keep deterministic finance/scoring/risk/execution code separate from LLM extraction and narrative generation.
- No real-money trading, brokerage account linking, live endpoints or automatic production-strategy promotion in this scope.
- Public advice, data redistribution and AI processing require documented permissions and release gates. Feature flags cannot be used to conceal features from review.
- Start with a modular backend and workers, not a distributed multi-agent system.

## Invariants

1. Every factual claim resolves to a permitted source and version. Missing values are null plus a reason, never fabricated zero.
2. Every decision has an immutable data cutoff, strategy version, model/prompt version and snapshot hash. No future data in features, explanations or universe selection.
3. Signals are frozen before their eligible execution session. A missed deadline cancels execution rather than backdating it.
4. Only deterministic code may calculate financial metrics, score, weights, quantity, fill prices, returns and evaluation labels.
5. Ranking scores and evidence quality are not calibrated probabilities. Probability fields remain null until independently calibrated and validated.
6. Long-only paper portfolios: no negative positions, no negative spendable cash, no leverage, no fills during halts, no repeated fill on retry.
7. Append corrections and reversals, never silently overwrite recommendation, cash or fill history.
8. Maintain a canonical system portfolio independent of user edits. Archive and fork on resets or strategy changes; never erase losses from published histories.
9. Financial recommendations and simulated performance must be visibly labelled experimental. No guaranteed returns or cherry-picked win rates.

## Delivery workflow

Implement phase by phase with small reviewable changes. Each PR states requirement IDs, migrations, tests actually run, commands, environment, known gaps and screenshots when relevant. Unrun tests must be labelled unrun. Keep secrets and paid datasets out of fixtures; use examples/ synthetic inputs.

Pin toolchains and dependencies in P0; do not invent current SDK/API versions. Keep contract changes versioned and test Swift decoding against backend responses. For data edits preserve original source/recorded timestamps and lineage. For new strategy candidates record the entire trial history including failed candidates.

Before claiming ready: execute contract, finance, point-in-time, ledger replay, risk, authorization and UI-state tests described in docs/09. Public launch needs data-license, legal/compliance, privacy and App Store gates separately from software correctness.
