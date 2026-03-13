# OP_NEXUS — L1 Financial Engine on Bitcoin

<div align="center">

```
  ██████╗ ██████╗       ███╗   ██╗███████╗██╗  ██╗██╗   ██╗███████╗
 ██╔═══██╗██╔══██╗      ████╗  ██║██╔════╝╚██╗██╔╝██║   ██║██╔════╝
 ██║   ██║██████╔╝      ██╔██╗ ██║█████╗   ╚███╔╝ ██║   ██║███████╗
 ██║   ██║██╔═══╝       ██║╚██╗██║██╔══╝   ██╔██╗ ██║   ██║╚════██║
 ╚██████╔╝██║           ██║ ╚████║███████╗██╔╝ ██╗╚██████╔╝███████║
  ╚═════╝ ╚═╝           ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝
```

**The first complete Financial Operating System on Bitcoin Layer 1**

OP_NEXUS: The Financial Operating System for Bitcoin L1
A unified platform of 13 interconnected DeFi protocols, operating as a single living organism on Bitcoin.


Main Contract: opt1sqr8y4gcjm3syum7awmu68pylqlynmxah3qeuu4m7
Network:       Bitcoin L1 via OP_NET
Status:        Active testing on testnet — seeking real-world adoption to mature security

Philosophy: Money Never Sleeps — Neither Should Your Capital

In traditional DeFi, every protocol is an island. You have to move your capital between a DEX, a lending market, a yield vault, and a futures exchange. Every time you move funds, you pay fees, lose time, and—most importantly—your capital sits idle during the transfers.

OP_NEXUS is not a collection of 13 applications glued together. It is a single system where 13 financial primitives live together, share the same state, the same liquidity, and feed each other in real time.

When your money is in OP_NEXUS, it never sleeps. Every satoshi is always working: as liquidity in the DEX, as collateral in loans, as backing for perpetual futures, or generating yield in the vault. And most importantly: it moves from one module to another without you having to do anything, because the system is designed so that capital flows autonomously to where it can generate the most return.

Vision: 13 Modules, One Contract
Unlike other platforms that deploy separate contracts for each functionality (fragmenting liquidity and multiplying risk), OP_NEXUS concentrates all logic into a single smart contract on Bitcoin L1. This means:

Unified liquidity: The DEX, lending, futures, and options all draw from the same pool of capital.

Atomic flows: Operations that involve multiple modules (like liquidating a loan using a flash loan and automatically deepening DEX liquidity) happen in a single transaction, with no intermediary risk.

Bitcoin‑inherited security: The settlement layer is the world's most secure blockchain.

No bridges, no external contracts: Everything is inside the same vault.

Below we break down each module, its function, its individual benefits, and how it connects to the others.

The 13 Modules of OP_NEXUS

1. Oracle Net — The Central Nervous System
What it does
A decentralized price feed. A set of nodes staking BTC report prices for BTC, MOTO, and PILL using an exponential weighted moving average (EWMA) that smooths manipulation.

Benefits

A single trusted price for all modules (lending, perps, options, bonds, etc.).

Malicious nodes are slashed (5% of stake) and funds go to a public goods pool (QuadFund).

Other modules don't need their own oracle; all trust the same one.

Interconnections

Lending uses it to calculate health factors and LTV.

Perps uses it for mark price and liquidations.

Options uses it to determine if an option is ITM at exercise.

Bonds indirectly via the health factor of the collateral.

Liquidator monitors positions using it.

2. DEX AMM — The Unified Liquidity Hub
What it does
A decentralized exchange with three liquidity pools (BTC/MOTO, BTC/PILL, MOTO/PILL) using the classic x·y=k formula. But this is not just any DEX — it's the financial heart of the system.

Benefits

Traders can swap tokens with a 0.30% fee (0.27% to LPs, 0.03% to protocol).

Liquidity providers earn fees from every swap, and those fees are automatically reinvested (the pool deepens itself).

The DEX is the reserve from which flash loans are drawn — no separate pool for quick loans.

Interconnections

Flash Loans: Can borrow directly from the DEX reserves (up to free liquidity).

Options: Writer collateral is deposited into pool 1, where it earns swap fees while the option is live.

Perps: Trader margin is locked in pool 1, acting as real backing for the derivatives market.

Lending: When a lending position is liquidated, the seized collateral is automatically sent to pool 1, deepening liquidity instead of being sold externally.

Vault: A portion of the vault's capital is deployed in the DEX (default 40%), generating yield for vault depositors.

3. Flash Loans — Unc collateralized Loans with Purpose
What it does
Allows borrowing any amount of BTC (up to the free liquidity of pool 1) as long as it is returned in the same transaction, plus a 0.09% fee.

Benefits

Useful for arbitrage, liquidations, and complex strategies.

The fee goes entirely to the LPs of pool 1, not to the protocol. This incentivizes providing liquidity.

Interconnections

Router: Executes routes that start with a flash loan (e.g., flash liquidation).

