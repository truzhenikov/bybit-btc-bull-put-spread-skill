---
name: bybit-btc-bull-put-spread
description: Use when analyzing, opening, monitoring, closing, or scheduling short-duration BTC USDT-settled bull put credit spreads on Bybit through the dedicated RSA-authenticated AI Subaccount. Enforces Funding-to-Unified checks, three-tranche risk budgets, immutable digest-bound approval, protective-long-first execution, conservative option fees, read-only monitoring, and explicit fresh approval before every trade.
version: 1.1.2
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [bybit, btc, options, bull-put-spread, usdt, rsa, monitoring, trading]
    related_skills: [trading-account-monitoring-analysis, private-api-exchange-connectors]
---

# Bybit BTC USDT Bull Put Spread

## Overview

Этот skill описывает отдельный Bybit workflow для краткосрочных BTC USDT-settled bull put credit spreads. Не смешивать его с OKX: у Bybit другие символы, settlement, fee cap, account types, API signing и order semantics.

Основные гарантии:

- только BTC PUT с суффиксом `-P-USDT`;
- обе ноги одной expiry и одинакового BTC-объёма;
- long protective PUT исполняется первым;
- short PUT разрешён только после terminal-validated full fill long;
- максимум три независимых транша;
- максимум два новых исходных транша в календарный день стратегии, причём второй разрешён только после подтверждённого закрытия другого транша в тот же день по take-profit или stop-loss;
- analysis/read-only не является разрешением на ордера;
- каждый LIVE-план связан с canonical SHA-256 digest и одноразовым `CONFIRM`;
- timeout не приводит к слепому повтору POST;
- состояние сверяется через private GET, journal и immutable order identity.

## Когда использовать

Использовать, когда пользователь просит:

- найти BTC bull put spread на Bybit;
- открыть, проверить, закрыть или проконтролировать Bybit spread;
- проверить баланс AI Subaccount, позиции, fills, комиссии или доступный риск;
- перенести USDT из Funding в Unified для этой стратегии;
- настроить 15-минутный watchdog или ежедневный анализ торгового окна.

Не использовать для:

- OKX и других бирж;
- naked short PUT;
- автоматической торговли без новой карточки и отдельного `CONFIRM`;
- вывода средств;
- смешивания USDC/BTC-settled и USDT-settled options;
- повторного использования старого approval/digest.

## Локальные пути

```text
Project: $BYBIT_BPS_HOME
Credentials: ~/.config/bybit-bps/credentials.env
RSA private key: ~/.config/bybit-bps/private.pem
Execution journal: ~/.local/state/bybit-bps/execution-journal.jsonl
Transfer journal: ~/.local/state/bybit-bps/transfer-journal.jsonl
Watchdog source: $BYBIT_BPS_HOME/bybit_watchdog.py
Cron wrapper: ~/.hermes/scripts/bybit_bps_watch.py
Watchdog state: ~/.local/state/bybit-bps/watchdog-state.json
```

Credential profile и private PEM обязаны быть regular non-symlink файлами владельца runtime-user с mode `0600`. Никогда не выводить API key или PEM в чат, source, git, journal или traceback.

## Среда и API security

Допустимый MAINNET host только:

```text
https://api.bybit.com
```

Testnet включается отдельно и не может автоматически подменять mainnet. RSA signing:

- RSA-SHA256;
- PKCS#1 v1.5;
- `X-BAPI-SIGN-TYPE: 2`;
- signing payload точно соответствует Bybit V5 timestamp + api key + recv window + query/body.

API key должен проходить:

- Unified Trading Account;
- `OptionsTrade`;
- Read+Write;
- Withdraw отсутствует;
- отдельный AI Subaccount;
- IP allowlist, если доступен.

## Funding → Unified

Депозит может оказаться в Funding, тогда `/v5/account/wallet-balance?accountType=UNIFIED` покажет ноль и preflight обязан вернуть REJECT.

Read-only диагностика Funding:

```text
GET /v5/asset/transfer/query-account-coins-balance?accountType=FUND&coin=USDT
```

Внутренний перевод под тем же UID:

```text
POST /v5/asset/transfer/inter-transfer
```

Payload содержит только:

```json
{
  "transferId": "<fresh UUID>",
  "coin": "USDT",
  "amount": "<explicitly approved amount>",
  "fromAccountType": "FUND",
  "toAccountType": "UNIFIED"
}
```

Правила перевода:

