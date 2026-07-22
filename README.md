# Bybit BTC Bull Put Spread Skill

A safety-first Hermes Agent skill for analyzing, opening, monitoring, and closing short-duration **BTC USDT-settled bull put credit spreads on Bybit**.

## What it provides

- protective-long-first execution;
- immutable, single-use approval artifacts;
- portfolio-aware limits for up to three hedged tranches;
- liquidity, delta, payoff-risk, and exchange-margin gates;
- timeout-safe reconciliation without blind order retries;
- read-only watchdog guidance and take-profit/stop-warning workflows;
- fee-aware realized P&L accounting.

This repository contains an **agent operating playbook**, not API credentials or a turnkey trading bot. Command names in the skill assume a compatible local implementation and should be adapted to your own project.

## Install for Hermes Agent

```bash
git clone https://github.com/truzhenikov/bybit-btc-bull-put-spread-skill.git
mkdir -p ~/.hermes/skills/trading
cp -R bybit-btc-bull-put-spread-skill/bybit-btc-bull-put-spread ~/.hermes/skills/trading/
```

Start a new Hermes session after installation so the skill catalog is refreshed.

## Security

- Never commit API keys or RSA private keys.
- Use a dedicated subaccount with withdrawals disabled.
- Keep credential files mode `0600` and outside the repository.
- Treat every LIVE order as requiring a fresh immutable plan and explicit confirmation.

See [SECURITY.md](SECURITY.md) before adapting the workflow.

## Disclaimer

This project is for educational and operational-safety purposes. Options trading can result in substantial losses. It is not financial advice and provides no warranty of profitability.

## License

MIT — see [LICENSE](LICENSE).
