# Take-profit close workflow and FOK recovery

Use this reference when a watchdog emits `TAKE_PROFIT` for a live Bybit BTC USDT bull-put tranche.

## Required response shape

Do not forward the raw alert. Refresh state with GET-only calls and report:

- estimated P&L if closed now;
- maximum P&L if both options expire worthless;
- incremental upside from waiting;
- close trading fees separately from buyback premium;
- short strike distance, delta, time to expiry, and margin released;
- a direct close/wait recommendation.

If closing is recommended, prepare an immutable single-use close plan and ask for a separate exact `CONFIRM`. No order POST occurs before that message.

## Close plan fields

Bind the digest to:

- exact tranche symbols and quantity;
- operation `CLOSE_BTC_USDT_BULL_PUT_SPREAD`;
- `SHORT_FIRST` execution order;
- short PUT `Buy` worst limit and FOK;
- long PUT `Sell` worst limit and FOK;
- maximum total close cost;
- minimum estimated P&L;
- fee cap and unique approval ID.

Record `PREPARED` in the append-only execution journal when the card is created. Consume it once immediately before the first POST.

## Safe execution sequence

1. Revalidate exact mainnet, permissions, positions, zero account-wide option open orders, instruments, tick/qty metadata, quotes, top size, and fees.
2. Buy back the short PUT first with Limit FOK.
3. Reconcile terminal order state and executions by immutable orderLinkId/orderId. Require full exact quantity.
4. If the short did not fully fill, send no long order. Position remains unchanged and hedged.
5. Only after the short is proven fully closed, refresh the long bid/size and sell the protective long with Limit FOK.
6. If the long sale fails, leave the residual long PUT; it is safe relative to naked-short risk. A new plan is required to retry.
7. Verify both target positions absent and zero open orders; report actual fills, fees, close cost, and realized strategy P&L.

## Approval lifecycle after an execution-process failure

Do not infer that `CONFIRM` was consumed merely because the local executor process was started. Reconcile the journal before deciding whether a new approval is required:

- `PREPARED` only, no `CONSUMED`, no order intent, and no POST: the immutable plan remains unused. Fix the local invocation/runtime issue, rerun all fresh pre-submit checks, and retry the same plan without asking for another `CONFIRM`.
- `CONSUMED` exists but no terminal result: never reuse the approval. Reconcile orders, executions, positions, and open orders by immutable identity before deciding the recovery path.
- Any order intent or POST may have reached Bybit: never blind-retry, even if the process exited with an exception.

This distinction prevents both duplicate approvals after harmless pre-submit failures and duplicate orders after ambiguous network/process failures.

## FOK no-fill handling

`EC_NoEnoughQtyToFill` with `cumExecQty=0` means no position change. Do not blindly retry and do not reuse the consumed approval. Refresh the book, prepare a new digest-bound plan with newly disclosed worst limits, show the changed minimum P&L/fees, and request a new exact `CONFIRM`.

A wider worst limit is acceptable only if the take-profit condition and minimum approved P&L still pass. Never widen silently after confirmation.

## Post-close follow-up

After successful close:

- refresh equity, available balance, total initial margin, positions, and open orders;
- update the watchdog tranche configuration immediately so it no longer monitors the closed expiry;
- run tests and validate the watchdog against the new live tranche set;
- size any replacement trade using adaptive residual margin capacity, preserving all remaining slots.
