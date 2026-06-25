# BlackForest BSC-20 — Architecture

> A BEP-20 (BSC-20) token contract on the Binance Smart Chain. Features redistribution rewards, auto-liquidity, burn, and multiple fee categories with different rates for buy and sell transactions.

| | |
|---|---|
| **Network** | Binance Smart Chain (BSC) |
| **Standard** | BEP-20 (BSC-20) |
| **Compiler** | Solidity 0.8.4 with optimization |
| **Symbol** | BFT |
| **Decimals** | 9 |
| **Total Supply** | 1,000,000,000,000 (1 trillion) |
| **Max Transaction** | 1% of total supply |
| **Max Wallet Balance** | 2% of total supply |

---

## Contract Architecture

```mermaid
flowchart TB
  subgraph Contract["BlackForest.sol"]
    TOKEN["BlackForest<br/>BEP-20 Token"]
    TOKENOMICS["Tokenomics<br/>abstract contract<br/>fee settings + constants"]
    UTILS["Utils<br/>SafeMath utilities"]
  end

  subgraph Imports["blackforest-imports.sol"]
    SAFEMATH["SafeMath"]
    CONTEXT["Context"]
    OWNABLE["Ownable"]
    IBEP20["IBEP20 interface"]
    BEP20["BEP20 base"]
  end

  subgraph Fees["Fee System"]
    FEE_LOGIC["FeesSettings<br/>dynamic fee configuration"]
    LIQUIFIER["Liquifier<br/>swap and liquify mechanism"]
  end

  TOKEN --> TOKENOMICS
  TOKENOMICS --> UTILS
  TOKEN --> BEP20
  BEP20 --> IBEP20
  BEP20 --> SAFEMATH
  BEP20 --> CONTEXT
  BEP20 --> OWNABLE
  TOKEN --> FEE_LOGIC
  TOKEN --> LIQUIFIER
```

---

## Tokenomics — Fee Structure

Different fee rates apply to buy and sell transactions. Fees are deducted on each transfer and routed to their respective destinations.

```mermaid
flowchart LR
  subgraph Buy["BUY Transaction Fees  (total 10.5pct)"]
    B_LIQ["Liquidity 1pct"]
    B_REDIST["Redistribution 2.5pct"]
    B_BURN["Burn 1.5pct"]
    B_CHARITY["Charity 1.5pct"]
    B_MARKETING["Marketing 1.5pct"]
    B_DEV["Dev Tip 4pct"]
  end

  subgraph Sell["SELL Transaction Fees  (total 15pct)"]
    S_LIQ["Liquidity 1pct"]
    S_REDIST["Redistribution 4pct"]
    S_BURN["Burn 3pct"]
    S_CHARITY["Charity 2pct"]
    S_MARKETING["Marketing 3pct"]
    S_DEV["Dev Tip 4pct"]
  end

  TX["Transfer or Swap"] --> CHECK{"Buy or Sell?"}
  CHECK -- Buy --> Buy
  CHECK -- Sell --> Sell
```

---

## Transfer Flow with Fee Processing

```mermaid
sequenceDiagram
  participant Sender as Sender
  participant Contract as BlackForest Contract
  participant Holder as Other Holders
  participant Burn as Burn Address
  participant Wallet as Fee Wallets

  Sender->>Contract: transfer(recipient, amount)
  Contract->>Contract: _beforeTokenTransfer<br/>check maxTransactionAmount
  Contract->>Contract: calculate fees (buy or sell)

  par Redistribution
    Contract->>Holder: distribute proportional to balance (Rfi)
  and Burn
    Contract->>Burn: send burn fee to burn address
  and Liquidity
    Contract->>Contract: accumulate liquidity fee in contract
    alt contract balance >= numberOfTokensToSwapToLiquidity
      Contract->>Contract: swapAndLiquify
      Note over Contract: swap half for BNB<br/>create LP token with other half
    end
  and Charity + Marketing + Dev
    Contract->>Wallet: send to dedicated wallet addresses
  end

  Contract->>Contract: _reflectFee — adjust reflected balances
  Contract->>Contract: update _tFeeTotal for redistribution
  Contract->>Sender: transfer remaining to recipient
```

---

## Redistribution Mechanism (Rfi)

The redistribution (Rfi) system rewards holders proportionally to their token balance. It uses a dual-balance system: `_rOwned` (reflected balances used for fee math) and `_tOwned` (true token balances for excluded accounts).

```mermaid
flowchart TD
  TX["Each transfer deducts Rfi fee"] --> REFLECT["_reflectFee"]
  REFLECT --> RATE["Calculate fee rate from rFee and tFee"]
  RATE --> ADJUST["_rOwned -= rFee for sender<br/>_rOwned += rFee for holders (proportional)"]
  ADJUST --> SUPPLY["_tFeeTotal += tFee<br/>_reflectedSupply -= rFee"]
  SUPPLY --> REWARD["All non-excluded holders<br/>gain proportional share<br/>automatically in their balance"]

  note right of REWARD
    Excluded accounts (contract, burn,
    charity, marketing, dev) use _tOwned
    and do NOT receive redistribution
  end note
```

---

## Swap and Liquify Mechanism

```mermaid
flowchart TD
  ACCUMULATE["Liquidity fee accumulates<br/>in contract balance"] --> THRESHOLD{"Contract balance >=<br/>numberOfTokensToSwapToLiquidity?"}
  THRESHOLD -- no --> ACCUMULATE
  THRESHOLD -- yes --> SPLIT["Split accumulated tokens<br/>into two halves"]
  SPLIT --> SWAP["Swap half for BNB<br/>via PancakeSwap router"]
  SWAP --> LP["Add other half + BNB<br/>to liquidity pool"]
  LP --> TOKEN_LP["Mint LP tokens<br/>sent to contract owner"]
  TOKEN_LP --> RESUME["Resume normal transfers<br/>accumulate next batch"]
```

**Threshold:** `numberOfTokensToSwapToLiquidity` = 0.1% of total supply. Once the contract's token balance reaches this threshold, the next transfer triggers swap-and-liquify.

---

## Safety Limits

```mermaid
flowchart LR
  TX["Transfer"] --> MAX_TX{"amount <=<br/>maxTransactionAmount?<br/>(1pct of supply)"}
  MAX_TX -- no --> REJECT_TX["Revert"]
  MAX_TX -- yes --> MAX_WALLET{"recipient balance <=<br/>maxWalletBalance?<br/>(2pct of supply)"}
  MAX_WALLET -- no --> REJECT_WALLET["Revert"]
  MAX_WALLET -- yes --> PROCEED["Proceed with transfer + fees"]
```

---

## Deployment

```mermaid
flowchart LR
  REMIX["Remix IDE"] --> COMPILE["Compile BlackForest.sol<br/>compiler 0.8.4<br/>enable optimization"]
  COMPILE --> DEPLOY["Deploy to BSC mainnet<br/>select BlackForest contract"]
  DEPLOY --> VERIFY["Verify on BscScan"]
  VERIFY --> TRADE["Trade on PancakeSwap<br/>max slippage 49.9pct<br/>fees over 35pct may fail"]
```
