# Portfolio Intelligence

A portfolio platform built around **separate decision engines** rather than one giant trading brain.

```text
                         Portfolio Intelligence
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
 IBKR Investment Engine   Crypto Trading Engine   Capital Flow Engine
```

## Core rule

The engines share normalized portfolio state, but they **never share trading logic**.

- **IBKR Investment Engine** asks: _Should I own this asset, and at what portfolio weight?_
- **Crypto Trading Engine** asks: _Should I trade this asset now, with what risk and stop?_
- **Capital Flow Engine** asks: _Where will investable cash come from over the next 12 months?_
- **Portfolio Intelligence** aggregates risk and recommendations. It never places orders.

## Venue and strategy mapping

| Strategy | Venue | Role |
|---|---|---|
| `BTCStrategy` | Bybit | Directional BTC futures/swing trading |
| `ETHStrategy` | Binance | Directional ETH futures/swing trading |
| `SQQQStrategy` | IBKR | Tactical Nasdaq hedge |
| `GoldStrategy` | IBKR | Strategic gold allocation |
| `TLTStrategy` | IBKR | Duration/rates allocation |
| `StockAllocationStrategy` | IBKR | Long-term stock allocation |
| `SAPStrategy` | SAP employee plan | Employer-stock concentration and transfer policy |
| `GridManager` | Bybit/Binance adapters | Range-regime bot control |
| `HedgeManager` | Cross-venue | Net-exposure targeting |

## Architecture

```text
src/portfolio_intelligence/
├── collection/                # Scheduling, HTTP collector contracts, single-writer ingestion
├── portfolio/                 # Aggregation, effective exposure, portfolio warnings
├── capital_flow/              # Salary, SAP purchases, contribution forecast
├── investment_engine/         # IBKR long-term and tactical-hedge strategies
├── crypto_engine/             # BTC, ETH, grid and hedge strategies
├── integrations/              # Broker/exchange adapters only
└── shared/                    # Common models and protocols, no strategy logic
```

Strategies depend on broker protocols through dependency injection. A strategy does not import a concrete exchange client.

The Phase 1 collection runtime separates provider processes while preserving one SQLite writer:

```text
MarketDataSubscription
        ↓
scheduler-controller
        ↓ internal HTTP
Yahoo / IBKR / Bybit / Binance collectors
        ↓ normalized CollectionResult
SQLiteIngestionService
        ↓
SQLite
```

## Safety model

This repository starts in **recommendation-only mode**.

1. Engines produce typed `StrategyDecision` objects.
2. Portfolio Intelligence merges and prioritizes them.
3. A human approves an order.
4. An execution adapter submits it.

Do not put live API keys in source control. `.env` and local state are ignored.

## Quick start

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e '.[dev,portfolia]'
alembic upgrade head
pytest
portfolio-intelligence
```

## Docker Compose collection runtime

The same image runs the migration, scheduler, and four provider collector services:

```bash
cp .env.example .env
# Configure read-only IBKR, Bybit, and Binance access in .env.
docker compose build
docker compose up -d
docker compose ps
docker compose logs -f scheduler
```

Only `migrate` and `scheduler` mount the SQLite volume. Collector containers communicate over the
internal Compose network and never write the database directly. See
[`docs/deployment/docker-compose.md`](docs/deployment/docker-compose.md) for configuration,
operations, and current coverage.

## Read-only IB Gateway discovery

Install and sign in to IB Gateway, then open **Configure → Settings → API → Settings**.
Keep API access limited to localhost and keep **Read-Only API** enabled. The default Gateway
ports are `4002` for paper accounts and `4001` for live accounts.

Install the IBKR extra and configure the local connection:

```bash
pip install -e '.[dev,portfolia]'
cp .env.example .env
# Review IBKR_PORT before continuing; paper (4002) is the safe default.
portfolio-intelligence collect-ibkr
```

The command contains no order operation. It writes a sanitized, timestamped JSON capture under
the Git-ignored `state/raw/ibkr/` directory and stores the capture plus its normalized snapshot in
`state/portfolio.db`. Account identifiers are replaced with aliases and short fingerprints before
the payload reaches either storage layer.

Previously captured files can be safely reprocessed; payload hashes prevent duplicate snapshots:

```bash
portfolio-intelligence ingest-capture state/raw/ibkr/<capture>.json
```

Validate the latest stored snapshot before analytics or recommendations:

```bash
portfolio-intelligence validate
```

The command exits non-zero for a blocking validation failure. Warning-only checks are reported but
do not block the pipeline.

Import a monthly SAP employee-equity report and validate its dedicated snapshot:

```bash
portfolio-intelligence import-sap /path/to/PortfolioDetails.xlsx
portfolio-intelligence validate --source SAP_PLAN
```

Import Portfolio Details together with an EquatePlus Activity List to preserve purchases, employer
matches, dividend reinvestments, and sales in an idempotent transaction ledger:

```bash
portfolio-intelligence import-sap \
  --portfolio /path/to/PortfolioDetails.xlsx \
  --activity /path/to/Activity_List_SAP.xlsx
