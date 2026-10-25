# Vigilend Liquidation Engine (`bot/`)

High-frequency, asynchronous MEV searcher service engineered in **Rust** using the [Tokio](https://tokio.rs/) runtime and [Ethers-rs](https://github.com/gakonst/ethers-rs). 

The bot monitors the [Vigilend Protocol](file:///home/zeth/Desktop/Flash-Loan-Bot/src/README.md) 24/7, detects undercollateralized borrow positions, computes optimal liquidation routes, filters out insolvent bad debt, and executes profitable liquidations using either direct wallet capital or zero-capital atomic flash loans.

---

## Key Features

1. **Dual Execution Strategies**:
   - **Direct Capital Liquidation**: Uses liquidator's own capital to directly call `VigilendPool.liquidate(...)`.
   - **Atomic Flash Loan Liquidation**: Calls `VigilendPool.flashLoan(...)` routed to [`FlashLiquidationReceiver.sol`](file:///home/zeth/Desktop/Flash-Loan-Bot/src/FlashLiquidationReceiver.sol), borrowing USDC atomically to repay borrower debt, seizing WETH collateral, and repaying the flash loan in a single transaction.
2. **Insolvent Bad Debt Protection**:
   - Protects the bot from executing negative-EV transactions where borrower collateral has plunged below the required repayment amount ($TotalCollateralUSD \le DebtToCoverUSD$).
3. **Zero-Downtime Hot Reloading**:
   - Monitors `bot/.env` on disk. When settings change (e.g. toggling `USE_FLASH_LOAN=true` or adjusting `MIN_PROFIT_USD`), the engine reloads the configuration in-memory without restarting or missing mempool/block events.
4. **Real-Time ASCII Quant Terminal UI**:
   - Formatted 78-column live telemetry displaying timestamp, target borrower, Health Factor ($HF$), strategy route, repaid debt, profit estimate, flash loan fee, and confirmed transaction hash.

---

## Engine Architecture

```
bot/src/
├── main.rs          # Tokio async runtime, main scanning loop & ASCII terminal dashboard
├── listener.rs      # EVM log filters (Deposit, Borrow, Liquidate) & borrower registry
├── evaluator.rs     # Liquidation profitability evaluation & bad debt filter
├── executor.rs      # Transaction encoder, signer middleware, gas estimation & broadcast
├── config.rs        # Configuration parser with disk hot-reloading
├── types.rs         # Data models (UserAccountSummary, LiquidationOpportunity, etc.)
└── lib.rs           # Public library exports for integration & unit testing
```

---

## Configuration (`bot/.env`)

Copy the example file and customize your local or live RPC settings:

```bash
cp bot/.env.example bot/.env
```

| Parameter | Type | Description | Default |
|---|---|---|---|
| `RPC_URL` | String | JSON-RPC provider (HTTP / WebSocket) | `http://127.0.0.1:8545` |
| `POOL_ADDRESS` | Address | Deployed address of `VigilendPool` | `0x9fe...` |
| `ORACLE_ADDRESS` | Address | Deployed address of `MockOracle` / Chainlink Feed | `0x5fb...` |
| `WETH_ADDRESS` | Address | Collateral asset address (WETH) | `0xcf7...` |
| `USDC_ADDRESS` | Address | Borrow/debt asset address (USDC) | `0xdc6...` |
| `BOT_ADDRESS` | Address | Searcher wallet address | Deployer / searcher pubkey |
| `PRIVATE_KEY` | HexString | Signer private key for executing liquidations | Local Anvil key |
| `FLASH_RECEIVER_ADDRESS` | Address | Deployed `FlashLiquidationReceiver` contract | `0x5fc...` |
| `USE_FLASH_LOAN` | Boolean | `true` for capital-free flash loans, `false` for direct capital | `true` |
| `SIMULATION_ONLY` | Boolean | `true` to dry-run `eth_call` without broadcasting on-chain | `false` |
| `MIN_PROFIT_USD` | Integer | Minimum net USD profit threshold required to trigger execution | `10` |

---

## Quickstart

### Prerequisites
* Rust toolchain (stable, 1.70+): `rustup update stable`
* Running EVM node (Anvil or testnet) with deployed Vigilend contracts.

### Running the Bot
```bash
# Navigate to the bot directory
cd bot

# Run in debug mode
cargo run

# Run in release mode for maximum throughput & low latency
cargo run --release
```

---

## Testing & Code Quality

The bot includes comprehensive unit tests verifying evaluation logic, bad debt pruning, profit formulas, and config loading.

```bash
# Run all unit tests
cargo test

# Run tests with output logging
cargo test -- --nocapture

# Run strict Clippy linter
cargo clippy -- -D warnings

# Check code formatting
cargo fmt --check
```

---

## Telemetry & Dashboard Output

When active, the bot outputs structured ANSI-formatted execution telemetry:

```
┌───────────────────────── VIGILEND SEARCHER TELEMETRY ─────────────────────────┐
│ Status: ACTIVE SCANNING          Strategy: FLASH LOAN ATOMIC REPAY            │
│ Target: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8                            │
│ Health Factor: 0.8412 (LIQUIDATABLE)                                          │
│ Debt Repaid:   $12,500.00 USDC                                                │
│ Seized WETH:   4.375 WETH ($13,125.00 USD)                                    │
│ Gross Profit:  $625.00 USD (5.00% Liquidation Incentive)                      │
│ Flash Fee:     $11.25 USD (0.09% Borrow Fee)                                  │
│ Est. Net Gain: $613.75 USD                                                    │
│ Tx Confirmed:  0x3a4f8b...c7e2 (Block #124)                                   │
└───────────────────────────────────────────────────────────────────────────────┘
```