1. После сообщения о пополнении сначала read-only проверить оба места хранения: Funding `transferBalance` и Unified `equity/totalAvailableBalance`. Не считать депозит доступным для options, пока он фактически находится в Funding.
2. Нужна отдельная явная команда пользователя с суммой и направлением. Короткое «Переведи» допустимо только как непосредственный ответ на предложенную агентом точную операцию вида «<точная сумма> USDT Funding → Unified»; не переносить согласие через смену темы или изменение суммы.
3. Непосредственно перед POST повторно сверить exact transferable Funding balance. Если он отличается от согласованной суммы — остановиться и запросить новое подтверждение; не округлять и не переводить «примерно всё».
4. Записать UUID и payload identity в transfer journal до POST.
5. Отправить ровно один POST.
6. При timeout/exception не повторять POST; reconciliation по transferId и балансам.
7. После `SUCCESS` подтвердить двумя GET: Funding transferable уменьшился на точную сумму, а Unified equity/available выросли. Расчёты сделки вести только от этих свежих значений.
8. Перевод не является подтверждением сделки; число trade requests должно оставаться нулевым.
9. Частичное пополнение принимать как новый доступный бюджет: не требовать довнести ранее рассчитанную «идеальную» сумму. Уменьшать qty по `qtyStep` и искать лучший транш, проходящий margin/risk/liquidity gates. Если даже минимальный qty не проходит — кратко показать фактический дефицит.

## Read-only preflight

Команда:

```bash
cd $BYBIT_BPS_HOME
python bybit_tool.py preflight
```

Preflight проверяет:

- exact mainnet host и RSA credentials;
- API permissions;
- positive Unified equity и available balance;
- все страницы account-wide option positions;
- все страницы account-wide active option orders;
- доступность BTC USDT option instruments;
- maker/taker fee rates;
- отсутствие Withdraw.

До первого транша preflight требует нулевые позиции и ордера. После появления действующих траншей нельзя трактовать этот initial-entry REJECT как поломку API: для следующего транша использовать отдельный **portfolio-aware execution preflight**, который:

- принимает только 1–2 однозначно сопоставленных полностью хеджированных bull-put транша;
- для каждого транша требует одинаковые expiry/qty, Buy PUT на нижнем strike и Sell PUT на верхнем strike;
- отклоняет naked, mismatched, ambiguous, unsupported позиции и любые активные option orders;
- считает conservative max loss каждого существующего транша по фактическим average fills и свежему index;
- проверяет `existing_max_loss + new_max_loss <= strategy_capital × 65%` и итог не более трёх траншей.

Начальный CLI preflight можно сохранить fail-closed для пустого аккаунта; LIVE executor второго/третьего транша не должен повторно использовать его вместо portfolio-aware проверки.

## Три транша и окно

Стратегия использует максимум три одновременно открытых транша. Обычно разрешён один новый исходный транш в календарный день. Второй новый транш в тот же день разрешён только если в этот же календарный день другой транш был terminal-validated полностью закрыт по take-profit или stop-loss, это закрытие подтверждено journal/API, после него нет остаточной ноги и open orders, а новый кандидат заново проходит все обычные gates. Закрытие вручную по иной причине, экспирация или освобождение слота без TP/SL не разрешают второй вход. Третий новый вход в тот же день запрещён. Каждому входу всё равно нужны отдельная свежая карточка и новый `CONFIRM`.

Базовое окно анализа и открытия:

```text
11:05 Europe/Moscow
```

Время ежедневного анализа должно быть явно настроено оператором. Пример — `11:05 Europe/Moscow`; если расписание биржи или expiry изменились, привязывать окно к подтверждённому времени expiry.

Окно expiry:

- предпочтительно 65–85 часов;
- допустимо 60–96 часов;
- меньше 48 часов для нового транша запрещено.

Risk budget:

```text
tranche_capital = strategy_capital / 3
per_tranche_max_loss_cap = strategy_capital × 0.65 / 3
portfolio_max_loss_cap = strategy_capital × 0.65
aggregate_after = sum(existing conservative max losses) + new conservative max loss

margin_utilization_cap = strategy_capital × 0.85
margin_reserve = strategy_capital × 0.15
per_tranche_initial_margin_cap = strategy_capital × 0.85 / 3
aggregate_initial_margin_after = current totalInitialMargin + estimated new spread initial margin
```

Размер позиции теперь **динамический**, а не фиксированный: проверять все qty по `qtyStep` и выбирать наибольший размер, который одновременно проходит payoff-risk и margin gates. Любая ранее названная требуемая сумма пополнения является только снимком на момент расчёта: после фактического пополнения заново читать equity, `totalInitialMargin` и `totalAvailableBalance`, а не прибавлять сумму вручную к старым цифрам. Новый транш запрещён, если:

- `estimated_new_initial_margin > per_tranche_initial_margin_cap`;
- `aggregate_initial_margin_after > margin_utilization_cap`;
- `estimated_new_initial_margin > totalAvailableBalance` до покупки long.

Если полной ранее рассмотренной qty не хватает маржи, scanner обязан попробовать меньшие размеры по `qtyStep`. Запрещено отклонять весь транш только потому, что пользователь пополнил меньше ориентировочной суммы; REJECT допустим лишь когда ни один разрешённый qty не проходит все gates. `totalAvailableBalance` сравнивать с полной conservative потребностью нового spread, включая short IM, long debit и long fee; будущую short premium заранее не учитывать.