```

Either workbook can also be imported independently with `--portfolio` or `--activity`. Portfolio
Details remains authoritative for current lots, available shares, and weighted-average cost. The
Activity List explains changes and can project quantity beyond an older snapshot, but it does not
identify which lots a sale consumed.

Reconciliation compares the activity ledger at the Portfolio Details effective date with an
inclusive `0.0001`-share tolerance. Later activity is reported as a projection rather than a false
mismatch. A discrepancy above tolerance is retained for audit, printed as
`RECONCILIATION FAILED`, exits with status `2`, and blocks SAP analytics until corrected. Logs and
CLI output exclude participant names, user IDs, account identifiers, and workbook paths.

Calculate read-only FIFO and moving-average analytical estimates from reconciled SAP activity:

```bash
portfolio-intelligence sap-cost-basis --database-url sqlite:///state/portfolio.db
```

FIFO is the primary analytical estimate and moving average is a comparison. The command keeps the
latest Portfolio Details weighted-average cost separate and authoritative, performs no network or
database writes, and emits JSON labeled as gross of commissions, taxes, and withholding and not
authoritative tax accounting. Incomplete activity or blocked reconciliation exits non-zero instead
of returning partial P&L.

Record an SAP sell order already placed manually in EquatePlus so later read-only assessments can
consider it before execution appears in an Activity List:

```bash
portfolio-intelligence record-sap-order \
  --side SELL \
  --quantity 9 \
  --limit-price 190 \
  --good-till 2026-08-31 \
  --reference '<private-confirmation-reference>'
```

The command writes only normalized local state. It never places, changes, cancels, or queries an
EquatePlus order. The private reference is used for idempotency; normal output shows only its last
four characters. Portfolio Intelligence may later suggest keeping or reviewing the order and
following staged levels, but those recommendations remain non-executing.

SAP lots are classified as salary-funded purchases, employer matching compensation, or dividend
reinvestment. Reported values remain currency-unresolved until the report currency is confirmed,
so they are not silently combined with IBKR USD totals.

Collect delayed or last-observed `SAP.DE` and `EURUSD=X` quotes, validate them, and value the
latest plan quantity in EUR and USD:

```bash
portfolio-intelligence collect-sap-price
```

`sap-summary` refreshes both market inputs before calculating, so its default output always uses
the latest quote Yahoo makes available:

```bash
portfolio-intelligence sap-summary
```

For deliberate offline inspection, use stored quotes. The same freshness validation still applies:

```bash
portfolio-intelligence sap-summary --offline
```

Attribute the SAP position between personal contributions, employer compensation, reinvested
dividends, and market performance, including the gross personal-capital break-even price:

```bash
portfolio-intelligence sap-economics
```

The economic model is gross of tax, social-security charges, and fees. Observed matching rates are
derived from allocated lots and are not represented as contractual plan parameters.

Combine the validated economic model with a non-actionable technical assessment:

```bash
portfolio-intelligence sap-assessment
```

The output keeps three independent concepts: technical state (`STRONG`, `NEUTRAL`, or `WEAK`),
contribution status, and existing-share disposal status. Technical indicators cannot create a
`BUY`, `SELL`, `CONTINUE`, `REDUCE`, or `PAUSE` action. Contribution remains
`RESEARCH_REQUIRED` until scenarios and fundamental research are implemented; disposal remains
`POLICY_REQUIRED` until an explicit policy is approved.

Optional RSU vesting events can be supplied as capital-flow evidence without becoming automatic
sell signals:

```bash
cp config/sap_events.example.toml state/private/sap_events.toml
portfolio-intelligence sap-assessment --event-lookahead-days 30
```

The private event file is Git-ignored. Existing IBKR collection already includes average cost,
market price, market value, and realized/unrealized P&L, so the legacy live-order client and its
duplicate broker interface are intentionally not imported. Live order execution remains absent.

Print the combined read-only SAP daily report using the approved policy:

```bash
portfolio-intelligence sap-combined-report \
  --policy config/portfolio.example.toml \
  --database-url sqlite:///state/portfolio.db
