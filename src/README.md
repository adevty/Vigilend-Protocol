# Vigilend Smart Contracts (`src/`)

Welcome to the **Vigilend Protocol** smart contract core. Vigilend is an overcollateralized, modular EVM lending market engineered with safety-critical invariant guarantees, dynamic variable interest rates, oracle staleness protections, and atomic flash loan capabilities.

---

## Architecture Overview

The contract subsystem is structured around modular, single-responsibility components:

```
src/
├── VigilendPool.sol                 # Core lending pool: deposits, borrows, liquidations, flash loans
├── FlashLiquidationReceiver.sol     # Flash loan callback contract for atomic zero-capital liquidations
└── interfaces/
    ├── IVigilendPool.sol            # Lending pool external interface, structs, & custom errors
    ├── IVigilendOracle.sol          # Chainlink-compatible oracle adapter interface with timestamp checks
    ├── InterestRateModel.sol        # Kinked variable borrow & supply interest rate curve
    └── IFlashLoanReceiver.sol       # Standardized flash loan execution callback interface
```

### Core Contracts

| Contract | Purpose | Key Mechanics |
|---|---|---|
| [`VigilendPool.sol`](file:///home/zeth/Desktop/Flash-Loan-Bot/src/VigilendPool.sol) | Main protocol liquidity hub | ERC-4626 style internal share accounting, dynamic interest accrual via global borrow index, reentrancy guards, and debt repayment bounds. |
| [`FlashLiquidationReceiver.sol`](file:///home/zeth/Desktop/Flash-Loan-Bot/src/FlashLiquidationReceiver.sol) | Liquidation execution agent | Receives flash-loaned USDC debt capital, settles undercollateralized positions, captures bonus collateral (WETH), pays back flash loan + fee, transfers profit to operator. |
| [`InterestRateModel.sol`](file:///home/zeth/Desktop/Flash-Loan-Bot/src/interfaces/InterestRateModel.sol) | Jump Rate Model | Two-slope utilization curve ($U_{\text{kink}}$) ensuring steep rate spikes when liquidity is scarce. |
| [`IVigilendOracle.sol`](file:///home/zeth/Desktop/Flash-Loan-Bot/src/interfaces/IVigilendOracle.sol) | Price Feed Interface | Normalized 8-decimal asset pricing with staleness window checks to prevent stale oracle arbitrage. |

---

## Mathematical & Risk Specifications

### 1. Loan-to-Value (LTV) & Liquidation Threshold
* **WETH**: 75% LTV, 80% Liquidation Threshold ($LT$).
* **USDC**: 85% LTV, 90% Liquidation Threshold ($LT$).

### 2. Account Health Factor ($HF$)
An account is healthy when $HF \ge 1.0 \times 10^{18}$ and becomes liquidatable when $HF < 1.0 \times 10^{18}$:

$$HF = \frac{\sum (Collateral_i \times Price_i \times LT_i)}{\sum (Debt_j \times Price_j)}$$

### 3. Liquidation Rules & Incentives
* **Close Factor**: 50% max debt repayment per liquidation call to prevent unfair cascading liquidations.
* **Liquidation Bonus / Premium**: 5% ($500 \text{ bps}$) bonus paid to the liquidator from seized collateral.
* **Collateral Seized Formula**:

$$\text{Collateral Seized} = \frac{\text{Debt Repaid} \times \text{Price}_{\text{debt}} \times (1 + \text{Bonus})}{\text{Price}_{\text{collateral}}}$$

### 4. Jump Rate Interest Model
Utilization Rate $U$ is defined as:

$$U = \frac{\text{Total Borrows}}{\text{Total Cash} + \text{Total Borrows}}$$

* When $U \le U_{\text{kink}}$:
  $$\text{Borrow Rate} = \text{Base Rate} + \left(\frac{U}{U_{\text{kink}}} \times \text{Multiplier}\right)$$
* When $U > U_{\text{kink}}$:
  $$\text{Borrow Rate} = \text{Base Rate} + \text{Multiplier} + \left(\frac{U - U_{\text{kink}}}{1 - U_{\text{kink}}} \times \text{JumpMultiplier}\right)$$

### 5. Flash Loan Fee
* **Fee**: 0.09% (9 basis points) calculated on borrowed principal.
* Repayment condition: $\text{Amount Repaid} \ge \text{Principal} + \text{Fee}$.

---

## Directory Structure

```
├── src/
│   ├── VigilendPool.sol
│   ├── FlashLiquidationReceiver.sol
│   └── interfaces/
│       ├── IFlashLoanReceiver.sol
│       ├── InterestRateModel.sol
│       ├── IVigilendOracle.sol
│       └── IVigilendPool.sol
├── script/
│   └── Deploy.s.sol                 # Production & Anvil local deployment script
└── test/
    ├── VigilendPool.t.sol           # Core pool unit & fuzz tests
    ├── InterestRateModel.t.sol      # Mathematical verification of rate curves
    ├── invariant/
    │   ├── InvariantTest.sol        # Stateful invariant tests (solvency, accounting)
    │   └── Handlers.sol             # Action generator for randomized invariant fuzzing
    └── mocks/
        ├── MockERC20.sol
        └── MockOracle.sol
```

---

## Testing & Verification

### Running Tests
Execute the comprehensive Foundry test suite:

```bash
# Run all unit and integration tests
forge test

# Run tests with detailed stack traces
forge test -vvvv

# Run stateful invariant fuzz tests (solvency & balance consistency)
forge test --match-path "test/invariant/**" -vvv
```

### Static Analysis & Formatting
```bash
# Check code formatting
forge fmt --check

# Format code automatically
forge fmt
```

---

## Deployment & Interaction

Deploy contracts to a local Anvil node or testnet:

```bash
# 1. Start local node
anvil

# 2. Deploy all protocol contracts, mocks, and liquidation receiver
forge script script/Deploy.s.sol \
  --fork-url http://127.0.0.1:8545 \
  --broadcast
```

The deployment script registers supported assets (`WETH`, `USDC`), sets up initial oracle prices, and outputs the contract addresses needed for the off-chain bot `.env`.
