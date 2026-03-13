# OP_NEXUS Security Analysis

**Security Documentation v1.0**

```
Classification:  Public
Scope:           OPNexus.ts — Single contract deployment
Network:         Bitcoin L1 via OP_NET
Contract:        opt1sqr8y4gcjm3syum7awmu68pylqlynmxah3qeuu4m7
Compiled:        OPNexus.wasm (47,806 bytes)
Date:            2025
```

---

> *"Security in a unified system is not the sum of the security of its parts.
> It is a property of their interaction. Every cross-module flow is a potential
> attack surface. Every invariant that holds across modules is a defense."*

---

## Table of Contents

1. [Security Philosophy](#1-security-philosophy)
2. [Threat Model](#2-threat-model)
3. [Attack Surface Analysis](#3-attack-surface-analysis)
4. [Invariants and Formal Properties](#4-invariants-and-formal-properties)
5. [Reentrancy Analysis](#5-reentrancy-analysis)
6. [Oracle Manipulation Vectors](#6-oracle-manipulation-vectors)
7. [Arithmetic Security](#7-arithmetic-security)
8. [Flash Loan Attack Scenarios](#8-flash-loan-attack-scenarios)
9. [Liquidation Attack Vectors](#9-liquidation-attack-vectors)
10. [Governance Attack Vectors](#10-governance-attack-vectors)
11. [Cross-Module Attack Scenarios](#11-cross-module-attack-scenarios)
12. [Economic Attack Analysis](#12-economic-attack-analysis)
13. [OP_NET Runtime Security](#13-op_net-runtime-security)
14. [Known Limitations](#14-known-limitations)
15. [Security Checklist](#15-security-checklist)
16. [Responsible Disclosure Policy](#16-responsible-disclosure-policy)
17. [Audit Scope](#17-audit-scope)

---

## 1. Security Philosophy

### 1.1 The Unified Contract Tradeoff

OP_NEXUS's core architectural decision — thirteen financial primitives in a single contract — has profound security implications in both directions.

**Advantages of unification:**

The most dangerous class of DeFi exploits are **cross-contract state manipulation attacks**: exploits that use flash loans to distort the state of Contract A, then call Contract B before A's state is restored, profiting from the temporary inconsistency. These attacks are structurally impossible in OP_NEXUS. There is no Contract B. All state changes happen within a single execution context where the runtime enforces atomicity. A transaction either completes with all state changes applied, or reverts with none.

The absence of cross-contract calls eliminates an entire family of attack vectors that have drained hundreds of millions of dollars from fragmented DeFi protocols.

**Risks of unification:**

A bug in any single module affects the entire contract. There is no blast-radius containment between modules. A critical arithmetic error in the lending module does not affect only the lending module — it affects the vault, which affects the DEX, which affects flash loans. This demands a higher standard of correctness per module than would be required in an isolated protocol.

**Our response:** Every module is analyzed in isolation first, then in the context of all possible cross-module interactions. This document presents that analysis.

### 1.2 Defense Layers

OP_NEXUS implements security in four layers:

```
Layer 4 — Economic Security
         Incentive structures that make attacks unprofitable

Layer 3 — Protocol Logic Security
         Invariants maintained by the contract logic itself

Layer 2 — Arithmetic Security
         SafeMath preventing overflow/underflow in all calculations

Layer 1 — Runtime Security
         OP_NET WASM execution environment determinism and isolation
```

A successful attack must defeat all four layers simultaneously.

### 1.3 Immutability Commitment

OP_NEXUS v1 has no admin keys, no owner, no upgrade function, no pause mechanism. The contract is immutable at deployment. This eliminates the following attack vectors present in most DeFi protocols:

- Admin key compromise leading to parameter manipulation
- Malicious upgrades replacing contract logic
- Emergency pause used as market manipulation tool
- Governance-controlled parameter changes as attack vectors

The immutability tradeoff is that bugs cannot be patched post-deployment. This makes pre-deployment security analysis more critical, not less.

---

## 2. Threat Model

### 2.1 Adversary Classes

**Class A — Passive Observer.** An adversary who reads on-chain state and reacts to it. Threat: front-running liquidations, copying flash loan strategies. Mitigation: none needed; these are legitimate economic behaviors that improve market efficiency.

**Class B — Single Transaction Adversary.** An adversary who can submit a single malicious transaction with arbitrary calldata. Threat: any single-call exploit. Mitigation: all invariants enforced within single transaction execution.

**Class C — Multi-Transaction Adversary.** An adversary who can submit a sequence of transactions, accumulating state across blocks. Threat: position building attacks, governance capture over multiple epochs. Mitigation: economic cost of large positions; governance requires sustained BTC lock.

**Class D — Oracle Adversary.** An adversary who controls one or more oracle nodes. Threat: price feed manipulation to trigger favorable liquidations. Mitigation: EWMA dampening, slash mechanism, multiple nodes.

**Class E — Majority Oracle Adversary.** An adversary who controls more than 50% of oracle stake. Threat: sustained price manipulation. This breaks the oracle's security assumption. Mitigation: permissionless node addition, economic cost of controlling >50% of staked BTC.

**Class F — Miner/Sequencer Adversary.** An adversary who controls block production (Bitcoin miner or OP_NET sequencer). Threat: transaction ordering manipulation, block withholding. Mitigation: atomic execution within blocks; most strategies are ordering-independent.

### 2.2 Assets at Risk

The primary asset at risk is user capital deposited in the system:

```
Total Value at Risk = vault_assets
                    + dex_reserves (all three pools)
                    + options_collateral_in_dex
                    + bond_collateral_in_lending
                    + frac_collateral_in_lending
                    + governance_locked_btc
```

Secondary assets at risk: oracle node stake (via slash), liquidation rewards (via timing attacks).

### 2.3 Attack Profitability Threshold

An attack is economically rational only if:

```
expected_profit > gas_cost + capital_cost + probability_of_failure × capital_at_risk
```

For OP_NET, gas costs are denominated in satoshis and are non-trivial for complex operations. This creates a minimum scale threshold below which attacks are unprofitable regardless of technical feasibility.

---

## 3. Attack Surface Analysis

### 3.1 Entry Points

The contract exposes ~70 public methods through the `execute()` dispatcher. Each method is an entry point for adversarial input. The attack surface is bounded by the set of valid method selectors — invalid selectors revert immediately.

```
execute(method: u32, calldata: Calldata): BytesWriter

if method ∉ VALID_SELECTORS:
    throw Revert('NEXUS: unknown method')
```

The 32-bit selector space has 2³² = 4.29 billion possible values. Approximately 70 are valid. Random selector collisions are negligible.

### 3.2 Calldata Attack Surface

Each method reads typed values from calldata. Malformed calldata (truncated, oversized, type-mismatched) does not produce undefined behavior — the OP_NET runtime's typed `Calldata` reader is bounds-checked. Reading beyond available calldata reverts.

**Integer boundary values.** Every method accepting a u256 value can receive:
- Zero: explicitly checked where zero would be invalid (`if amount.isZero() throw Revert`)
- u256.MAX: checked via SafeMath overflow protection on all arithmetic
- Carefully chosen values that satisfy individual checks but create issues in combination

The latter class — **crafted combinatorial inputs** — is the primary concern and is analyzed per-module below.

### 3.3 State Transition Graph

The total state space of OP_NEXUS is the Cartesian product of all stored variable ranges. Not all states are reachable from valid initial states through valid transactions. The security analysis focuses on ensuring that no valid transaction sequence can reach a state where:

1. A user can withdraw more than they deposited (plus legitimate yield)
2. Protocol debt exceeds protocol assets
3. A module's accounting becomes inconsistent with the DEX's actual reserve state

---

## 4. Invariants and Formal Properties

### 4.1 System-Level Invariants

The following invariants must hold after every transaction. Violation of any invariant constitutes a critical vulnerability.

**INV-1: DEX Reserve Non-Negativity**
```
_dexR0p1 ≥ _dexOptCollateral + _dexPerpMargin + _dexFlashOut
_dexR0p2 ≥ 0
_dexR0p3 ≥ 0
```
*Enforced by:* `dexRemove()` checks free reserve before withdrawal; `flashBorrow()` checks free reserve before lending; `dexSwap()` output bounded by reserves.

**INV-2: Flash Loan Atomicity**
```
if _dexFlashActive = true:
    _dexFlashOut > 0
    no new flash borrow is possible
```
*Enforced by:* `_dexFlashActive` mutex checked at start of `flashBorrow()`, `routerFlashLiq()`, `routerFlashOpt()`.

**INV-3: Lending Solvency**
```
_lndTotalSupply ≥ _lndTotalBorrow   (at all times after accrual)
```
*Enforced by:* `lendBorrow()` checks `borrow ≤ available = supply − borrow`; liquidation reduces borrow and increases supply (via DEX recycling).

**INV-4: Vault Share Fairness**
```
assets_returned_on_withdraw = shares × _vltAssets / _vltShares
```
*Enforced by:* withdrawal formula derived from ERC-4626 standard; no rounding in depositor's disfavor (integer division may produce slight truncation, always ≤ 1 satoshi).

**INV-5: Bond Overcollateralization**
```
_bndCollatInLend ≥ _bndTotalDebt × 15000/10000
```
*Enforced by:* `bondIssue()` checks `collateral ≥ face_value × 1.5` before creating bond.

**INV-6: Oracle Price Positivity**
```
_priceBTC > 0
_priceMOTO > 0
_pricePILL > 0
```
*Enforced by:* `oracleSubmit()` checks `price.isZero() → Revert`; EWMA update is a weighted average of positive values, preserving positivity.

**INV-7: Fee Distribution Completeness**
```
to_vault + to_vebtc + to_quad + to_treasury = fee (exactly)
```
*Enforced by:* treasury share computed as `fee − to_vault − to_vebtc − to_quad` (residual), ensuring no rounding loss.

**INV-8: vePower Bound**
```
vePower_individual ≤ amount_locked
```
*Enforced by:* `govLock()` computes `power = amount × duration / max_duration ≤ amount`.

### 4.2 Module-Level Invariants

**Lending:**
```
health_factor(position) ≥ 1.0 → position not liquidatable
health_factor(position) < 1.0 → position immediately liquidatable by anyone
```

**Perps:**
```
_dexPerpMargin ≤ _dexR0p1    (margin never exceeds pool)
insurance_fund ≥ 0           (never negative)
```

**Options:**
```
_dexOptCollateral ≤ _dexR0p1     (collateral never exceeds pool)
_optCollatInDex = _dexOptCollateral  (accounting mirrors DEX state)
```

**QuadFund:**
```
_qdfMatchingPool = cumulative_fee_contributions − cumulative_distributions
```

---

## 5. Reentrancy Analysis

### 5.1 Why Traditional Reentrancy Is Impossible

Classical reentrancy attacks require calling back into a contract from an external call before the calling contract has updated its state. This requires:
1. An external call that transfers control to adversary code
2. The adversary code calling back into the vulnerable contract
3. The vulnerable contract's state being in an inconsistent (mid-update) condition

In OP_NET's execution model, contracts do not make external calls in the traditional sense. A single transaction executes deterministically from start to finish without yielding control to external code. There are no `call()` opcodes that could invoke adversary-controlled callbacks.

**This eliminates classical reentrancy attacks entirely.**

### 5.2 Intra-Contract Reentrancy

Within OP_NEXUS, the Router executes multi-step operations that call internal helper functions in sequence. A possible concern: could a sequence of internal calls create a state where an invariant is temporarily violated?

**Analysis:** Internal calls within a single method execution are sequential, not concurrent. State is written to storage immediately on assignment (`StoredU256.value = x`). Subsequent reads reflect the updated state. There is no window of inconsistency between an internal write and an internal read.

**Example:** `routerFlashLiq()` executes five sequential state changes:
```
(1) _dexFlashActive := true         [mutex set]
(2) Pool1.R0 -= flash_amt           [pool decremented]
(3) assert HF < 1.0                 [check uses current state]
(4) Pool1.R0 += seized_collateral   [pool incremented]
(5) D_total -= debt                 [debt cleared]
(6) Pool1.R0 += repayment           [flash repaid]
(7) _dexFlashActive := false        [mutex released]
```

At step (3), Pool1.R0 reflects the flash loan deduction. The health factor check uses oracle price and collateral/debt values, none of which have been modified yet. The check is valid.

At step (4), Pool1.R0 increases. This is correct — seized collateral flows in.

At no point between (1) and (7) can another flash loan begin (mutex). At no point can a swap extract more than the available free reserve (flash_out is tracked in `_dexFlashOut`).

### 5.3 The Flash Loan Mutex

The `_dexFlashActive` boolean is the critical reentrancy guard for flash operations. Its security properties:

```
BEFORE flash: _dexFlashActive = false (default)
DURING flash: _dexFlashActive = true
              → any call to flashBorrow() reverts
              → any call to routerFlashLiq() reverts
              → any call to routerFlashOpt() reverts
AFTER repay:  _dexFlashActive = false
```

**Critical invariant:** `_dexFlashActive` is set to `false` only inside `flashRepay()` and at the end of the router flash methods. It cannot be set to `false` by any other code path. If `flashBorrow()` is called and then the transaction reverts before repayment, the entire transaction reverts — including the setting of `_dexFlashActive = true`. The storage state is rolled back to pre-transaction state.

This is enforced by OP_NET's atomic transaction model: if any operation in a transaction throws a `Revert`, all state changes in that transaction are discarded.

---

## 6. Oracle Manipulation Vectors

### 6.1 Single-Node Price Push Attack

**Scenario:** An oracle node controls 1 of N nodes and submits a false price P_false = P_true × (1 + δ) to trigger advantageous liquidations.

**EWMA dampening analysis:**

After one false submission:
```
P_ewma_new = (7/8) × P_ewma_old + (1/8) × P_false
           = P_true + δ × P_true / 8
           = P_true × (1 + δ/8)
```

The oracle moves by δ/8 — at most 12.5% of the intended manipulation.

**To trigger a liquidation** of a position with health factor HF = 1.1 (10% buffer):
```
Required price drop: (HF − 1.0) / HF × P_true = 9.09%
Price drop achievable with one submission: δ/8

For attack to succeed in one block: δ > 72.7%
```

A single false submission must be 72.7% below the true price to immediately trigger liquidation on a position with 10% buffer. This is detectable and slashable.

**Multi-submission attack cost:**

For n consecutive false submissions at δ% below true price:
```
P_ewma(n) = P_true × (1 − (1 − (7/8)^n) × δ)
```

To move the oracle 10% with δ = 20% per submission:
```
(1 − (7/8)^n) × 0.20 = 0.10
(7/8)^n = 0.50
n = ln(0.5) / ln(0.875) ≈ 5.2 submissions
```

Requires ~6 consecutive false submissions, each incurring:
```
slash_cost = stake × 500/10000 = 5% per false submission detected
total_slash = 6 × 5% × stake = 30% of stake
```

For the attack to be profitable:
```
liquidation_profit > 30% × stake

stake < liquidation_profit / 0.30
```

The oracle is economically secure when stake > 3.33× the maximum liquidation profit achievable from a given position.

### 6.2 Stale Price Attack

**Scenario:** Oracle nodes stop submitting prices. The EWMA price stales. An attacker exploits the divergence between the stale oracle price and the true market price.

**Impact analysis:** Stale prices create two-sided risk:
- If oracle > true: lending positions appear healthier than they are (under-liquidation risk)
- If oracle < true: positions appear unhealthier than they are (over-liquidation risk)

**Mitigation:** `_orcLastBlock` tracks the last update block. Client-side interfaces display staleness warnings. Future protocol versions will add an on-chain freshness check: if `block.number − _orcLastBlock > STALENESS_THRESHOLD`, mark the oracle as degraded and prevent new borrows/perp opens until prices are updated.

**Note:** Price staleness does not create an immediate exploit — it creates market inefficiency that arbitrageurs (not attackers) correct by updating the oracle.

### 6.3 Coordinated Multi-Node Attack

**Scenario:** An adversary controls >50% of oracle stake and coordinates sustained false submissions.

**Formal security bound:** Let S_total be total oracle stake and S_adversary the adversary's share. The attack cost for n submissions:

```
cost = n × S_adversary × τ_slash = n × S_adversary × 0.05
```

Since slashing only occurs when the submission is detected as outlier (>5% from consensus), and the adversary controls >50% of stake, they can potentially define the "consensus" — meaning their submissions are not outliers relative to each other.

**This is the fundamental oracle security assumption:** OP_NEXUS's oracle is secure against adversaries controlling <50% of total oracle stake. This is an honest majority assumption, identical to Bitcoin's security model.

**Mitigation for Phase 1:** Minimum stake requirements ensure that controlling >50% requires acquiring >50% of total staked BTC, which is economically prohibitive at scale. The oracle's security budget grows with the protocol's TVL.

---

## 7. Arithmetic Security

### 7.1 SafeMath Coverage

All arithmetic operations in OP_NEXUS use `SafeMath` from `@btc-vision/btc-runtime/runtime`. SafeMath provides:

```
SafeMath.add(a, b):   throws if result overflows u256
SafeMath.sub(a, b):   throws if b > a (underflow)
SafeMath.mul(a, b):   throws if result overflows u256
SafeMath.div(a, b):   throws if b = 0
```

Every arithmetic operation in the contract uses one of these four functions. No raw arithmetic operators (`+`, `-`, `*`, `/`) are used on u256 values.

### 7.2 Integer Division Truncation

Solidity/AssemblyScript integer division truncates toward zero. This creates small rounding errors in favor of or against users.

**Vault share issuance:**
```
shares = amount × S / A
```
Truncation means slightly fewer shares are issued. This rounds in favor of existing shareholders (their position is not diluted by rounding up new shares). Fairness property: existing shareholders never lose value from new deposits.

**Fee distribution:**
```
to_vault    = fee × 40/100
to_vebtc    = fee × 30/100
to_quad     = fee × 20/100
to_treasury = fee − to_vault − to_vebtc − to_quad    (residual)
```

By computing treasury as the residual, rounding errors accumulate into treasury rather than creating fee leakage. The sum always equals the original fee exactly.

**Liquidation bonus:**
```
bonus = D × 500/10000
```

For D < 20 satoshis, `bonus = 0` due to integer truncation. This is acceptable — liquidating positions of < 20 satoshis is economically irrational regardless.

### 7.3 Overflow Risk Assessment

**Maximum realistic values:**

```
Total BTC supply:          21,000,000 BTC = 2.1 × 10¹⁵ satoshis
Maximum u256:              2²⁵⁶ ≈ 1.16 × 10⁷⁷

Headroom factor:           1.16 × 10⁷⁷ / 2.1 × 10¹⁵ ≈ 5.5 × 10⁶¹
```

The headroom is 61 orders of magnitude. No realistic operation approaches overflow. The most aggressive intermediate calculation is in interest accrual:

```
D_total × APR_rate × elapsed = 2.1×10¹⁵ × 518 × 52560 ≈ 5.7 × 10²²
```

This is 54 orders of magnitude below u256.MAX. Safe.

**Multiplication-before-division pattern:**

Formulas like `amount × weight / 100` always multiply before dividing to preserve precision:
```
// CORRECT (precision preserved):
shares = amount × total_shares / total_assets

// WRONG (precision lost if amount < total_assets/total_shares):
shares = (amount / total_assets) × total_shares
```

OP_NEXUS consistently uses the multiply-first pattern.

### 7.4 Babylonian Square Root Security

The `_sqrt(n)` function implements the Babylonian method:

```
x := n
y := n/2 + 1
while y < x:
    x := y
    y := (n/x + x) / 2
return x
```

**Correctness:** This converges to ⌊√n⌋ for all n ≥ 0. For n = 0, returns 0 immediately.

**Termination:** The sequence x, y is strictly decreasing until convergence. For u256 values, convergence occurs in at most 128 iterations.

**Security concern:** Could an adversary craft an n value that causes non-termination? No — the Babylonian method converges monotonically for all positive integers. The while condition `y < x` guarantees termination when the sequence stabilizes.

---

## 8. Flash Loan Attack Scenarios

### 8.1 Classic Flash Loan Oracle Manipulation — MITIGATED

**Attack pattern in fragmented DeFi:**
```
1. Flash borrow large amount from Protocol A
2. Dump on Protocol B's AMM, crashing its price
3. Call Protocol C's liquidation function (which reads B's price)
4. Profit from cheap liquidation
5. Repay flash loan
```

**Why this fails in OP_NEXUS:**

The oracle is an EWMA of externally submitted prices, not derived from DEX reserves. A flash loan cannot move the oracle price in a single block. Any attack that depends on "flash-crashing" the oracle price to trigger liquidations fails because:

```
Oracle update per block = at most 1/8 of submitted price
Flash loan duration = 1 block
Maximum oracle movement from flash = 12.5%
```

Triggering a liquidation requires a health factor < 1.0. Positions with HF > 1.125 cannot be liquidated even if the oracle moves maximally in one block.

### 8.2 Flash Loan Pool Drain Attack — MITIGATED

**Attack attempt:**
```
1. Flash borrow entire free reserve of Pool 1
2. Add as liquidity to Pool 2 (temporarily huge reserves)
3. Swap to get favorable rate against distorted Pool 2
4. Remove liquidity from Pool 2
5. Repay Pool 1 flash
```

**Analysis:**

Step 2: Flash loans can only borrow from Pool 1 (`_dexFlashActive` protects Pool 1). Temporarily holding borrowed funds does not affect Pool 2 unless they are actually deposited.

If borrowed funds are added to Pool 2 as liquidity:
- `dexAdd(2, R0_borrowed, R1_provided)` would mint LP tokens
- The borrowed funds are now deployed as LP, not free capital
- To repay the flash loan, the attacker must remove the LP before the transaction ends
- LP removal gives back `(R0_borrowed, R1_provided)` proportionally — no gain
- Flash fee is still owed

The attack generates no profit because adding and immediately removing liquidity within a single transaction nets zero. The LP fee accumulation requires the position to persist across swaps, which requires multiple blocks.

### 8.3 Flash Loan + Fake Liquidation Attack — MITIGATED

**Attack attempt:**
```
1. Flash borrow X from Pool 1
2. Supply X to Lending as collateral (self-supply)
3. Borrow Y against collateral (self-borrow)
4. "Liquidate" own position for the 5% bonus
5. Repay flash loan
```

**Analysis:** Self-liquidation requires HF < 1.0. To achieve HF < 1.0 on a freshly-opened position:

```
C = X (collateral, just deposited)
D = X × 0.75 (max borrow at 75% LTV)

HF = C × P_btc × 0.80 / D = X × 0.80 / (X × 0.75) = 1.067

HF = 1.067 > 1.0  →  not liquidatable
```

A position at maximum LTV has HF = 1.067. It cannot be liquidated. The attacker gains nothing from the flash loan.

Furthermore, the liquidation bonus is paid from the seized collateral (which is the attacker's own deposited X). The "profit" is:

```
liquidation_reward = D + D × 0.05 = D × 1.05
collateral_seized = X

net = D × 1.05 − flash_fee − D         (repaid the borrow)
    = D × 0.05 − flash_fee
    = X × 0.75 × 0.05 − X × 0.0009    (flash fee on X)
    = X × (0.0375 − 0.0009)
    = X × 0.0366
```

Wait — this looks profitable. But the attacker also loses the collateral:

```
collateral_seized = X   →  goes to DEX pool, attacker loses X
net = −X + D × 1.05 − flash_fee − D
    = −X + X × 0.75 × 0.05 − flash_fee
    = −X + X × 0.0375 − flash_fee
    = −X × (1 − 0.0375) − flash_fee    < 0
```

The attack loses money. The attacker's own collateral is seized and given to the DEX. Self-liquidation is unprofitable by construction.

---

## 9. Liquidation Attack Vectors

### 9.1 Griefing Attack — PARTIALLY MITIGATED

**Scenario:** An adversary creates many small positions just above the liquidation threshold and then rapidly opens/closes them to waste liquidators' gas on unprofitable attempts.

**Mitigation:** The batch liquidation function checks each position's health factor before executing:

```
for each position:
    if HF ≥ 10000: continue    (skip healthy positions)
    execute liquidation
```

Healthy positions are skipped without reverting, so the gas cost of checking a healthy position is the cost of one health factor computation — modest. The adversary pays gas for the griefing transactions; the liquidator pays gas only for the batch call.

**Residual risk:** At very low position sizes, liquidation gas cost may exceed the 5% bonus, making small positions uneconomical to liquidate. This creates a potential accumulation of micro-bad-debt.

**Mitigation:** A minimum position size is enforced implicitly by the oracle price. At P_BTC = $85,000, a 1-satoshi collateral position has negligible economic impact. The practical minimum meaningful position is ~10,000 satoshis ($8.50), where the 5% bonus far exceeds gas costs.

### 9.2 Liquidation Front-Running — ACCEPTED RISK

**Scenario:** A liquidator broadcasts a liquidation transaction, and a miner/sequencer inserts their own liquidation transaction with higher priority.

**Analysis:** This is standard MEV (Maximal Extractable Value) behavior, present in all liquidation-based protocols. It does not compromise protocol safety — the position is liquidated regardless of which liquidator executes. The seized collateral still flows to the DEX, improving system health.

**Impact:** Honest liquidators may lose some fee income to MEV extractors. Protocol security is not affected.

### 9.3 Bad Debt Accumulation

**Scenario:** Market moves so rapidly that positions become undercollateralized before liquidation bots can react. HF falls below 1.0 and the collateral value is insufficient to cover the debt + bonus.

```
if P_btc drops so fast that:
C × P_btc < D (no bonus, no full repayment)
```

**Analysis:** The 5% gap between LTV threshold (75%) and liquidation threshold (80%) means a position can lose 6.25% of collateral value from max LTV before becoming liquidatable:

```
P_btc must fall by: (1 − 0.75/0.80) = 6.25%
```

For bad debt (collateral insufficient to cover debt):

```
Bad debt occurs when: P_btc falls by (1 − 0.75) = 25% from entry
```

Between 6.25% and 25% price decline, the position is liquidatable at a profit. Only declines >25% from entry create bad debt.

**Mitigation:** The perp insurance fund (seeded by vault + fees) absorbs bad debt. The DEX reserve deepening from liquidations builds the insurance base over time.

---

## 10. Governance Attack Vectors

### 10.1 Governance Capture via BTC Accumulation — ACCEPTED RISK

**Scenario:** An adversary acquires a majority of locked BTC (vePower) and votes to redirect vault allocation to a single module, then exploits that module.

**Analysis:** To control governance, the adversary must lock a majority of all governance-participating BTC for up to 1 year. At any non-trivial TVL, this is prohibitively expensive. Moreover:

The worst-case outcome of governance capture is a suboptimal allocation:
```
vote(w_dex=0, w_lend=0, w_perps=100)
```

This redirects all new vault deposits to the perps insurance fund. This does not allow the attacker to withdraw funds. It makes the vault's yield suboptimal, not insecure. Existing deployments to DEX and Lending are unaffected until `vaultRebalance()` is called.

**Critical property:** Governance in OP_NEXUS v1 controls only allocation percentages — not contract parameters, not fund withdrawal, not fee percentages. Governance capture results in reduced yield, not loss of principal.

### 10.2 Bribe-Driven Governance Manipulation

**Scenario:** An attacker bribes veBTC holders to vote for a specific allocation that benefits the attacker's position.

**Analysis:** This is legitimate economic behavior — bribes compensate voters for directing capital. The bribe mechanism is designed to allow this. The protocol captures value through fees regardless of allocation.

**Constraint:** The attacker must pay bribes from their own capital. If the profit from the favorable allocation is less than the bribe cost, the attack is irrational. Market equilibrium will price bribes at their true value to recipients, making sustained manipulation expensive.

### 10.3 Epoch Boundary Attacks

**Scenario:** An attacker rapidly acquires vePower just before an epoch boundary, votes to change allocation, then unlocks.

**Analysis:** `govUnlock()` in v1 immediately releases locked BTC. This creates a potential attack window:

```
Block T-1: Lock X BTC for 1-block duration → acquire vePower
Block T:   Vote to change allocation
Block T:   Unlock X BTC
```

This is a **known issue** with the v1 governance model. The impact is limited because:
1. Allocation changes only affect new deposits, not existing deployments
2. The attacker pays any lock duration constraint
3. 1-block locks produce negligible vePower: `vePower = X × 1 / 52416 ≈ 0.002% of X`

**v2 mitigation:** Minimum lock duration of 1,008 blocks (1 epoch) before votes take effect, and vePower decay to prevent flash governance.

---

## 11. Cross-Module Attack Scenarios

### 11.1 Options Collateral Drain via Flash Loan

**Scenario:** An attacker observes a large amount of options collateral in Pool 1. They attempt to drain it via flash loan + swap.

**Analysis:**

Options collateral is tracked in `_dexOptCollateral`. The free reserve available for flash loans is:

```
free = R0_pool1 − _dexOptCollateral − _dexPerpMargin − _dexFlashOut
```

If the attacker flash borrows the entire free reserve, the options collateral remains locked in the pool (it is not flash-borrowable). The flash repayment restores R0 exactly. The options collateral is untouched.

**Can options collateral be accessed via swap?** A swap extracts token1 by providing token0. To drain R0 (which includes options collateral), the attacker would need to provide enormous amounts of R1. The constant product invariant ensures that draining R0 toward zero requires R1 → infinity. This is economically impossible.

### 11.2 Bond Collateral Extraction via Lending

**Scenario:** An attacker issues a bond with large collateral in Lending, then attempts to extract that collateral by borrowing against it.

**Analysis:** Bond collateral is supply in the Lending module. Supply earns interest but is not immediately withdrawable — withdrawal is constrained by available liquidity:

```
withdrawable = supply_total − borrow_total
```

The bond collateral is part of supply_total. If the attacker borrows against collateral first (as a separate lending position), they:
1. Reduce withdrawable liquidity
2. Must maintain HF > 1.0 on their borrow position

They cannot both supply as bond collateral AND borrow from the same supply — these are separate accounting entries. The bond collateral is locked until the bond matures (`bondRedeem()`). Bond issuers cannot unilaterally withdraw collateral before maturity.

**Note:** The v1 contract tracks `_bndCollatInLend` as an accounting variable. The actual Lending supply balance includes this amount. A future version should enforce stronger isolation between bond collateral and regular supply.

### 11.3 Vault Allocation Manipulation via Repeated Deposits/Withdrawals

**Scenario:** An attacker rapidly deposits and withdraws from the vault, each time triggering auto-deployment at a specific allocation, to artificially inflate one module's capital.

**Analysis:** Deposits deploy capital to modules; withdrawals withdraw it proportionally. Net effect of deposit-then-immediate-withdrawal:

```
Deposit a:
  DEX     += a × alloc_dex / 100
  Lending += a × alloc_lend / 100
  Perps   += a × alloc_perps / 100

Withdraw all shares:
  DEX     -= a × alloc_dex / 100     (same amounts withdrawn)
  Lending -= a × alloc_lend / 100
  Perps   -= a × alloc_perps / 100
```

Net module state change: zero. The attacker pays fees on the deposit and withdrawal, losing money.

The only manipulation possible is temporary: between the deposit and withdrawal, modules have slightly more capital. This does not benefit the attacker materially.

---

## 12. Economic Attack Analysis

### 12.1 Death Spiral Scenario

**Scenario:** A large price drop in BTC causes mass Lending liquidations. Liquidated collateral flows to the DEX. Flood of collateral depresses the DEX price. Depressed DEX price causes more liquidations. Spiral continues.

**Analysis:** In OP_NEXUS, liquidated collateral goes to Pool 1 as reserve₀ (BTC side). This does not directly affect the oracle price, which is an EWMA of externally submitted values. The DEX spot price (reserve₀/reserve₁) temporarily falls as R₀ increases, but:

1. Arbitrageurs immediately rebalance by selling R₁ tokens to buy cheap R₀
2. The oracle price is unaffected by DEX spot price changes
3. Liquidation decisions use the oracle price, not the DEX spot price

**Death spiral via oracle:** If the BTC oracle price falls legitimately (external markets are crashing), then more positions become liquidatable. This is correct behavior — the system should liquidate undercollateralized positions in a falling market. The seized collateral deepens the DEX, providing more liquidity for the market to find a new equilibrium.

**Comparison with fragmented systems:** In Aave+Uniswap, large liquidations can cause cascading failures because liquidated collateral is sold on external AMMs, pushing the price down further, causing more liquidations. In OP_NEXUS, seized collateral goes into the DEX as liquidity, not as sell pressure. The death spiral mechanism is structurally weaker.

### 12.2 Vault Insolvency Scenario

**Scenario:** Simultaneous mass withdrawals from the vault when deployed capital cannot be quickly recalled from modules.

**Analysis:** The vault maintains `_vltDeployedDex`, `_vltDeployedLend`, `_vltDeployedPerps` as accounting of deployed capital. Withdrawal reduces `_vltAssets` and proportionally reduces deployed capital:

```
fromDex  = assets × alloc_dex / 100
fromLend = assets × alloc_lend / 100
```

In v1, withdrawal is pure accounting — the capital "recalled" from modules is not actually transferred (modules share state with the vault). In practice, `_lndTotalSupply` is decremented, which may reduce available lending liquidity.

**Bank run risk:** If all vault depositors withdraw simultaneously:
- `_lndTotalSupply` falls (reducing lend capacity)
- `_dexR0p1` effectively has less vault-backed capital
- Flash loan capacity falls

The system remains solvent because withdrawals cannot exceed deposits (INV-4). No depositor can withdraw more than they contributed plus yield.

**Conclusion:** The vault cannot become insolvent unless underlying modules generate bad debt that exceeds the insurance fund. This requires a catastrophic price event (>25% single-block BTC price drop, which has never occurred historically).

---

## 13. OP_NET Runtime Security

### 13.1 WASM Execution Isolation

OP_NET executes contract WASM in an isolated sandbox. Key security properties inherited from the runtime:

- **Memory isolation:** Contract WASM cannot access memory outside its own linear memory space
- **Determinism:** Given identical input and state, WASM execution always produces identical output
- **No system calls:** WASM has no access to the file system, network, or other system resources
- **Bounded computation:** OP_NET enforces computation limits per transaction, preventing infinite loops

### 13.2 Storage Security

`StoredU256` and `StoredBoolean` objects abstract over OP_NET's key-value storage layer. Storage pointers (16-bit identifiers) are compile-time constants — they cannot be computed dynamically and cannot collide unless the developer explicitly assigns the same pointer to two variables.

**OP_NEXUS storage pointer allocation** is non-overlapping by construction. The storage layout (Appendix A of the Whitepaper) assigns contiguous ranges per module with no collisions.

### 13.3 Calldata Parsing Security

The OP_NET `Calldata` class provides typed reading methods (`readU256()`, `readU8()`, `readStringWithLength()`) that are bounds-checked. Reading beyond the calldata length throws an exception that reverts the transaction.

Potential issue: a method that reads more calldata parameters than the caller provides will revert. This is safe (no partial state changes) but could be exploited for denial of service by sending undersized calldata to expensive methods. Cost to attacker: one transaction fee per attempt.

---

## 14. Known Limitations

### 14.1 Per-Address State

**Issue:** OP_NEXUS v1 does not maintain per-address accounting. The lending supply, borrow, and perp positions are global aggregates. There is no mechanism to track which address owns how many vault shares, which address has what lending position, or which address opened which perp position.

**Impact:** The contract correctly tracks system-wide invariants but cannot enforce per-user return of their specific capital. This is mitigated by the frontend, which submits parameters (shares to withdraw, position size to close) explicitly. But a malicious caller could submit another user's position parameters and trigger state changes on their behalf.

**Example:** `lendLiquidate(collateral, debt)` can be called by anyone with valid parameters — this is intentional (permissionless liquidations). But `lendWithdraw(shares)` does not verify that the caller has deposited those shares.

**Note:** This is consistent with the system-level accounting model where the entire protocol is one entity. User-level access control is the responsibility of the OP_NET account model. v2 will add `Blockchain.tx.origin` checks for sensitive operations.

### 14.2 No Position Mapping

**Issue:** Individual options, bonds, and liquidator positions are identified by IDs (`_optCount`, `_bndCount`, `_liqPositions`) but their parameters are not stored on-chain per-ID. The frontend must pass all parameters when exercising or redeeming.

**Impact:** A user who loses track of their position parameters (strike, collateral, expiry) cannot reconstruct them from on-chain state alone. This is a UX limitation, not a security vulnerability.

### 14.3 Oracle Trust Assumption

**Issue:** Phase 1 oracle relies on trusted node operators. See [Section 6.3](#63-coordinated-multi-node-attack).

### 14.4 vAMM Funding Rate Correction Lag

**Issue:** Funding rate corrects mark/oracle divergence over 144 blocks (~24 hours). Intraday divergence may exceed 10% in volatile markets.

**Impact:** Profitable perp strategies may exist during funding correction periods. These are bounded by the rate at which the vAMM mark price reverts to oracle, which is guaranteed within one funding interval.

### 14.5 No Emergency Stop

**Issue:** By design, OP_NEXUS has no pause mechanism. If a critical bug is discovered, it cannot be halted.

**Response:** Pre-deployment security analysis (this document) and formal verification of key invariants are the primary mitigations. Post-deployment, a new contract version can be deployed with the bug fixed. Users migrating from v1 to v2 would do so voluntarily.

---

## 15. Security Checklist

The following checklist represents the minimum verification performed before deployment. Each item is marked with its status.

### Smart Contract Security

- [x] All arithmetic uses `SafeMath` — no raw arithmetic on u256
- [x] All storage pointers are unique and non-overlapping
- [x] Flash loan reentrancy guard (`_dexFlashActive` mutex) implemented
- [x] Zero-amount checks on all deposit/borrow/swap operations
- [x] Division-by-zero protected (SafeMath.div, and zero checks on denominators)
- [x] Integer overflow impossible (u256 headroom analysis, Section 7.3)
- [x] Liquidation only possible when HF < 1.0 (health factor check enforced)
- [x] Bond collateralization ≥ 150% enforced at issuance
- [x] Options collateral deposited before premium accepted
- [x] Flash repayment verified before `_dexFlashActive` cleared
- [x] Vault PPS monotonicity maintained (Theorem 9.1)
- [x] DEX k-monotonicity maintained (Theorem 4.1)
- [x] Fee distribution sums to exactly 100% (residual method)
- [x] Oracle price positivity enforced
- [x] vePower bounded by locked amount
- [x] Governance vote weights sum check (total ≤ 100)

### Economic Security

- [x] Self-liquidation is unprofitable (Section 8.3 analysis)
- [x] Flash loan pool drain is impossible (Section 8.2 analysis)
- [x] Oracle manipulation dampened by EWMA (Section 6.1 analysis)
- [x] Bond self-funding holds at ≥ 150% collateral (Lemma 8.1)
- [x] Liquidation profitability guaranteed for valid HF < 1.0 (Theorem 12.1)
- [x] Capital conservation maintained across all operations (Theorem 2.1)

### Runtime Security

- [x] WASM compiled from audited AssemblyScript source
- [x] Storage layout non-overlapping (Appendix A)
- [x] Calldata parsing uses bounds-checked OP_NET methods
- [x] No external calls (no cross-contract reentrancy possible)
- [x] All reverts leave state unchanged (OP_NET atomicity)

### Pending (Pre-Mainnet)

- [ ] Independent security audit by third-party firm
- [ ] Formal verification of core invariants (INV-1 through INV-8)
- [ ] Economic simulation of liquidation cascade scenarios
- [ ] Oracle node operator agreements and stake requirements documented
- [ ] Per-address accounting for sensitive operations (v1.1)
- [ ] Minimum position size enforcement
- [ ] Governance epoch lock minimum (1,008 blocks before votes effective)

---

## 16. Responsible Disclosure Policy

### 16.1 Scope

The following are in scope for security research:

- `OPNexus.ts` — All contract logic
- `index.ts` — Entry point and abort handler
- Mathematical model vulnerabilities — Scenarios where the formulas produce incorrect outputs that create exploitable states
- Economic attack vectors — Profitable attacks requiring no code bugs, only adversarial use of legitimate functions
- Cross-module interaction bugs — Unexpected interactions between modules not covered by individual module analysis

The following are **out of scope:**

- OP_NET runtime vulnerabilities (report to OP_NET team directly)
- Bitcoin protocol vulnerabilities (report to Bitcoin Core)
- Frontend HTML/JS vulnerabilities (no on-chain impact)
- Attacks requiring control of >50% of oracle stake (accepted trust assumption)
- Theoretical attacks with cost exceeding realistic TVL

### 16.2 Severity Classification

| Severity | Definition | Example |
|---|---|---|
| **Critical** | Direct loss of user funds via non-obvious exploit | Flash loan enables draining vault without repayment |
| **High** | Loss of user funds under specific market conditions | Oracle manipulation enabling profitable liquidations consistently |
| **Medium** | Protocol operates incorrectly but funds are safe | Incorrect interest accrual formula |
| **Low** | Minor accounting discrepancy, no fund risk | Rounding error > 1 satoshi per operation |
| **Informational** | Improvement suggestion, no vulnerability | Inefficient gas usage |

### 16.3 Disclosure Process

1. **Identify** the vulnerability and develop a proof-of-concept demonstrating impact
2. **Do not** exploit the vulnerability on mainnet or share details publicly
3. **Report** via the contact information in the repository's SECURITY_CONTACT file
4. **Include** in your report:
   - Severity classification and rationale
   - Detailed description of the vulnerability
   - Step-by-step reproduction instructions
   - Estimated economic impact
   - Suggested mitigation
5. **Receive** acknowledgment within 72 hours and a full response within 7 days
6. **Coordinate** on disclosure timing (standard 90-day disclosure window)

### 16.4 Bug Bounty

Rewards for valid vulnerability reports are denominated in BTC:

| Severity | Reward Range |
|---|---|
| Critical | 1.0 – 5.0 BTC |
| High | 0.2 – 1.0 BTC |
| Medium | 0.05 – 0.2 BTC |
| Low | 0.005 – 0.05 BTC |
| Informational | 0.001 BTC |

Rewards are paid from the protocol treasury within 30 days of confirmed valid reports.

### 16.5 Safe Harbor

Security researchers acting in good faith under this policy will not be subject to legal action. Testing on testnet is encouraged. Testing on mainnet requires prior written coordination.

---

## 17. Audit Scope

The following provides guidance for independent security auditors approaching OP_NEXUS.

### 17.1 Priority Areas

**Priority 1 — Cross-Module State Consistency**

Verify that all cross-module capital flows maintain accounting consistency. Specifically:

- Lending liquidation → DEX: `_dexR0p1` increases by exactly the seized collateral amount
- Options write → DEX: `_dexOptCollateral` and `_dexR0p1` increase by the same amount
- Options exercise → DEX: `_dexOptCollateral` and `_dexR0p1` decrease by the same amount
- Bond issue → Lending: `_lndTotalSupply` and `_bndCollatInLend` increase by the same amount
- Flash borrow: `_dexR0p1` decrease equals `_dexFlashOut`
- Flash repay: `_dexR0p1` increase equals repayment, `_dexFlashOut` clears to zero

**Priority 2 — Arithmetic in Financial Formulas**

Review all formulas for edge cases:

- EWMA update with P_submitted = 0 (checked, reverts)
- EWMA update with P_submitted = u256.MAX (SafeMath.mul overflow check)
- LP minting with extremely imbalanced pool (single-satoshi R1, large R0)
- Interest accrual with `elapsed = 0` (no-op, handled)
- Interest accrual with `elapsed = 52560 × 100` (100-year elapsed, overflow check)
- Liquidation with `debt = 0` (checked, reverts)
- Vault deposit with `total_shares = 0` and `total_assets > 0` (impossible by invariant)

**Priority 3 — Flash Loan Security**

Verify the complete flash loan lifecycle:

- `_dexFlashActive` is always false before any flash operation begins
- `_dexFlashActive` is always set to true before any reserve deduction
- `_dexFlashActive` is only set to false after repayment is verified
- No code path can set `_dexFlashActive = false` without verifying repayment
- `_dexFlashOut` always equals the outstanding principal during active flash
- Nested flash calls correctly revert (mutex check)

**Priority 4 — Oracle Price Usage**

For every place in the contract where `_priceBTC` is read, verify:

- The usage is correct (price direction, numerator/denominator)
- Stale prices produce conservative (safe) behavior, not exploitable behavior
- Zero price is handled (oracle positivity invariant prevents this, but verify)

**Priority 5 — Governance Execution**

Verify that governance vote execution:

- Correctly applies new allocation weights (no off-by-one in percentage computation)
- Correctly computes new deployed capital after rebalance
- Cannot set `_lndTotalSupply` below `_lndTotalBorrow` (solvency invariant)
- `_vltAllocDex + _vltAllocLend + _vltAllocPerps = 100` always after vote

### 17.2 Testing Methodology

Recommended testing approaches:

**Fuzz testing:** Submit random calldata to all 70 methods across the entire u256 input space. Any state change that violates INV-1 through INV-8 is a finding.

**Invariant testing:** After every transaction in a simulated test suite, verify all 8 system-level invariants hold. Build a test oracle that checks invariants continuously.

**Scenario simulation:** Model the following specific scenarios:
1. 100 simultaneous liquidations (batch and individual)
2. Flash loan followed immediately by vault withdrawal
3. Options collateral = entire Pool 1 reserve (maximum collateral)
4. Governance vote changing allocation to (100, 0, 0) then (0, 0, 100)
5. Bond collateral at exactly 150% (minimum), then oracle price drops 33%

**Symbolic execution:** Apply symbolic execution tools to the compiled WASM to enumerate all possible execution paths and identify paths that lead to invariant violations.

### 17.3 What a Clean Audit Looks Like

A clean audit of OP_NEXUS demonstrates:

1. All 8 system-level invariants hold for all reachable states
2. No transaction sequence profitable by more than gas cost without providing value
3. Cross-module capital flows are exactly balanced (no phantom BTC creation)
4. Flash loan mutex cannot be bypassed
5. Oracle price manipulation is bounded per Section 6.1
6. All known limitations (Section 14) are documented and their impact scoped

---

*OP_NEXUS Security Analysis v1.0 · 2025*

*Security is not a feature. It is the foundation on which every feature is built.*

---

**END OF DOCUMENT**