Так сохраняются три маржинальных слота и минимум 15% капитала свободными. Не увеличивать первые транши за счёт будущих слотов.

Одного статического `per_tranche_initial_margin_cap` недостаточно, если существующий транш фактически занял больше планового слота или биржевая маржа изменилась. Перед каждым новым траншем дополнительно считать:

```text
remaining_slots_including_new = 3 - existing_tranches_count
residual_margin_capacity = strategy_capital × 85% - current totalInitialMargin
adaptive_new_margin_cap = min(
  strategy_capital × 85% / 3,
  residual_margin_capacity / remaining_slots_including_new
)
```

Подбирать максимальный qty, у которого `estimated_new_initial_margin <= adaptive_new_margin_cap`. Это оставляет равную фактическую маржинальную ёмкость каждому ещё не открытому траншу. После fill повторно читать `totalInitialMargin`: если observed consumption выше estimate, последующие размеры уменьшать по новой фактической ёмкости, а не продолжать старый план.

Предпочтительный риск транша: 15–20% strategy capital. Абсолютный cap: 21.6667%. Aggregate всех траншей не выше 65% strategy capital.

Для каждого нового анализа и нового immutable плана задавать `strategy_capital` по **свежему `totalEquity` Unified-счёта**, прочитанному непосредственно перед scan/карточкой. Не переносить в новый план старое значение капитала из предыдущего плана: рост equity должен увеличивать допустимый размер следующих траншей, а снижение equity — уменьшать его. Это является динамическим реинвестированием капитала; пополнения отдельного выделенного AI Subaccount также входят в его капитал. В самой карточке `strategy_capital` фиксируется как снимок свежего equity, а LIVE executor по-прежнему использует `min(plan_strategy_capital, current_equity)`, поэтому рост после подтверждения не расширяет одобренный риск, а падение автоматически ужесточает его. **Не уменьшать risk budget до `available_balance`** после открытия первого транша: available balance уже отражает занятую маржу и иначе ложно блокирует второй транш. `available_balance` использовать отдельно для проверки стоимости protective long, ожидаемой маржи short и операционного буфера. Увеличивать qty только дискретно по `qtyStep` и лишь если кандидат проходит все margin, payoff-risk, delta и liquidity gates; реинвестирование не является разрешением на ордер без новой карточки и отдельного `CONFIRM`.

Не дублировать уже используемую expiry, если следующая дневная expiry ликвидна и проходит gates. Пропущенный день не компенсируется двумя открытиями.

## Analysis-only scan

```bash
cd $BYBIT_BPS_HOME
python bybit_tool.py scan \
  --capital 600 \
  --qty 0.08 \
  --taker-fee-rate 0.0003 \
  --fee-cap-rate 0.07
```

Scanner обязан использовать fresh Bybit instruments/tickers и фильтровать:

- BTC only;
- PUT only;
- `settleCoin = USDT`;
- одинаковую expiry;
- long strike ниже short strike;
- short delta примерно `-0.10…-0.18`;
- long delta примерно `-0.02…-0.05`;
- executable long ask и short bid;
- достаточный top-of-book size;
- qty step, min/max qty и price tick;
- minimum net credit: `max(1 USDT, conservative max loss × 4.5%)`;
- conservative max loss;
- estimated Bybit regular-margin short PUT IM plus protective-long debit/fee;
- per-tranche margin cap `strategy_capital × 85% / 3`.

Для short PUT regular-margin estimate:

```text
short_IM = (max(10% × index − OTM_amount, 5% × index) + max(short_mark, short_limit)) × qty
new_spread_margin = short_IM + long_debit + conservative_long_fee
```

Это оценочный fail-closed gate. Если Bybit предоставляет более точный read-only order pre-check, использовать большее из pre-check и formula estimate.

Public scan не учитывает автоматически уже открытые транши. Перед предложением следующего транша вручную добавить их conservative max loss и проверить aggregate cap.

Short delta должна повторно проверяться при создании карточки и непосредственно перед LIVE submit. Если она вышла за `−0.10…−0.18`, обычный plan блокируется. Пользователь может отдельно согласовать strategy exception, но карточка обязана явно показать свежую delta, границу и факт исключения; существенное дальнейшее ухудшение delta после карточки требует новой карточки и нового `CONFIRM`, даже если limits цены ещё формально проходят.

Если стандартный scan вернул пустой список, не останавливаться на этом результате. Сначала перечислить fresh expiry ladder: не каждая календарная дата обязательно листингуется как daily option. Если запрошенной expiry нет вообще, прямо сообщить `не листится`, показать ближайшую следующую expiry и её часы до исполнения; не синтезировать и не интерполировать отсутствующую дату. Кандидат за пределами окна 60–96 часов остаётся запрещённым.

Для фактически листингованной запрошенной expiry:

