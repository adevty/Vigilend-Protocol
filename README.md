# Vigilend — Modular EVM Lending Protocol & MEV Liquidation Engine

[![Solidity](https://img.shields.io/badge/Solidity-0.8.24-363636?logo=solidity)](file:///home/zeth/Desktop/Flash-Loan-Bot/src/README.md)
[![Rust](https://img.shields.io/badge/Rust-1.70+-orange?logo=rust)](file:///home/zeth/Desktop/Flash-Loan-Bot/bot/README.md)
[![Foundry](https://img.shields.io/badge/Foundry-Framework-red?logo=ethereum)](https://getfoundry.sh/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](file:///home/zeth/Desktop/Flash-Loan-Bot/LICENSE)

**Vigilend** is an institutional-grade, overcollateralized EVM lending protocol coupled with a high-frequency, asynchronous Rust MEV liquidation bot. The system is designed from first principles with formal safety invariants, capital-efficient interest rate curves, zero-capital atomic flash loan liquidation execution, and automated bad-debt filtering.

---

## Documentation Subsystems

This monorepo contains dedicated documentation for each primary component:

* 📖 **[Smart Contracts Core (`src/README.md`)](file:///home/zeth/Desktop/Flash-Loan-Bot/src/README.md)**: Deep dive into the lending pool mechanics, Jump Rate Model formulas, price oracle validation, flash liquidation callbacks, and Foundry invariant tests.
* 🦀 **[Rust Liquidation Engine (`bot/README.md`)](file:///home/zeth/Desktop/Flash-Loan-Bot/bot/README.md)**: Architecture of the Tokio async searcher, dual execution strategies (direct capital vs flash loans), insolvent bad-debt filters, zero-downtime `.env` hot-reloading, and real-time ASCII quant telemetry.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           VIGILEND SMART CONTRACTS                          │
├──────────────────────────────┬──────────────────────────────┬───────────────┤
│ VigilendPool.sol             │ InterestRateModel.sol        │ MockOracle.sol│
│ - Deposit & Share Accounting │ - Jump Rate Model (Kinked)   │ - Price Feeds │
│ - Collateralized Borrowing   │ - Dynamic Borrow APY         │ - Stale Check │
│ - Flash Loans (0.09% Fee)    │ - Supply APY Curve           │               │
└──────────────▲───────────────┴──────────────▲───────────────┴───────┬───────┘
               │                              │                       │
               │  eth_call / Mempool Tx       │ Flash Loan Callback   │ Prices
               │                              │                       │
┌──────────────┴──────────────────────────────┴───────────────────────▼───────┐
│                      RUST MEV LIQUIDATION ENGINE (bot/)                     │
├──────────────────────────────┬──────────────────────────────┬───────────────┤
│ listener.rs                  │ evaluator.rs                 │ executor.rs   │
│ - EVM Log Filter             │ - Profitability Estimator    │ - Signer MW   │
│ - Active Borrower Tracking   │ - Bad Debt Loss Filter       │ - Hot-Reload  │
│ - Health Factor Monitoring   │ - Multi-Strategy Router      │ - Gas Tuning  │
└──────────────────────────────┴──────────────────────────────┴───────────────┘
```

---

## Monorepo Layout

```
├── bot/                             # High-frequency Rust liquidation searcher
│   ├── README.md                    # Dedicated Rust bot documentation
│   ├── Cargo.toml                   # Rust package manifest & dependencies
│   ├── .env.example                 # Bot environment template
│   └── src/
│       ├── main.rs                  # Async runtime & ASCII dashboard
│       ├── listener.rs              # EVM log filtering & borrower tracking
│       ├── evaluator.rs             # Solvency checks & bad debt filtering
│       ├── executor.rs              # Multi-strategy transaction dispatcher
│       ├── config.rs                # Dynamic configuration & hot-reload engine
│       └── types.rs                 # Domain models
├── src/                             # Solidity smart contracts
│   ├── README.md                    # Dedicated smart contracts documentation
│   ├── VigilendPool.sol             # Core lending & flash loan pool
│   ├── FlashLiquidationReceiver.sol # Receiver contract for atomic flash liquidations
│   └── interfaces/
│       ├── IVigilendPool.sol        # External pool interface & errors
│       ├── IVigilendOracle.sol      # Price feed contract interface
│       ├── InterestRateModel.sol    # Two-slope interest curve interface
│       └── IFlashLoanReceiver.sol   # Flash loan callback interface
├── test/                            # Foundry test suite
│   ├── VigilendPool.t.sol           # Core contract unit & fuzz tests
│   ├── InterestRateModel.t.sol      # Interest rate calculation tests
│   ├── invariant/                   # Stateful invariant solvency tests
│   └── mocks/                       # Mock ERC20 tokens and price oracles
├── script/                          # Foundry deployment scripts
│   └── Deploy.s.sol                 # Automated deployment & setup script
└── simulate.sh                      # Automated market stress-test & borrower simulator
```

---

## Key Protocol Parameters

| Parameter | Value | Description |
|---|---|---|
| **WETH Collateral LTV** | `75.0%` | Maximum borrow power against WETH collateral |
| **WETH Liquidation Threshold** | `80.0%` | Health factor inflection point ($HF < 1.0$) |
| **USDC Collateral LTV** | `85.0%` | Maximum borrow power against USDC collateral |
| **USDC Liquidation Threshold** | `90.0%` | Health factor inflection point for USDC |
| **Liquidation Bonus** | `5.0%` (500 bps) | Incentive bonus awarded to the liquidator |
| **Close Factor** | `50.0%` | Maximum debt repayable in a single liquidation transaction |
| **Flash Loan Fee** | `0.09%` (9 bps) | Fee charged on flash loan principal |

---

## End-to-End Quickstart

### 1. Prerequisites
* [Foundry](https://getfoundry.sh/) (`forge`, `cast`, `anvil`)
* [Rust](https://rustup.rs/) (1.70+)

### 2. Start Local Anvil EVM Chain
In **Terminal 1**:
```bash
anvil
```

### 3. Deploy Protocol Contracts
In **Terminal 2**:
```bash
forge script script/Deploy.s.sol --fork-url http://127.0.0.1:8545 --broadcast
```

### 4. Configure & Launch the Rust Bot
In **Terminal 3**:
```bash
cd bot
cp .env.example .env
cargo run --release
```

### 5. Run Market Simulation Traffic
In **Terminal 4**:
```bash
# Run continuous loop generating borrowers, borrows, and oracle price crashes
./simulate.sh --loop
```

Watch **Terminal 3** — the bot will continuously track state updates, identify undercollateralized accounts, and atomically liquidate them with live ASCII telemetry!

---

## Verification & Test Suite

| Target | Description | Command |
|---|---|---|
| **Solidity Build** | Compile all contracts and interfaces | `forge build` |
| **Unit & Fuzz Tests** | Execute unit tests and high-iteration fuzzing | `forge test` |
| **Invariant Solvency Tests** | Run multi-action stateful invariant tests | `forge test --match-path "test/invariant/**"` |
| **Solidity Formatting** | Check Solidity formatting standards | `forge fmt --check` |
| **Rust Unit Tests** | Verify bot evaluator, math, and encoding | `cargo test` (in `bot/`) |
| **Rust Clippy Linter** | Strict zero-warning lint check | `cargo clippy -- -D warnings` (in `bot/`) |
| **Rust Formatting** | Ensure Rust styling standards | `cargo fmt --check` (in `bot/`) |

---

## License

This project is licensed under the [MIT License](file:///home/zeth/Desktop/Flash-Loan-Bot/LICENSE).
