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
- **Image Publishing & Coolify Integration**:
  - `.github/workflows/docker-publish.yml` builds and pushes the backend image to `ghcr.io/markenjaden/quantdinger-backend:latest` on push to `main` (for linux/amd64).
  - Coolify is configured to run `BACKEND_IMAGE=ghcr.io/markenjaden/quantdinger-backend`, ensuring custom fork features (USDC/EUR symbols, search filters, MiCA support) execute in production.
  - Deployments are hosted on a self-hosted Coolify instance (`https://quantdinger.markenjaden.de`).

---

## 3. Binance Crypto Classification (bStock False Positives Fix)

- **Context & Motivation**:
  - The upstream classification logic in `app/services/market/instrument_products.py` contained a heuristic for legacy Binance bStocks (stock tokens): `exchange == "binance" and mt == "spot" and base.endswith("B") and base[:-1] in known_equities`.
  - In production, `known_equity_symbols` ingests over 10,000 US & HK stock tickers from Nasdaq, NYSE, and HKEX, including short tickers like `BN` (Brookfield), `AR` (Antero Resources), `SU`, `ST`, `VI`, `AM`, etc.
  - This heuristic falsely classified major cryptocurrencies on Binance ending with "B" (most notably **`BNB`**, **`ARB`**, **`SLB`**, etc.) as `tokenized_equity` rather than `crypto`.
  - When deploying live strategies targeting Binance `@spot` with `BNB` or `ARB`, Strategy V2 preflight rejected them with `strategyV2.equityProductVenueRequired` (*"Select the exchange explicitly for this exchange-listed equity product"*), because equity products require explicit venue declarations.
  - Binance permanently terminated stock tokens in October 2021; real bStocks only ever existed for ~6 large cap tech tickers (`TSLAB`, `COINB`, `AAPLB`, `MSFTB`, `MSTRB`, `NVDAB`).
- **Divergence Details**:
  - **`backend_api_python/app/services/market/instrument_products.py`**:
    - Added `crypto_native_bases` guard (`BNB`, `ARB`, `SHIB`, `SLB`, etc.) and `len(base) >= 5` requirement for `binance_bstock` matching.
    - Prevents standard 3- and 4-letter cryptocurrencies from colliding with equity ticker substrings.
  - **`backend_api_python/migrations/init.sql`**:
    - Includes automatic repair query on boot to reset falsely classified Binance spot equity rows in `qd_market_symbols` back to `product_type = 'crypto'` and `asset_class = 'crypto'`.