1. Вывести листингованные страйки из fresh instruments; не предполагать, что промежуточный страйк (например, шаг $500) существует.
2. Проверить все пары и все допустимые размеры по `qtyStep`, а не только дефолтный `--qty`.
3. Кратко показать ближайшие short-кандидаты и единственную решающую причину отказа: delta, minimum credit, liquidity или risk cap.
4. Не называть отсутствующий страйк доступным и не интерполировать его котировку между соседними инструментами.

## Комиссии Bybit options

Account fee-rate GET является источником maker/taker rates. Для conservative pre-trade используется:

```text
fee_per_leg = min(
  account_fee_rate × BTC_index_price,
  premium_cap_rate × option_price
) × qty_BTC
```

Текущая conservative premium cap constant в проекте: `0.07`.

В observed fills дешёвых OTM PUT комиссия фактически составила около 6% gross option premium каждой ноги, хотя pre-trade модель намеренно резервировала до 7%. Поэтому:

- для approval использовать conservative 7% cap;
- после fill брать фактическую `execFee`/`execFeeV2` из executions;
- учитывать комиссии обеих ног при открытии и закрытии;
- не считать gross spread credit прибылью;
- отдельно показывать gross credit, opening fees, net credit и closing-fee reserve.

На дешёвых OTM options premium cap может доминировать над maker/taker rate. Поэтому maker order не гарантирует заметно меньшую комиссию; его главное преимущество — улучшение цены исполнения.

## Limit FOK против maker PostOnly/GTC

`Limit FOK`, который покупает по текущему ask или продаёт по текущему bid, является marketable limit и исполняется как taker. Само слово Limit не означает maker.

Чтобы пытаться получить maker:

- использовать `PostOnly` или пассивный GTC;
- long BUY ставить не выше текущего bid/внутри spread без пересечения ask;
- после полного long fill short SELL ставить не ниже текущего ask/внутри spread без пересечения bid;
- задать конечный execution window и небольшие разрешённые price steps;
- не пересекать утверждённые worst limits;
- при каждом изменении цены сохранять approved net-credit/max-loss constraints.

Current executor использует Limit FOK. Нельзя просто заменить FOK на maker mode без тестов и нового explicit plan. Безопасный maker workflow:

1. Passive protective long first.
2. Дождаться terminal full fill.
3. Только затем passive short объёмом не больше заполненного long.
4. Если long не заполнился — отменить его, short не отправлять.
5. Если long заполнен, а short не заполнился — остаётся только купленный PUT; не форсировать short market order без нового решения.
6. Partial long fill допускает short только на точно заполненный и валидный hedge volume, если такой режим заранее реализован и одобрен; иначе cancel/reconcile.
7. Любое изменение legs, qty или worst limits требует новой карточки/digest.

## Immutable approval и LIVE

Plan обязан включать:

- long/short symbols;
- long max buy limit;
- short min sell limit;
- qty BTC;
- unique approval ID;
- max-loss limit;
- strategy capital;
- trusted fee cap.

Workflow:

```bash
python bybit_tool.py prepare-confirmation \
  --plan /secure/path/approved-plan.json \
  --artifact /secure/path/confirmation.json
```

Показать пользователю полную карточку и canonical SHA-256. Только отдельное сообщение ровно:

```text
CONFIRM
```

разрешает:

```bash
python bybit_tool.py execute \
  --plan /secure/path/approved-plan.json \
  --artifact /secure/path/confirmation.json \
  --confirm CONFIRM
```

Execution повторно проверяет fresh account, instruments, tickers, liquidity, qty/tick metadata и submitted-limit risk. Любое ухудшение сверх approved limits блокирует сделку.

Approval одноразовый. После consume старый `CONFIRM`, artifact или digest нельзя использовать повторно.

### Превращение cron-кандидата в LIVE-карточку

Если пользователь отвечает на ежедневный `ANALYSIS ONLY` отчёт командой вроде «давай запрос на сделку», это означает: подготовить свежую immutable-карточку и запросить отдельный `CONFIRM`, но **не отправлять trade POST**.

Порядок:

