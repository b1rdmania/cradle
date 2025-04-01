# Cradle.build - V1 Technical Specification

**Document Version:** 1.2
**Date:** 2025-04-01

## 🧱 Cradle.build — Sonic-Native Token Launch Infrastructure

Cradle is envisioned as a permissionless, tokenless launchpad built for clean, fixed-price token raises on the Sonic network. It provides smart contracts, tooling, and frontend components to launch transparent public token sales—integrated with Hedgey Finance for post-sale vesting.

## 🔒 Guiding Principles

Cradle aims to be a neutral infrastructure provider. It does not hold user funds (post-sweep), offer investment advice, or issue its own platform token. Responsibility for project quality and outcomes rests with the project teams using the platform.

## 🚀 V1 MVP Overview

**Goal:** Enable teams to launch Sonic-native token sales via a factory contract with the following core features:

* Fixed-price ERC20 contributions (e.g., USDC).
* Optional whitelist-based presale phase using a Merkle root.
* Standard public raise phase following the presale (or as the only phase).
* Configurable fee routing (e.g., 5% suggested default) applied during fund withdrawal by the project.
* Enforceable Minimum and Maximum contribution limits per wallet (denominated in the token being sold).
* Owner-controlled ability to cancel the sale *before* it starts.
* Deployment of individual sale contracts via a central `CradleFactory`.
* Exportable final contribution allocations for post-raise vesting setup via Hedgey Finance.

## 🧱 Smart Contracts (V1)

### 1. `CradleRaise.sol`

* **Description:** Immutable smart contract governing a single token sale instance. Stores sale configuration, manages contribution phases, tracks contributions, enforces hard caps and per-wallet limits, and facilitates fund withdrawal with fee application. Includes a pre-start cancellation mechanism.
* **Constructor Parameters:**
    * `address token`: The ERC20 token being sold.
    * `address acceptedToken`: The ERC20 token used for payment (e.g., USDC on Sonic).
    * `uint256 pricePerToken`: Price defined as the amount of `acceptedToken` base units required per 1 *whole* `token` (e.g., if 1 USDC buys 1 TOKEN (18 decimals), `pricePerToken` = `1 * 10^6`). Must account for decimals precisely.
    * `uint256 presaleStart`: Unix timestamp for presale start.
    * `uint256 publicSaleStart`: Unix timestamp for public sale start (must be >= presaleStart).
    * `uint256 endTime`: Unix timestamp for sale end (must be >= publicSaleStart).
    * `bytes32 merkleRoot`: Merkle root for presale whitelist (provide `bytes32(0)` if no presale).
    * `address owner`: The project's wallet address receiving net funds after fee. Set as the contract owner.
    * `address feeRecipient`: Address receiving the platform fee.
    * `uint16 feePercentBasisPoints`: Fee percentage in basis points (e.g., 500 = 5.00%). Max 10000.
    * `uint256 maxAcceptedTokenRaise`: The hard cap for the sale, denominated in `acceptedToken` base units.
    * `uint256 minTokenAllocation`: Minimum amount of `token` (in base units) allowed per contribution transaction.
    * `uint256 maxTokenAllocation`: Maximum total amount of `token` (in base units) allowed per contributor wallet.
* **Key Functions:**
    * `contribute(uint256 _tokenAmountToBuy, bytes32[] calldata _proof)`: Allows users to contribute; verifies phase, time, whitelist (if applicable), min/max limits, and hard cap. Requires prior `acceptedToken` approval.
    * `finalizeRaise()`: Owner callable after `endTime` to formally mark the sale as finalized (if not cancelled).
    * `sweep()`: Owner callable after finalization; transfers `acceptedToken` funds to `owner` and `feeRecipient`.
    * `getContribution(address _account)`: View function returning the total amount of `token` purchased by an account.
    * `cancelSale()`: Owner callable *only before* `presaleStart` to irreversibly cancel the sale.
* **Security:** Inherits `Ownable`, `ReentrancyGuard`. Uses `SafeERC20`. Implements Merkle proof verification. State locked after finalization or cancellation.

*(Optional: Insert CradleRaise.sol skeleton code here)*

### 2. `CradleFactory.sol`

* **Purpose:** Serves as a deployer for `CradleRaise.sol` instances and maintains an on-chain registry of all sales created through it.
* **Permissions (V1):** Deployment via `createRaise` is restricted to the factory owner initially. Ownership is transferable.
* **Key Functions:**
    * `createRaise(address token, address acceptedToken, ..., uint256 maxTokenAllocation)`: Takes all `CradleRaise` constructor parameters, deploys a new `CradleRaise` instance using `create`, stores the new instance's address in the registry, and emits a `RaiseCreated` event.
    * `getDeployedRaises()`: View function returning an array of all deployed `CradleRaise` contract addresses.
    * (Standard `Ownable` functions: `owner()`, `transferOwnership(...)`).
* **Registry:** Implemented as a dynamic array (`address[] public deployedRaises`).

*(Optional: Insert basic CradleFactory.sol skeleton code here)*

## 🖼 Raise Flow (V1)

