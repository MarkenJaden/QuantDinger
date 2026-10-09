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

---

## 4. Spot Position Ownership & Cross-Quote Reconciliation (Base Asset Matching)

- **Context & Motivation**:
  - In crypto spot trading (Binance, OKX, Bybit, Bitget, etc.), exchange wallets hold **base assets** (`BTC`, `SOL`, `ETH`, `USDC`), not contracts.
  - Upstream's `spot_wallet_snapshot.py` hardcoded `f"{ccy_u}/USDT"` when reading spot wallet balances.
  - When live spot strategies trade pairs with alternative quote currencies (such as `BTC/USDC`, `SOL/USDC`, or `ETH/EUR`), upstream's position reconciliation and ownership checks (`filter_position_rows_by_symbols`, `_load_ownership_rows`, `build_ownership_rows`) performed strict pair matching against `allowed_symbols`.
  - Because `canonical_symbol("BTC/USDT") != canonical_symbol("BTC/USDC")`, account spot inventory was filtered out to `0`, falsely flagging an allocation shortfall (`status = "drift_blocked"` / `"Entry paused" / "Position shortfall"`), marking strategy health as `Degraded`, and rejecting repair requests with `accountBelowStrategyAllocation`.
- **Divergence Details**:
  - **`backend_api_python/app/services/live_trading/spot_wallet_snapshot.py`**:
    - `_spot_position_row` and `list_spot_wallet_positions` accept dynamic `quote_currency` (defaulting to `"USDT"` for backward compatibility) and populate `"base_asset"` on every spot row.
  - **`backend_api_python/app/services/live_trading/account_positions.py`**:
    - `filter_position_rows_by_symbols`, `filter_legs_by_symbols`, and `list_account_positions_for_strategy` match spot holdings by base asset and remap them to the strategy's active trading pair (e.g. holding `BTC` covers `BTC/USDC`).
    - `reconcile_strategy_vs_account` aligns account holdings by base asset to active strategy allocations so cross-quote spot portfolios reconcile as `ok`.
    - `snapshot_rows_to_account_legs` preserves `base_asset`.
  - **`backend_api_python/app/services/live_trading/position_ownership.py`**:
    - `build_ownership_rows` aligns spot account holdings to allocated strategy pairs by base asset, avoiding false-positive shortfall flags.
  - **`backend_api_python/app/routes/strategy_position_ownership_routes.py`**:
    - `_load_ownership_rows` uses `filter_position_rows_by_symbols` with `market_type` awareness so the modal shows true account inventory and `Normal / ok` status.
  - **`backend_api_python/app/services/pending_order_position_sync.py`**:
    - Uses strategy's primary quote currency and maps base assets to strategy's `allowed_symbols` when syncing spot balances.
    - Implements auto-recovery in the sync cycle: checks any existing `status = 'drift_blocked'` reservations in `qd_position_reservations`, and automatically resets them to `status = 'ok'` when account balances cover the allocations, unpausing entries and clearing Degraded state.