1. Не копировать котировки, delta, equity, margin **или qty** из cron-сообщения как будто они всё ещё актуальны. Повторно выполнить fresh private `portfolio_preflight`, затем перебрать **все** разрешённые размеры по `qtyStep` от минимального до первого отказа и выбрать максимальную qty, одновременно проходящую liquidity, payoff-risk, adaptive-margin и available-balance gates. Проверка только cron-qty или только минимальной qty запрещена.
2. Проверить, что legs/expiry/максимальная допустимая qty всё ещё проходят delta, liquidity, minimum-credit, aggregate-risk, adaptive-margin и daily-entry gates. Если лучший кандидат, размер или top-of-book изменились, карточка должна содержать свежие limits и пересчитанные risk numbers; старый cron-снимок не сохранять ради совпадения.
3. Зафиксировать в plan свежий `totalEquity` как `strategy_capital`, exact fresh long max-buy / short min-sell limits, qty, trusted fee cap и max-loss limit, рассчитанный на этих submitted limits.
4. Создать новый уникальный approval ID, сохранить plan и вызвать `prepare-confirmation` для digest-bound single-use artifact.
5. До отправки карточки проверить artifact readback и наличие только `PREPARED` в execution journal. Не должно быть `CONSUMED`, `ORDER_INTENT` или trade requests.
6. В Telegram показать decision-critical LIVE-карточку: expiry, обе ноги и limits, qty, net credit, max loss, estimated margin, execution order, approval ID и полный SHA-256. Явно написать, что ордера ещё не отправлялись.
7. Запросить отдельное сообщение ровно `CONFIRM`. Фраза «давай запрос» разрешает только подготовку карточки и не заменяет подтверждение исполнения.

Если рынок изменится после карточки, LIVE executor всё равно обязан повторить fresh gates и fail closed при выходе за approved limits.

## Margin precheck до protective long

Положительный available balance и прохождение payoff-risk gates **не доказывают**, что Bybit примет short PUT. Особенно для второго/третьего транша биржевая initial margin short-leg может превышать остаток Unified balance даже при полностью определённом spread payoff.

До покупки protective long обязательно:

1. Получить свежие equity, available balance, `totalInitialMargin`, позиции, open orders, fees и котировки.
2. Рассчитать conservative Bybit short PUT IM по fresh index/mark и exact qty; при наличии поддерживаемого read-only order pre-check взять большее значение.
3. Добавить debit+fee protective long и проверить `new_spread_margin <= strategy_capital × 85% / 3`.
4. Вести **трёхслотовый ledger биржевой маржи**, а не только общий cap:
   - `slot_cap = strategy_capital × 85% / 3`;
   - при `k` уже открытых траншах кандидат проходит только если `current totalInitialMargin + estimated_new_margin <= (k + 1) × slot_cap`;
   - эквивалентно, после кандидата должны остаться полностью зарезервированы `3 − (k + 1)` маржинальных слота;
   - проверять это для первого и второго транша, а не обнаруживать нехватку лишь перед третьим.
5. Отдельно проверить общий предел `totalInitialMargin + new_spread_margin <= strategy_capital × 85%`, оставляя минимум 15% капитала свободными. Общий предел не заменяет трёхслотовый ledger.
6. Проверить `totalAvailableBalance >= new_spread_margin` **до** покупки protective long; будущую short premium не считать доступной заранее.
7. Если qty не проходит — уменьшать по `qtyStep`; нельзя занимать маржинальный слот следующего транша. Если даже минимальная qty нарушает ledger — `REJECT`, даже если Bybit технически принял бы ордер.
8. Если точную margin sufficiency нельзя подтвердить без пробного trade POST — `REJECT`; не покупать long «на пробу».
9. Для каждого qty повторить проверку после округления и непосредственно перед submit.
10. После fill обеих ног заново прочитать фактический `totalInitialMargin`, записать фактическое потребление слота и пересчитать два оставшихся слота. Если фактический IM вышел за бюджет уже открытых слотов, остановить новые входы и явно сообщить о нарушении ёмкости; не оправдывать следующий вход остатком payoff-risk.

Подробный разбор и recovery checklist: `references/second-tranche-margin-preflight.md`.

Для расчёта доступности следующего транша, conservative short-PUT IM estimate, различия pre-submit/post-fill margin consumption и учёта времени освобождения маржи: `references/bybit-option-margin-capacity.md`.

## Порядок исполнения и reconciliation

1. Persist order identity в journal до submit.
2. Protective long Limit FOK first.
3. Подтвердить terminal `Filled`, exact symbol/side/qty/price/orderLinkId/orderId.
4. Сверить executions, dedupe по execId и сумму qty.
5. Только после полного long fill отправить short.
6. Short qty не больше long filled qty.
7. После short сверить terminal state и executions.
8. При timeout не повторять POST; искать по immutable orderLinkId в realtime/history/executions.
9. После обеих ног заново прочитать positions, open orders и Unified wallet snapshot.
10. Завершать операцию только после тройной сверки:
   - execution journal содержит `CONSUMED`, обе записи `ORDER` и финальный `RESULT status=FILLED` для того же digest;
   - live positions показывают exact symbols/sides, одинаковый BTC size и фактические average fills;
   - account-wide active option orders равны нулю, naked short отсутствует.