```

Printing is the safe default. Add `--notify` only when the same sanitized report should also be
sent through the existing Telegram bot and chat configuration:

```bash
portfolio-intelligence sap-combined-report \
  --policy config/portfolio.example.toml \
  --database-url sqlite:///state/portfolio.db \
  --notify
```

The `[investment.sap_policy]` table controls the concentration comfort, reduction, and hard-cap
bands; the staged quantity and limit-price scenario; expiry review; daily-drop threshold; and the
moving-average period. The report never places, changes, or cancels an order. It includes only a
masked order-reference suffix and omits account identifiers.

Concentration requires fresh USD whole-account totals for IBKR, Bybit, and Binance in addition to
the reconciled SAP value. During the initial bridge, explicitly approved observations may use
`USER_SCREENSHOT` provenance with `temporary=true`; screenshot images and account identifiers are
not stored. These temporary observations are valid for seven calendar days. A stale or missing
provider is named in a precise incomplete-data notice, and no concentration percentage or
concentration-dependent advice is invented. Authoritative provider observations replace only the
matching provider's temporary total.

The command reads normalized stored inputs and does not perform an ad hoc market-data download.
Until a deterministic stored six-signal snapshot and prior-assessment history are available, the
six approved technical rows and prior-report change line are labeled unavailable instead of being
mixed with a different observation time.

Quotes are stored independently from the employee-plan snapshot with their Yahoo provider,
symbol, exchange, currency, observation time, and price type. Analytics is blocked for wrong,
non-positive, future-dated, or stale quotes. Weekend freshness is based on elapsed weekdays.
Yahoo Finance is suitable for periodic dry-run valuation but is not an execution-grade
market-data dependency.

Generate the long-term investment view containing only IBKR and SAP employee equity. Yahoo market
inputs are refreshed automatically; the latest IBKR snapshot must pass validation:

```bash
portfolio-intelligence portfolio-summary
```

The summary keeps SQQQ classified as a tactical hedge, SAP as employer equity, GLD as gold,
TLT/LQD as fixed income, and negative IBKR cash as financing. Binance and Bybit positions are not
part of this long-term allocation. SAP concentration is reported as information, not compared to
an arbitrary threshold. The summary calculates the quantity-weighted SAP entry price and
unrealized return first; a concentration policy will be added only after its decision rules are
defined explicitly.

## First implementation milestones

1. Normalize real positions from IBKR, Bybit and Binance.
2. Calculate effective BTC/ETH exposure including spot, futures and grids.
3. Implement read-only recommendations for BTC/ETH/SQQQ/GLD/TLT/SAP.
4. Add audit logging and human approval.
5. Add execution only after paper-trading tests pass.

See [`MIGRATION.md`](MIGRATION.md) for how the uploaded `trading-advisor` code maps into this repository.
