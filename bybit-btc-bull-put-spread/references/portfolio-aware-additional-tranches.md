# Portfolio-aware additional-tranche execution

Use this reference when initial-entry preflight rejects because a valid spread already exists.

## Required split

- Keep the operator/CLI initial-entry preflight fail-closed when any option position exists.
- Use a separate execution-only portfolio preflight for adding tranche 2 or 3.

## Existing-position validation

Accept only 0–2 unambiguous pairs where each pair has:

- BTC PUT, USDT settlement;
- same expiry;
- Buy leg at lower strike;
- Sell leg at higher strike;
- equal positive BTC size;
- positive average prices.

Reject duplicate symbols, unsupported instruments, unknown sides, odd leg counts, quantity mismatch, naked legs, ambiguous pairings, more than two existing tranches, or any active option order.

## Risk accounting

For every accepted pair, calculate conservative max loss from width, observed average prices, account taker fee, fixed premium fee cap, and a closing-fee reserve. Sum all existing max losses.

For a proposed spread enforce both:

```text
new_max_loss <= min(plan_strategy_capital, live_equity) * 0.65 / 3
existing_max_loss + new_max_loss <= plan_strategy_capital * 0.65
```

`available_balance` is not strategy capital. Use it only to prove the protective-long debit plus fee can be paid.

## Approval ordering

Complete portfolio preflight and fresh quote/risk calculations before preparing the immutable approval artifact. Execution still requires exact one-use `CONFIRM`, repeats fresh checks, buys the protective long first, and never blindly retries a POST.

## Regression tests

Cover:

- initial-entry CLI still rejects existing positions;
- execution accepts valid second and third tranches;
- naked, mismatched, unsupported and ambiguous portfolios reject before POST;
- aggregate risk rejects before POST;
- reduced available balance does not shrink strategy risk capital when it still covers the protective long;
- insufficient available balance rejects the protective long;
- no test sends a real request.
