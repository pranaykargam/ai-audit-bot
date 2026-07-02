

# Multi-Phase Security Auditor

## When Invoked

Perform a structured, multi-phase security audit of all Solidity (.sol) files in the current project.

## Phase 1: Reconnaissance

Before looking for any bugs, understand the codebase:

1. **File inventory**: List all .sol files with line counts
2. **Architecture map**: Identify contract relationships (inheritance, imports, calls)
3. **Entry points**: List all external/public functions
4. **Value flows**: Identify where tokens/ETH move (transfers, approvals, minting)
5. **Risk scoring**: Rank files by risk (more external calls + state writes + value handling = higher risk)
6. **Dependencies**: Note any external protocols or oracles used

Output this as a "Recon Summary" before proceeding to detection.

## Phase 2: Detection

Starting with the highest-risk files from Phase 1, analyze each contract for:

**For each external/public function:**
- Can it be called by anyone? Should it be restricted?
- Does it follow checks-effects-interactions pattern?
- Are there external calls? What happens if they fail or reenter?
- Are arithmetic operations safe? Could they overflow or lose precision?
- Are there any trust assumptions that could be violated?

**For each state variable:**
- Is it updated correctly across all functions that modify it?
- Could it be manipulated by an external actor?
- Is it used in calculations that affect value?

**Cross-function analysis:**
- Are there state variables modified by multiple functions that could conflict?
- Are there functions that should be called in a specific order?
- Are there invariants that should always hold?

Record ALL potential findings before moving to Phase 3.

## Phase 3: Verification

For EACH finding from Phase 2, ask:

1. **Is this exploitable?** Describe the exact steps an attacker would take.
2. **Who is the attacker?** (external user, contract owner, privileged role, other protocol)
3. **What do they gain?** (stolen funds, DoS, governance manipulation)
4. **How much is at risk?** (quantify if possible)
5. **Is this intentional design?** (check comments, documentation, README)
6. **Is this already protected?** (check for guards, modifiers, validation elsewhere)

**KILL the finding if:**
- No concrete exploit steps can be described
- The impact is purely theoretical
- It requires admin/owner to act maliciously (trust assumption)
- The loss is less than gas costs to exploit
- It is a generic best practice, not a specific vulnerability

Only findings that survive all checks proceed to Phase 4.

## Phase 4: Report

For each verified finding:

**[SEVERITY] Title**
- **File**: contract.sol:lineNumber
- **Description**: What is wrong and why
- **Exploit Scenario**: Step-by-step attack path
- **Impact**: Quantified damage
- **Recommendation**: Specific fix with code if possible

Severity: CRITICAL (fund loss), HIGH (significant loss or protocol break), MEDIUM (conditional loss or DoS), LOW (minor issues)

Begin with the Recon Summary, then list verified findings by severity.


## Detection Phase: Domain-Specific Analysis

After completing reconnaissance, classify the protocol type and apply targeted detection.

### Step 1: Protocol Classification

Based on the recon results, classify the protocol as one of:
- **DEX/AMM**: Swap functions, liquidity pools, LP tokens, price curves
- **Lending**: Borrow/repay, collateral, liquidation, interest rates
- **Staking**: Deposit/withdraw, reward distribution, lock periods
- **Governance**: Voting, proposals, timelocks, delegation
- **Vault/Yield**: Deposit/withdraw, strategies, share calculation
- **NFT/Gaming**: Minting, transfers, marketplace, randomness
- **Bridge/Cross-chain**: Message passing, token locking, relay verification
- **Other**: If none match, use general detection

State which domain was detected and why.

### Step 2: Apply Domain Primer

#### If DEX/AMM:
- Constant product invariant (x*y=k) maintained after every operation?
- Swap fee calculation correct? (997/1000 pattern, rounding)
- LP token mint/burn proportional to reserves?
- Oracle (TWAP) updated correctly on every trade?
- Minimum liquidity lock prevents first-depositor attacks?
- Flash swap callback validates K invariant?
- MEV protection: deadline checks, slippage bounds?
- Router: token sorting, pair address computation correct?

#### If Lending:
- Collateral factor correctly applied?
- Liquidation threshold and bonus calculated correctly?
- Interest rate model: utilization-based, compounding correct?
- Bad debt handling: what happens when collateral < debt?
- Oracle dependency: stale price, manipulation resistance?
- Flash loan: fee calculation, callback validation?
- Share-based accounting: rounding direction favors protocol?

#### If Staking:
- Reward per token calculation correct across deposits/withdrawals?
- Reward distribution handles zero total supply?
- Lock period correctly enforced?
- Withdrawal timing: can someone deposit right before reward?
- Slashing: correctly reduces stake, handles edge cases?
- Compounding: auto-compound logic correct?

#### If Governance:
- Voting power snapshot at proposal creation, not execution?
- Quorum calculated correctly (total supply vs circulating)?
- Timelock enforced between proposal and execution?
- Delegation: voting power transfers correctly?
- Flash-vote protection: snapshot mechanism?
- Proposal cancellation: who can cancel, when?

#### If Vault/Yield:
- Share price calculation: deposits/withdrawals proportional?
- First depositor attack: minimum deposit or seed liquidity?
- Strategy losses: how are they socialized?
- Emergency withdrawal: bypasses strategy?
- Fee calculation: management fee, performance fee correct?
- ERC-4626 compliance: preview functions match actual?

### Step 3: General Checks (Always Apply)
Regardless of domain, also check:
- Access control on sensitive functions
- Reentrancy on value-transferring functions
- Integer safety on user-controlled inputs
- Event emission for state changes
- Constructor/initializer protection

For each issue found, report with: domain context, specific invariant violated, affected function, and recommended fix.


## Reconnaissance Phase

## Reconnaissance: Understanding Before Hunting

*** Why Recon Matters ***
- 
The single biggest mistake AI auditing tools make is jumping straight to vulnerability detection without understanding the codebase first. The result: findings about "missing slippage protection" in a governance contract, or "reentrancy risk" in a pure view function. Without context, your AI will produce irrelevant findings.

Reconnaissance gives your auditor context. Before hunting bugs, it should know:

What contracts exist and how big they are
Which files handle the most value (highest risk)
How contracts call each other (trust boundaries)
What external protocols are used (dependencies)
What the protocol is actually trying to do

Before analyzing for vulnerabilities, perform these steps:

### 1. File Inventory
List all Solidity (.sol) files in scope. For each file, report:
- File path
- Line count
- Whether it is a contract, interface, library, or abstract contract
- Number of external/public functions
- Number of state variables

### 2. Risk Scoring
For each contract (not interfaces or libraries), calculate a rough risk score:
- +5 for each external call (address.call, token.transfer, etc.)
- +4 for each state-modifying external/public function
- +4 for each payable function
- +3 for each use of assembly or unchecked blocks
- +1 for each 20 lines of code

Sort files by risk score descending. The top files get the deepest analysis.

### 3. Architecture Overview
Identify:
- Contract inheritance chains
- Which contracts call which other contracts
- Trust boundaries (who can call what, access control patterns)
- External protocol integrations (oracles, DEXes, lending pools)

### 4. Entry Point Catalog
List all external/public functions with:
- Function signature
- Whether it modifies state
- Whether it handles value (ETH or tokens)
- Access control (onlyOwner, role-based, or unrestricted)


