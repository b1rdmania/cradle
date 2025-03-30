# 🧱 Cradle.build — Sonic-Native Token Launch Infrastructure

Cradle is a permissionless, tokenless launchpad built for clean, fixed-price token raises on the Sonic network. It provides smart contracts, tooling, and frontend components to launch transparent public token sales—integrated with Hedgey for post-sale vesting.

> 🔒 Cradle does not hold funds, offer investment advice, or issue a platform token.

---

## 🚀 MVP Overview

**Goal:**  
Enable teams to launch Sonic-native token sales with:
- Fixed-price ERC20 contributions (e.g., USDC)
- Optional whitelist-based presale
- Public raise phase
- 5% fee routing
- Exportable allocations for post-raise vesting (via Hedgey)

---

## 🧱 Smart Contracts

### `CradleRaise.sol`
Immutable raise contract for a single sale.  
Stores configuration, contributions, and enforces caps.

**Constructor Parameters:**
- `address token`
- `uint256 pricePerToken`
- `uint256 presaleStart`
- `uint256 publicSaleStart`
- `uint256 endTime`
- `address acceptedToken` (e.g., USDC)
- `bytes32 merkleRoot`
- `address owner`
- `address feeRecipient`
- `uint256 feePercent`

**Key Functions:**
- `contribute(uint256 amount, bytes32[] proof)`
- `finalizeRaise()`
- `sweep()` (withdraws funds to project, applies fee)
- `getContribution(address)` (view total for user)

**Security:**
- `ReentrancyGuard`, `Ownable`, `SafeERC20`
- Merkle-based whitelist validation (pre-sale only)
- Finalization locks state

---

## 🖼 Raise Flow

1. Project sets up raise via UI  
2. Frontend generates Merkle root from CSV whitelist  
3. CradleRaise is deployed (immutable config)  
4. Users contribute in presale (if whitelisted) or public phase  
5. Raise finalizes → project withdraws funds  
6. Contributions exported → tokens vested via Hedgey  
7. Cradle UI links to Hedgey claim portal

---

## 🧩 Hedgey Integration (Confirmed)

Cradle does not handle vesting on-chain.  
Instead, raise allocations are exported after finalization to Hedgey, which:

- Receives wallet + allocation data
- Handles vesting logic (TGE %, cliff, linear)
- Offers claim portal per raise

Cradle UI links to the Hedgey claim page post-raise.

---

## 🎨 Frontend Structure

### 1. Admin Panel
- Raise setup form
- Upload whitelist CSV → generates Merkle root
- Deploy raise via wallet

### 2. Raise Page
- Token + project info
- Countdown to presale/public phase
- Contribution form
- Post-contribution status
- Claim button (linked to Hedgey)

### 3. Metadata Store
- Project name, logo, description, socials
- Stored off-chain in Postgres/Firebase/Supabase
- Used to populate frontend tiles + search

---

## 🛠 Tech Stack

| Layer | Stack |
|-------|-------|
| Contracts | Solidity 0.8+, OpenZeppelin, Hardhat |
| Frontend | React, Tailwind, Wagmi, Ethers.js |
| Backend | Supabase/Postgres, optional Node API |
| Deployment | Sonic testnet/mainnet |

---

## 📤 Deliverables

- [x] `CradleRaise.sol` (ERC20 fixed-price raise)
- [x] Merkle tree generator (JS)
- [x] Raise creation + participation UIs
- [x] Metadata store schema
- [x] Hedgey export + integration hook
- [x] Basic QA & unit tests
- [ ] (Optional) RaiseFactory.sol for managing multiple raises
- [ ] (Optional) Raise Registry for frontend discovery

---

## 🔒 Legal Model

- Raise contracts are **permissionless**
- Cradle UI **only displays curated raises** (approved by Sonic team)
- Users may interact with any raise contract directly
- Cradle does **not issue tokens, custody funds, or guarantee outcomes**

---

## 🧪 Audit Plan

- Internal test + fuzz coverage (v0.1)
- Suggested external audit scope:
  - Allocation & cap logic
  - Fee routing
  - Merkle validation
  - Finalization/sweep security

---

## 🧭 Future Add-Ons

| Feature | Description |
|--------|-------------|
| Balancer LBP | Dynamic price discovery via weight-shifting pools |
| Axis Finance Raffle | Lottery-based allocation system |
| Overflow Model | Accept all contributions → pro-rata token allocation |
| Phase-Based Rounds | Discounted early round → public raise |
| Claim NFTs | Vesting “badges” via ERC-721 |

---

## 🤝 Credits & Partners

- **Vesting Partner:** [Hedgey Finance](https://hedgey.finance) (Confirmed)
- **Network:** Sonic L1
- **Contract Base:** Inspired by OpenZeppelin + FairLaunch models

---

## 📬 Contact

Questions or contributions?  
Email: team@cradle.build  
Twitter: [@CradleBuild](https://twitter.com/CradleBuild)  