11. Рассчитать и сохранить фактический opening credit только из terminal executions: `short_exec_value - long_exec_value - sum(all opening exec fees)`. Не использовать submitted limits или scanner estimate как фактический результат. Если short получил price improvement, показать actual net credit и отдельно не выдавать estimate за fill.
12. Сразу после verified fill добавить новый транш в authoritative tranche registry/watchdog с actual net credit, затем запустить tests и ручной read-only watchdog. Healthy watchdog: exit code 0 и пустой stdout. Если watchdog всё ещё содержит статический список старых траншей, сделка не считается полностью введённой в эксплуатацию.
13. При чтении wallet учитывать контракт локального client wrapper: если `client.wallet()` уже возвращает первую wallet-запись, не извлекать из неё `list` повторно; иначе equity/margin могут ложно отображаться как `null`. Проверять ожидаемые ключи `totalEquity`, `totalAvailableBalance`, `totalInitialMargin` перед публикацией snapshot.
14. Если short отклонён после полного long (например, `110044 Insufficient margin balance`): не повторять POST; reconciliate journal/order/executions/positions, зафиксировать фактический long fill и fee. Остаточный long PUT безопасен относительно naked-short риска, но транш считается `INCOMPLETE`. Любой следующий вариант — добавить margin, открыть меньший short и затем убрать excess long или закрыть long — требует новой точной карточки и нового одноразового `CONFIRM`.
15. Текущий `bybit_tool.py execute` принимает только обычную spread-схему (`long_symbol/long_limit/...`) и не исполняет recovery-план `COMPLETE_HEDGED_SHORT`. Не передавать recovery JSON в обычный executor. Для recovery нужен отдельный fail-closed path, который проверяет canonical digest/artifact, exact existing protective long, отсутствие target short/open orders, fresh bid/size/metadata/risk, consumes approval один раз, пишет `ORDER_INTENT` до единственного POST, reconciles terminal fill и после сделки подтверждает равные ноги и ноль open orders.

## Мониторинг

Позиция проверяется read-only watchdog каждые 15 минут. Scheduler wrapper:

```text
~/.hermes/scripts/bybit_bps_watch.py
```

Поведение:

- пустой stdout при норме — Telegram молчит;
- уведомление только при новом fingerprint набора alerts;
- повторяющийся неизменный alert не спамит;
- API/script failure должен дать scheduler error alert;
- никаких POST или автоматического закрытия.

Триггеры:

- одна нога отсутствует;
- sides или BTC sizes не совпадают;
- обнаружены неожиданные open orders;
- short delta по модулю достигла примерно `0.30`;
- BTC приблизился к short strike на 1% или оказался ниже;
- до expiry осталось ≤12 часов;
- estimated total close cost достиг take-profit threshold;
- estimated loss достиг stop-warning threshold;
- котировки недоступны.

При `TAKE_PROFIT` пользователь не должен получать голый watchdog alert. Agent-driven cron обязан повторно прочитать fresh positions/open orders/quotes/fees/margin, сравнить закрытие сейчас с ожиданием до expiry, показать фиксируемый P&L, максимум при благоприятной expiry, упускаемую выгоду, комиссии и освобождаемую маржу. Если сигнал всё ещё действителен, подготовить новый immutable single-use close plan/artifact и сразу запросить отдельный `CONFIRM`; до него число trade POST равно нулю. Закрывать short PUT первой и только после terminal full fill — long PUT. Пустой stdout watchdog остаётся бесшумным.

Полная close/retry/reconciliation схема, включая `EC_NoEnoughQtyToFill` и обязательный новый approval после FOK no-fill: `references/take-profit-close-workflow.md`.

После каждого нового транша watchdog constants/state должны быть обновлены или архитектура расширена до multi-tranche config. Нельзя считать старый position-specific watchdog универсальным для новых legs.

## Ежедневное расписание

В настроенное оператором время (например, `11:05 Europe/Moscow`) выполнить read-only workflow:

1. Прочитать текущие positions/open orders/equity/available.
2. Сверить действующие ноги и estimated close P&L.
3. Рассчитать aggregate conservative max loss.
4. Запустить fresh public scan.
5. Исключить уже занятую expiry.
6. Выбрать только кандидата в допустимом expiry/risk/liquidity окне.
7. Отправить краткий `ANALYSIS ONLY` отчёт в Telegram.
8. Не создавать и не исполнять ордера автоматически.
9. Для любой новой сделки нужна новая digest-bound карточка и новый `CONFIRM`.

One-shot или recurring cron prompt должен быть self-contained и иметь project workdir. Не включать credentials или текущие order IDs в prompt/skill; актуальное состояние читать из API/journal.

## Take-profit и risk alerts

После fill сохранить actual net credit из executions:

```text
actual_net_credit = short_exec_value - long_exec_value - all_opening_exec_fees
```

Базовая цель прибыли: 65% actual net credit.

```text
target_profit = actual_net_credit × 0.65
target_total_close_cost = actual_net_credit × 0.35
```

Estimated close cost должен использовать executable short ask, long bid и conservative closing fees. Watchdog только уведомляет. Закрытие обеих ног — отдельный новый plan/approval; short leg закрывается приоритетно или используется safe combo, чтобы не оставить naked short.

