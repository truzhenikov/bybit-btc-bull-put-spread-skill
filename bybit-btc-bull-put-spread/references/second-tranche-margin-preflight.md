# Second-tranche short-margin preflight and recovery

## Observed failure pattern

A second BTC USDT bull-put tranche passed payoff-risk, liquidity, fee, equity, available-balance and aggregate-risk checks. The executor then:

1. Bought the protective long PUT successfully with Limit FOK.
2. Reconciled the full long fill and fee.
3. Submitted the approved short PUT.
4. Bybit rejected the short with `retCode=110044` (`Insufficient margin balance`).

The account was left with an extra long PUT only. There was no naked short, but the tranche was incomplete.

## Root cause class

Defined payoff risk and exchange initial-margin sufficiency are different gates. Bybit can reserve short-option margin conservatively and may not grant enough spread offset at the moment/sequencing used. A positive Unified available balance is not enough: the exact short order can require more margin than remains after the protective-long debit and fee.

This is especially relevant for second/third tranches because an earlier spread already consumes margin while strategy equity remains close to its original value.

## Required pre-submit checks

Before buying the long:

- reconcile all positions and open orders account-wide;
- compute existing and aggregate payoff max loss;
- obtain fresh exact long ask / short bid, qty steps and fees;
- verify exact short-leg margin sufficiency using a supported Bybit pre-check/read-only facility, or a conservative current Bybit margin formula plus buffer;
- subtract the protective long debit and fee before declaring the short affordable;
- fail closed if the short-margin requirement cannot be established without a trade POST;
- never use a real short order as a margin probe.

## Capital separation

Use:

```text
actual_strategy_capital = min(approved_strategy_capital, current_equity)
```

Use `available_balance` only for:

- protective-long debit and fee;
- exact short initial-margin requirement;
- execution buffer.

Do not set strategy capital to `min(..., available_balance)`, because available balance falls after earlier positions and would distort payoff-risk limits.

## Recovery after long filled / short rejected

1. Do not retry the short blindly.
2. Reconcile journal, order identity, terminal state, executions, exact filled qty/price/fee, positions and open orders.
3. Explicitly state whether naked-short risk exists. With only an extra long PUT, it does not.
4. Mark the tranche `INCOMPLETE`; do not count it as a valid equal-leg spread.
5. Require a new plan and fresh one-time approval for exactly one recovery path:
   - add/transfer approved margin, then complete the exact short;
   - reduce spread qty: short no more than filled long, then close only the excess long;
   - close the residual long at a defined minimum sell price.
6. Re-run margin precheck before any recovery short.
7. Never reuse the consumed approval or digest.

## Regression tests

Maintain tests that prove:

- initial-entry CLI preflight still rejects existing positions;
- portfolio-aware LIVE preflight accepts only 1–2 unambiguous equal-qty hedged spreads;
- naked, mismatched, ambiguous and open-order states reject before POST;
- aggregate risk rejects before POST;
- low available balance blocks the exact long/margin gate but does not redefine strategy capital;
- simulated short rejection leaves no blind retry path.
