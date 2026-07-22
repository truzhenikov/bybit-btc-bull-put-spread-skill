# Security Policy

## Never commit

- Bybit API keys or secrets;
- RSA private keys;
- account IDs, subaccount IDs, or credential exports;
- live approval artifacts, execution journals, order IDs, or transfer IDs;
- private position snapshots.

Use placeholder paths and synthetic examples in documentation. Run a secret scanner before every public push.

## Recommended account controls

- dedicated trading subaccount;
- withdrawals disabled;
- least-privilege API permissions;
- IP allowlisting where available;
- credentials stored outside the repository with mode `0600`;
- no blind retries after API timeouts.

To report a vulnerability, open a GitHub security advisory rather than a public issue.
