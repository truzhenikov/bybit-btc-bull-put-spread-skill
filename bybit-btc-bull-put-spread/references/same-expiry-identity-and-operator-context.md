# Same-expiry tranche identity and terse close commands

## Why symbol-only pairing can become ambiguous

Two valid bull-put tranches can share one expiry while their strikes overlap. Example shape:

- tranche A: long 60,000 / short 62,500;
- tranche B: long 61,500 / short 64,000.

A generic matcher that pairs every lower-strike Buy PUT with every higher-strike Sell PUT can find multiple mathematically valid matchings. The account is hedged, but inferred tranche identity is ambiguous. A fail-closed `portfolio_preflight` may therefore reject a later entry even though the watchdog knows the intended pairs.

## Durable workflow

1. Treat immutable entry plans, execution-journal digests/order links, and the live watchdog tranche configuration as authoritative pair identity.
2. Reconcile each planned pair against live positions: exact symbol, side, expiry, and quantity; require zero unexplained legs and zero open option orders.
3. Do not create a LIVE entry card unless the actual executor can consume that authoritative identity safely. If the executor still uses combinatorial pairing and returns ambiguity, fix it and add regression tests first; analysis may remain GET-only.
4. A robust implementation should persist a tranche registry keyed by entry digest/approval and reconcile it against positions, rather than reconstructing identity solely from strike ordering.
5. Before opening a second tranche on an already-used expiry, test the resulting portfolio state for future unambiguous reconciliation. Prefer a different listed expiry when strategy gates allow it.

## Terse operator commands after alerts

Commands such as “закрывай” commonly refer to the most recently delivered watchdog/take-profit card, not necessarily the last trade discussed in the interactive transcript.

- Resolve the latest relevant delivered alert/card before naming a target tranche.
- Repeat the exact target legs and quantity before asking for approval.
- Never infer a different target from conversational recency alone.
- The command “закрывай” is intent, not the digest-bound approval token. Execute only after the exact separate `CONFIRM` required by the displayed close card.

## Expiry discovery

Do not assume every calendar date has a listed daily option. Before scanning a requested date:

1. Enumerate fresh instrument expiries and strikes.
2. If the requested expiry is absent, say it is not currently listed.
3. Show the nearest later listed expiry and its hours-to-expiry.
4. Reject it when outside the strategy’s 60–96 hour window; do not synthesize/interpolate an unlisted expiry.
