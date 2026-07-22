# Bybit option margin capacity: payoff risk vs exchange IM

## Core distinction

A hedged bull-put spread can pass payoff max-loss limits while failing Bybit's short-option initial-margin check. Treat these as independent gates:

1. **Strategy/payoff gate:** spread width × qty, net credit, fees, aggregate portfolio cap.
2. **Exchange margin gate:** current Unified available balance versus the exact short leg's estimated/official initial margin after protective-long debit and fee.

A protective long can show `positionIM = 0` while each short PUT still consumes substantial `positionIM`; do not assume Bybit grants full spread offset in Regular Margin.

## Useful conservative estimate

When an official read-only order pre-check is unavailable, estimate short PUT IM from fresh index, strike and mark:

```text
short_put_IM_per_BTC ≈ max(
  10% × index − max(index − strike, 0),
  5% × index
) + option_mark

estimated_short_IM = short_put_IM_per_BTC × qty
estimated_total_needed = estimated_short_IM + protective_long_debit + protective_long_fee + buffer
```

This is an estimate, not a substitute for an exchange-supported exact pre-check. Compare it against observed `positionIM` on existing short PUTs; if the estimate does not reconcile closely, fail closed.

## Why a transfer can make a rejected recovery succeed

Before submit, Bybit may require available margin sufficient for the short IM without relying on the premium that the unfilled short would later credit. After fill, the resulting available-balance reduction is closer to:

```text
short_IM − short premium received + fees/mark effects
```

Therefore distinguish:

- **pre-submit funding requirement** — enough to pass order validation;
- **post-fill net margin consumption** — often lower because the premium is credited.

## Forecasting the next tranche

For tomorrow's feasibility:

1. Read current `totalAvailableBalance`, `totalInitialMargin`, `accountIMRate`, all short `positionIM`, positions and open orders.
2. Recalculate existing actual net credits from executions and conservative max losses.
3. Find a candidate using tomorrow's projected hours-to-expiry, but fresh current quotes only as an indicative scenario.
4. Enforce both short and long delta ranges; do not trust a scanner result if the implementation omits the long-delta gate.
5. Estimate exact candidate short IM plus protective-long cost and a safety buffer.
6. Account for expiry timing: margin is not released until the earlier position settles. A tranche expiring the day after tomorrow still consumes margin during tomorrow's scan.
7. Report separately:
   - payoff-risk capacity;
   - available margin;
   - candidate estimated requirement;
   - estimated shortfall;
   - earliest known margin-release time.

Never transfer funds or close an earlier spread merely to create capacity without separate explicit approval.