1.  Project owner uses the Cradle UI Admin Panel to input all sale parameters.
2.  Frontend generates the Merkle root hash from the provided whitelist CSV (if applicable).
3.  Project owner's wallet calls `createRaise` on the `CradleFactory` contract via the UI, passing all parameters including the Merkle root. The Factory deploys a new `CradleRaise` instance and records its address.
4.  Users discover the sale via the Cradle UI (using the Factory registry + metadata).
5.  Users contribute to the specific `CradleRaise` instance during the presale (providing proof) or public phase, respecting the `minTokenAllocation` and `maxTokenAllocation` limits. Users must approve the `CradleRaise` contract to spend their `acceptedToken`.
6.  `endTime` is reached.
7.  Project owner calls `finalizeRaise()` on the `CradleRaise` instance (unless cancelled).
8.  Project owner calls `sweep()` on the `CradleRaise` instance to withdraw the collected `acceptedToken` (fees are routed automatically).
9.  Cradle system (or project owner) exports the final contribution data (`address` -> `tokenAmount`) from the `CradleRaise` instance.
10. Exported data is used to set up vesting schedules via Hedgey Finance's platform/API.
11. Cradle UI provides a link on the Raise Page directing users to the corresponding Hedgey claim portal.

## 🧩 Hedgey Integration (Confirmed)

Cradle **does not handle vesting on-chain** for V1. Vesting logic (TGE%, cliff, linear/custom schedules) is managed entirely by Hedgey Finance. Cradle's role is to conduct the initial sale and provide the final allocation data required by Hedgey. The Cradle frontend will link to the Hedgey claim interface.

## 🎨 Frontend Structure (V1)

1.  **Admin Panel:**
    * Multi-step form to collect all parameters needed for `CradleFactory.createRaise`.
    * Includes CSV upload and client-side Merkle root generation for the whitelist.
    * Facilitates the transaction call to `CradleFactory.createRaise` via connected wallet.
2.  **Raise Page (for each `CradleRaise` instance):**
    * Displays sale information (token details, project info from metadata store, sale times, caps, price, current total raised).
    * Shows countdown timers for different phases.
    * Includes contribution section (input amount, connects wallet, handles `acceptedToken` approval, calls `contribute`).
    * Displays user-specific info (connected wallet's contribution amount, applicable min/max limits).
    * Shows post-contribution status feedback.
    * Provides a "Claim on Hedgey" button/link after the sale (directing to the appropriate Hedgey portal).
3.  **Metadata Store Integration:**
    * Project name, description, logo URL, banner URL, social links associated with each deployed `CradleRaise` address.
    * Stored off-chain (e.g., Supabase/Postgres).
    * Data fetched and displayed on Discovery and Raise pages.
4.  **Discovery Page:**
    * Lists active and potentially past raises created via the `CradleFactory`.
    * Reads the list of addresses from `CradleFactory.getDeployedRaises()`.
    * Fetches corresponding metadata for each raise to display summary cards/tiles.

## 🛠 Tech Stack

* **Blockchain:** Sonic (Testnet & Mainnet)
* **Smart Contracts:** Solidity ^0.8.20+, OpenZeppelin Contracts v4.x/v5.x, Hardhat or Foundry
* **Frontend:** React (or similar modern framework), Wagmi / Ethers.js, TailwindCSS (or similar UI library)
* **Backend/DB:** Supabase / Postgres (for Metadata Store)
* **Vesting Partner:** Hedgey Finance

## 📤 Deliverables (V1)

* Audited and tested `CradleRaise.sol` smart contract (fixed-price, min/max, cancel features).
* Audited and tested `CradleFactory.sol` smart contract (deployer, registry, owner-controlled).
* Client-side Merkle tree generation script/library (JS).
* Frontend components: Admin Panel (Sale Creation via Factory), Raise Page (Participation), Discovery Page (Listing).
* Backend Metadata store schema definition and basic setup/API if needed.
* Mechanism/script for exporting contribution data post-sale for Hedgey.
* Frontend integration hook/link for Hedgey claim page.
* Comprehensive Test Suite (Unit, Integration, Fork tests highly recommended for contracts).
* Deployment scripts (for Factory contract primarily).
* Basic Documentation (README covering setup, deployment, usage; NatSpec comments in contracts).

## 🔒 Legal Model

Raise contracts deployed via the Factory are permissionless in their operation between the project (`owner`) and contributors. Cradle UI may initially display curated/approved raises but users can interact with any deployed contract directly via block explorers. Cradle (the platform entity) does not issue tokens, custody contributor funds (except transiently in `CradleRaise` before `sweep`), or guarantee project outcomes.

## 🧪 Audit Plan

* Thorough internal testing (unit, integration, potentially fuzzing) is required.
* **Mandatory:** A professional third-party security audit covering both `CradleRaise.sol` and `CradleFactory.sol` is required before mainnet deployment.
* **Suggested External Audit Scope:**
    * `CradleRaise`: Allocation & cap logic (hard cap, min/max per wallet), Fee routing, Merkle validation, Finalization/sweep security, `cancelSale` logic, State management across phases, Access control, Reentrancy protection, Decimal handling.
    * `CradleFactory`: Deployment logic (ensuring correct parameters passed, secure deployment), Registry integrity (correctly storing/retrieving addresses), Access control/Ownership, Event emission.

## 🧭 Future Add-Ons (Out of Scope for V1)

* Balancer LBP integration (Dynamic pricing).
* Raffle/Lottery allocation systems.
* Overflow sale models (Pro-rata allocations).
* More complex phase structures (discounted rounds).
* Claimable NFT badges for participation/vesting.
* Permissionless factory deployment.
* Pause/Unpause functionality.
* Soft caps / Refunds.

## 🤝 Credits & Partners

* **Vesting Partner:** Hedgey Finance (Confirmed)
* **Network:** Sonic L1
* **Contract Base:** Inspired by OpenZeppelin standards.

## 📬 Contact

* **Questions or Contributions:** team@cradle.build
* **Twitter:** @CradleBuild

*(Project team contact details can be added here for development proposals)*