Stop-warning: estimated loss около 1.5× initial credit, short delta/strike proximity, expiry ≤12h или mismatch. Alert не является автоматическим разрешением на close/roll.

## Доходность и годовая экстраполяция

Для вопросов «сколько в день», «сколько годовых» и «почему капитал такой» не подменять текущий размер счёта значением `strategy_capital` из старого approval-плана.

1. Сначала read-only получить свежие `totalEquity`, `totalWalletBalance`, `totalAvailableBalance` и `totalInitialMargin`.
2. Явно разделить:
   - **account equity** — текущая база для доходности всего счёта;
   - **approved strategy capital** — риск-лимит, зафиксированный в плане и используемый execution gates;
   - **available balance** — технически свободная маржа, не капитал и не база доходности.
3. Если пользователь спрашивает годовую доходность без уточнения, считать простую annualized доходность от свежего account equity: `daily_profit × 365 / totalEquity`. При необходимости отдельно показать вариант от approved strategy capital, но не выдавать его за размер счёта.
4. Не называть дневную прибыль гарантированной. Для открытых траншей показывать отдельно: target каждой сделки (`actual_net_credit × 65%`), предполагаемое среднее при одном входе в день и фактически реализованный результат. Не усреднять цели двух траншей разных размеров так, будто это уже подтверждённый ежедневный P&L.
5. Годовую экстраполяцию помечать как арифметический сценарий без реинвестирования, убытков, простоев и проскальзывания; это не прогноз.

Pitfall: число из поля `strategy_capital` в последнем JSON-плане может быть старым согласованным risk budget и заметно отличаться от live equity после пополнений или P&L. Перед ответом о размере счёта или процентах всегда обновлять account snapshot.

## Быстрый operator UX

Для запросов «найди сделку», «других нет?», «почему не хватило маржи?», «пересчитай всё» и «открывай» отвечать максимально коротко:

- первая строка — прямой ответ на вопрос пользователя: найдено / нет / исполнено / не исполнено / завтра маржи хватит или не хватит;
- затем не более трёх коротких блоков: **причина**, **сейчас**, **следующий шаг**;
- показывать только decision-critical числа: available margin, required margin/shortfall, legs, qty, net credit, max loss и expiry;
- отдельно называть payoff-risk capacity и exchange-margin capacity, не смешивая их;
- не печатать audit IDs, digest, полный pre-trade шаблон или внутреннюю последовательность проверок, если пользователь этого не просил;
- если пользователь уже пожаловался на объём текста, не повторять объяснение другими словами и не добавлять общие сведения о стратегии;
- при задержке немедленно назвать конкретный blocker и подтвердить число отправленных trade requests.
- не начинать долгую доработку executor после команды «открывай», не сообщив сначала, что текущий execution path заблокирован.

Команда «открывай» не отменяет delta, liquidity, aggregate-risk и digest-bound approval gates. Если рынок изменил параметры после карточки, показать только изменившийся существенный параметр и подготовить новую карточку либо отказать. Никогда не трактовать старое согласие как разрешение на ухудшившуюся конструкцию.

Короткая команда «закрывай» после watchdog/cron-сообщения обычно относится к последней доставленной close-карточке, а не автоматически к последней сделке в интерактивном диалоге. Перед тем как назвать target, восстановить последнюю релевантную карточку/alert, повторить exact legs и qty и только затем запросить обязательный отдельный `CONFIRM`. Само «закрывай» выражает намерение, но не заменяет digest-bound token. Не выбирать транш только по conversational recency.

До показа исполнимой карточки второго/третьего транша проверить именно `portfolio_preflight`, а не initial-entry CLI preflight. Portfolio preflight обязан:

- принять только 1–2 однозначно сопоставленных fully hedged bull-put транша;
- отклонить naked/mismatched/ambiguous legs и любые open option orders;
- посчитать conservative max loss существующих траншей;
- проверить aggregate existing + new risk до создания approval artifact;
- использовать strategy capital/equity для risk budget, а available balance — только для проверки возможности купить protective long. Не уменьшать strategy capital до available balance после резервирования маржи существующими позициями.

Если два транша имеют одну expiry и перекрывающиеся strike ranges, простое combinatorial pairing по условию `long strike < short strike` может дать несколько допустимых сопоставлений. В таком состоянии нельзя создавать LIVE-карточку через executor, который умеет только угадывать пары: identity брать из immutable entry plans, execution journal/order links и tranche registry/watchdog config, затем exact-reconcile symbols/sides/qty с live positions. Если executor ещё не поддерживает authoritative identity, сначала исправить его и добавить regression test; GET-only анализ допустим, LIVE — fail closed. Перед открытием второго транша на уже занятую expiry проверять не только текущий вход, но и однозначность последующей reconciliation.

Подробности: `references/same-expiry-identity-and-operator-context.md`.

Подробная схема и regression checklist: `references/portfolio-aware-additional-tranches.md`.

