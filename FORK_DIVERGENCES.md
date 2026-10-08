# Fork Divergences & Intentional Customizations

This document serves as the canonical handoff and divergence specification for the `MarkenJaden/QuantDinger` fork.
It is consulted during automated upstream synchronization and CI validation (via AI resolvers) to distinguish intentional customizations from accidental regressions or merge defects.

---

## 1. EU / German Regulatory Compliance (MiCA) & USDC/EUR Support

- **Context & Motivation**:
  - Under the European Union Markets in Crypto-Assets (MiCA) regulation, Binance accounts registered in Germany / EU face strict restrictions on Tether (USDT), preventing direct EUR deposits, fiat purchases, and spot trading with USDT.
  - German retail users utilize **USDC** (Circle) and **EUR** for spot trading on Binance.
  - Consequently, hardcoding `quote == "USDT"` renders QuantDinger unusable for compliant European users.

- **Divergence Details**:
  - **`backend_api_python/app/services/symbol_master_sync.py`**:
    - `STATIC_MARKET_ROWS` includes primary Binance USDC pairs (`BTC/USDC`, `ETH/USDC`, `SOL/USDC`, `BNB/USDC`, `XRP/USDC`, `DOGE/USDC`, `ADA/USDC`, `AVAX/USDC`, `LINK/USDC`, `DOT/USDC`, `SUI/USDC`, `NEAR/USDC`, `LTC/USDC`, `UNI/USDC`) and EUR pairs (`BTC/EUR`, `ETH/EUR`, `SOL/EUR`).
    - `fetch_context`: Ingests spot markets where `quote in ("USDT", "USDC", "EUR", "USD")` rather than strictly `quote == "USDT"`.
    - Dynamic currency assignments: `SymbolMasterRow` stores the dynamic `quote` currency instead of hardcoded `"USDT"`.
    - Fallback parser `_okx_public_payload_to_rows` also accepts `("USDT", "USDC", "EUR", "USD")`.
  - **`backend_api_python/app/services/market/symbol_search.py`**:
    - `_search_crypto_exchange`: Filters allow `info.get("quote") in ("USDT", "USDC", "EUR", "USD")`.
    - Search keyword sanitization strips `/USDC`, `/EUR`, `/USD` in addition to `/USDT`.
  - **Strategy Runtime & Preflight Compatibility**:
    - Strategy configurations targeting `@spot` with 0 leverage (e.g. `Crypto:BTC/USDC@spot`) are fully recognized in `qd_market_symbols` and pass preflight catalog checks (`get_catalog_product`).

---

## 2. Automated Upstream Sync & Deployment

- **Automated Upstream Synchronization (`.github/workflows/upstream-sync.yml`)**:
  - Automatically syncs from upstream `OpenByteInc/QuantDinger` (`main`) daily.
  - Automatically resolves non-divergent changes, and applies AI resolution via Gemini for conflicts, preserving the USDC/EUR fork features specified above.
- **Coolify Integration**:
  - Deployments are hosted on a self-hosted Coolify instance (`https://quantdinger.markenjaden.de`).
  - Merged and validated commits pushed to `main` trigger Coolify webhook rebuilds.