Lending & Perps: Can be used to liquidate positions without own capital (see Router module).

DEX: Draws directly from DEX reserves.

4. Lending (BTCLend) — Collateralized Lending
What it does
Allows users to deposit BTC as collateral and borrow up to 75% of its value (LTV). Loans accrue variable interest based on pool utilization.

Benefits

Depositors earn an APY (currently up to 3.42%).

Borrowers can leverage without selling their BTC.

Seized collateral from liquidations is not sold — it goes to the DEX, avoiding sell pressure and benefiting LPs.

Interconnections

DEX: Liquidations feed pool 1.

Bonds: Bond issuers deposit their collateral in the lending module, generating interest that funds the bond yield.

FracMarket: NFT fractions can be used as collateral (with a 50% haircut) for loans.

Oracle: Needs BTC price to calculate health factors.

Liquidator: The liquidation module constantly monitors lending positions.

5. Perpetual Futures (PerpForge) — vAMM Perpetuals
What it does
A perpetual futures market with leverage up to 50x, using a virtual AMM (vAMM) for price discovery. Trader margin is deposited into the DEX (pool 1) and an insurance fund (seeded by the vault and fees) covers losses.

Benefits

Allows long or short positions with leverage without needing a counterparty.

Trading fees (0.30% per open/close) are distributed to the protocol.

The insurance fund grows with every trade, protecting traders from catastrophic losses.

Interconnections

DEX: Margin is locked in pool 1, and the vAMM uses the DEX price as reference.

Vault: A portion of the vault (default 25%) is deployed into the perps insurance fund, generating yield.

Oracle: Uses BTC price for mark price and funding rate calculations.

Liquidator: Leveraged positions can be liquidated if health factor drops below 1.

6. Options Market — European Options on BTC
What it does
Allows writing (selling) and buying CALL and PUT options on BTC, with BTC collateral. Options are European (exercisable only at expiry).

Benefits

Option writers deposit collateral that earns swap fees in the DEX while the option is active (their collateral sits in pool 1).

Buyers can speculate on volatility by paying a premium.

Exercise is settled automatically against the oracle, with no external intervention.

Interconnections

DEX: Option collateral lives in pool 1, earning fees.

Oracle: Determines if the option is ITM at expiry.

Lending: Option collateral could be used as loan backing in the future (v2).

7. Yield Bonds — Fixed‑Income Bonds
What it does
Issuance of over‑collateralized bonds (minimum 150%) paying a fixed APY. The issuer deposits BTC into the lending module, and that collateral earns interest that funds the bond payments.

Benefits

Investors get predictable returns.

Issuers can access liquidity without selling their BTC, and if the lending APY exceeds the bond APY, they capture the spread.

Backed by real collateral in lending.

Interconnections

Lending: Bond collateral is deposited in lending, earning interest.

Oracle: Indirectly, the health factor of the collateral depends on BTC price.

Vault: Bonds could be purchased by the vault as an investment (future versions).

8. YieldForge Vault — The Yield Accumulator
What it does
An ERC‑4626‑style vault that accepts BTC deposits and automatically deploys them into DEX, Lending, and Perps according to allocation percentages voted by veBTC holders. It also collects fees from all modules and reinvests them (40% of all protocol fees go back to the vault).

Benefits

Depositors gain exposure to multiple yield sources in one position.

The vault share price (PPS) always increases (or stays flat), never decreases, because reinvested fees increase total assets.

Money never sleeps: even while you decide what to do, your capital is already working in the underlying modules.

Interconnections

DEX, Lending, Perps: The vault deploys capital into these three modules.

Governance: veBTC holders vote on allocation percentages (default 40/35/25).

All modules: Every time any module generates a fee, 40% goes to the vault, increasing its assets.

9. veBTC Governance — Real‑Time Governance Power
What it does
Users can lock BTC for a period (up to 1 year) to receive vePower, which lets them vote on the vault's allocation. More BTC and longer lock = more voting power.

Benefits

Voters receive 30% of all protocol fees, distributed proportionally.

Allocation decisions take effect immediately (no multi‑day delays), because they only affect new deposits, not existing ones.

Bribes can be added to incentivize voting toward certain allocations.

Interconnections

Vault: Votes change the vault's allocation percentages.

Fee distribution: 30% of all fees go to voters.

10. Liquidator — The Automatic Watchdog
What it does
Monitors lending and perps positions, and allows anyone to liquidate positions with health factor < 1.0, earning a 5% bonus on the debt.

Benefits

Keeps the system healthy by removing undercollateralized positions.

The bonus incentivizes a decentralized network of liquidators.

Batch liquidations: Up to 10 positions can be liquidated in one transaction.

Interconnections

Lending and Perps: It executes liquidations in both modules.

DEX: Seized collateral is sent to pool 1, deepening liquidity.

Router: Flash liquidations can be performed without own capital (see Router).