## Tests и verification

```bash
cd $BYBIT_BPS_HOME
python -m unittest discover -s tests -v
ruff check .
python bybit_watchdog.py
```

Healthy manual watchdog run должен завершиться exit 0 с пустым stdout.

Перед завершением любой операции проверить:

- [ ] Bybit, а не OKX credentials/project
- [ ] exact mainnet host
- [ ] credential и PEM mode 0600
- [ ] Funding/Unified различены
- [ ] только BTC USDT PUT
- [ ] fresh option metadata и quotes
- [ ] positions/open orders account-wide и paginated
- [ ] aggregate risk включает все транши
- [ ] strategy capital ограничен approved capital/equity, а available balance проверяется отдельно
- [ ] до protective long подтверждена достаточная margin exact short-leg после long debit+fee
- [ ] комиссии обеих ног и closing reserve учтены
- [ ] approval digest соответствует immutable plan
- [ ] protective long полностью исполнен первым
- [ ] executions reconciled без blind retry
- [ ] post-trade legs равны, naked short отсутствует
- [ ] cron/watchdog read-only и не спамит
- [ ] следующий ежедневный анализ назначен на 11:05 МСК

## Operator communication

Операторские ответы по подбору и исполнению держать максимально короткими:

- сначала итог: `найдена / нет / исполнено / не исполнено`;
- затем только legs, qty, net credit, max loss, expiry и одна блокирующая причина;
- не публиковать длинный общий pre-trade шаблон, если все поля нормальны; полный audit сохранять локально, а в Telegram показывать только decision-critical данные;
- при задержке сразу назвать конкретный gate, который проверяется;
- после partial/incomplete execution первой строкой сообщить, какая нога исполнилась, какая нет, и есть ли naked-short риск.

## Common Pitfalls

1. **«Limit значит maker».** Marketable Limit FOK по ask/bid является taker.
2. **Ожидать огромную экономию комиссии от maker.** На дешёвых OTM options premium cap может доминировать; улучшение execution price важнее.
3. **Считать 600 USDT доступными, когда они в Funding.** Options требуют Unified.
4. **Повторить transfer/order POST после timeout.** Всегда reconcile по immutable ID.
5. **Продать short первой.** Защитная long должна быть terminal-filled.
6. **Использовать initial-entry preflight после появления позиции как единственный gate.** Он намеренно отклоняет существующие позиции; для следующего транша нужен portfolio-aware preflight.
7. **Не добавить существующий max loss к новому кандидату.** Public scanner не знает portfolio aggregate.
8. **Зашить текущие order IDs в skill или cron.** Это stale state; использовать API и journal.
9. **Оставить старый watchdog после новых legs.** Position-specific constants нужно обновлять или заменить multi-tranche config.
10. **Считать старый CONFIRM действующим.** Approval всегда одноразовый и plan-specific.
11. **Считать payoff max loss доказательством достаточной маржи.** Биржа может отклонить short с `110044` уже после fill long; precheck exact short margin обязателен до первой ноги.
12. **Использовать available balance как strategy capital.** Это двойное ужесточение после первого транша; available отдельно проверяет debit/margin, а risk budget строится от approved capital и equity.
13. **Молча продолжать после изменения delta.** Выход за согласованный диапазон или ухудшение после карточки требует явного exception/new approval.
14. **Доверять scanner output без повторной проверки long delta.** Если реализация scanner не применяет диапазон long delta `−0.02…−0.05`, она может вернуть формально прибыльную, но не соответствующую стратегии пару. Перед карточкой всегда повторно проверять обе delta из fresh ticker; такой пробел в scanner нужно исправлять и покрывать regression test.
15. **Смешивать payoff capacity и margin capacity.** Наличие свободного лимита max loss не означает, что есть Unified margin для следующего short PUT; показывать эти два остатка отдельно.
16. **Считать approval использованным только потому, что executor был запущен.** Сначала сверить journal. Если есть только `PREPARED`, нет `CONSUMED`/order intent и POST не отправлялся, после исправления локального запуска тот же immutable план можно повторить с полным fresh pre-submit recheck. Если есть `CONSUMED` или возможный POST — approval не переиспользовать, сначала reconcile orders/executions/positions по immutable identity.
17. **Выбирать транш для «закрывай» по последней обсуждавшейся сделке.** Сначала найти последнюю доставленную watchdog/close-карточку и повторить exact target; cron-alert может быть новее интерактивного контекста.
18. **Предполагать, что календарная дата обязательно листится.** Читать fresh expiry ladder; отсутствующую expiry не интерполировать, а следующую листингованную всё равно проверять на окно 60–96 часов.
19. **Восстанавливать две same-expiry позиции только по strike ordering.** Перекрывающиеся диапазоны дают неоднозначные пары; использовать immutable tranche identity из plan/journal/registry и блокировать LIVE, пока executor не умеет её exact-reconcile.