11. FracMarket — NFT Fractionalization
What it does
Allows NFTs (OP_721) to be fractionalized into 1,000,000 fungible shares. These shares can be listed, bought, and even used as collateral in lending (with a 50% haircut).

Benefits

Democratizes access to high‑value NFTs.

Fraction holders can obtain liquidity without selling the whole NFT (by using fractions as collateral).

Creates a secondary market for fractions.

Interconnections

Lending: Fractions can be deposited as collateral (with reduced valuation).

DEX: Fractions could eventually be listed on the DEX (v2).

12. QuadFund — Quadratic Funding for Public Goods
What it does
A matching pool that receives 20% of all protocol fees and distributes it to projects via quadratic funding. Anyone can contribute to a project, and the pool matches contributions in a way that projects with many small contributors receive more matching.

Benefits

Funds OP_NET ecosystem development in a decentralized, democratic way.

Requires no human decisions: it is auto‑fed by every protocol operation.

Oracle nodes that are slashed also contribute to this fund.

Interconnections

All modules: Every fee from any module contributes 20% to QuadFund.

Oracle: Slashed stake goes to QuadFund.

13. Atomic Router — The Strategy Orchestrator
What it does
Enables operations that involve multiple modules in a single atomic transaction. Predefined routes include:

Flash Liquidation: Take a flash loan, liquidate a position, repay the loan, and keep the profit — all without own capital.

Borrow‑Swap‑Bond: Borrow against collateral, swap on the DEX, and issue a bond with the proceeds.

Flash Option Exercise: Use a flash loan to exercise an option, sell the payoff on the DEX, and repay the loan.

Circular Arbitrage: Exploit price discrepancies among the three DEX pools to capture profit.

Benefits

Native composition: Any multi‑step strategy runs in one transaction, with no intermediate front‑running risk.

Deep liquidity access: By using the DEX as the flash loan source, strategies can scale without artificial limits.

The router proves that the 13 modules are truly one system: You can touch them all in a single call.

Interconnections

All modules: The router has direct access to each module's functions and can combine them arbitrarily.

Why OP_NEXUS is Unique
        Feature                      In Traditional DeFi  	                          In OP_NEXUS
       Liquidity	      Fragmented across multiple pools	             Unified: the DEX is the central pool
     Option collateral	            Idle while waiting	                        Earns swap fees in the DEX
      Liquidations	   Collateral sold externally (sell pressure)	  Injected into the DEX (deepens liquidity)
      Flash loans	       Separate pool, often small	              Unlimited (up to DEX liquidity)
      Governance	    Proposals and execution with delays	                 Immediate on new allocations
        Fees	            Each protocol has its own treasury	       40% reinvested in the vault (everyone benefits)
    Number of contracts	       Multiple (one per protocol)	         Single contract (smaller attack surface)
     Settlement layer	          Ethereum, BSC, etc.	                              Bitcoin L1

Technical Overview 
Single contract: Written in AssemblyScript, compiled to WASM (~47 KB), deployed on OP_NET.

Storage: Global variables at 16‑bit pointers, all as u256 (SafeMath on every operation).

Oracle: EWMA with α = 1/8, staked nodes, 5% slash for deviation >5%.

Fees: 0.30% per operation, distributed: 40% vault, 30% veBTC, 20% QuadFund, 10% treasury.

Flash loans: 0.09% fee, 100% to DEX LPs.

Vault: ERC‑4626 logic, share price always non‑decreasing.

Governance: vePower = amount × duration / 52416 (max 1 year).

Liquidations: 5% bonus to liquidator; seized collateral goes to DEX.

Current Status and Roadmap
OP_NEXUS is currently deployed on OP_NET testnet and available for experimentation at:

text
opt1sqr8y4gcjm3syum7awmu68pylqlynmxah3qeuu4m7
Why still in testing?
The security of such an interconnected system cannot be proven by theoretical audits alone. It requires real use, real users, and real capital at stake to uncover unexpected behaviours. That is why we invite the community to test, break, and suggest improvements. Only through constant interaction can we mature the system to a reliable production state.

Planned phases:

Active testnet (current): Contract deployed, functional frontend, invitation to developers and users to experiment.

Independent security audit: After the system has been exercised and obvious issues corrected.

Mainnet: Deployment on Bitcoin L1 with real liquidity, active governance, and decentralized oracles.

V2: Enhancements such as per‑address accounting, vePower decay, more trading pairs, integration with RUNES/BRC‑20.

Conclusion: Money Never Sleeps, and in OP_NEXUS It Doesn't Either
OP_NEXUS is not just another protocol. It is a complete ecosystem where every satoshi is always generating value, moving from one module to another according to market conditions, without you having to do anything. It is the first time on Bitcoin L1 that such a level of integration and versatility exists.

We invite all Bitcoin builders, curious minds, and skeptics to try OP_NEXUS, challenge it, and help us build the financial future of Bitcoin.

Because money that doesn't work loses value. And on Bitcoin, work never stops.