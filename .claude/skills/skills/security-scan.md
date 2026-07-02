
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

Output this as a "Recon Summary" section BEFORE proceeding to detection.

## Exploit Context

When analyzing for vulnerabilities, reference these real-world examples to calibrate your detection:

### Reentrancy Examples
- The DAO hack (2016): $60M stolen. Recursive call in withdraw drained funds before balance update.
- Cream Finance (2021): $130M. Read-only reentrancy via price oracle manipulation during reentrant call.
- Curve (2023): $62M. Vyper compiler bug allowed reentrancy despite @nonreentrant decorator.

Pattern to detect: External call to untrusted address BEFORE state update. Look for: .call{value:}, .transfer(), token.safeTransfer() followed by state changes.

### Oracle Manipulation Examples
- Mango Markets (2022): $114M. Attacker pumped spot price of MNGO token, borrowed against inflated collateral.
- Inverse Finance (2022): $15.6M. Used flash loan to manipulate Uniswap TWAP in single block.
- Bonq (2023): $120M. Manipulated TellorFlex oracle with single staking transaction.

Pattern to detect: Any use of spot price (balanceOf / totalSupply, or single DEX read) for value calculations. Look for: missing TWAP, missing price deviation checks, single-source oracles.

### Access Control Examples
- Ronin Bridge (2022): $624M. Compromised 5 of 9 multisig validators. Threshold too low.
- Wormhole (2022): $326M. Uninitialized proxy allowed attacker to call guardian set upgrade.
- Wintermute (2022): $162M. Vanity address generator vulnerability gave attacker private key access.

Pattern to detect: Missing access modifiers on functions that change critical state. Unprotected initialization. Single-point-of-failure admin patterns.

### Flash Loan Examples
- bZx (2020): $8M. Flash loan + oracle manipulation + leveraged trade in single transaction.
- Pancake Bunny (2021): $45M. Flash loan to manipulate LP token price, then deposit at inflated value.
- Euler Finance (2023): $197M. Flash loan + donate function to create bad debt.

Pattern to detect: Functions where large temporary capital could create profit. Price calculations that use current reserves. Donation attacks on share-based systems.

When reporting findings, reference the most similar real exploit to justify the severity and demonstrate the attack is practical, not theoretical.



# Choose Your Verification Approach

## Kill Gates (Rules-Based Filtering)
Apply deterministic rules to each finding. If it fails any gate, it gets killed.

## Result: 100% precision across 50 blind contests. Zero false positives.

## Devil's Advocate (Adversarial Challenge)
A second analysis pass challenges every finding. "Prove this is exploitable or it gets demoted."

## Confidence Scoring (Statistical Weighting)
Assign confidence scores based on how many detection methods flagged the same issue. High confidence = likely real. Low confidence = likely FP.

## Verification Phase: Devil's Advocate✅

After detection, perform a SECOND independent analysis that challenges every finding.

For EACH finding from the detection phase, apply the Devil's Advocate protocol:

### Challenge 1: Protection Elsewhere
Search the ENTIRE codebase for protection mechanisms that prevent this exploit.
- Is there a reentrancy guard in a parent contract?
- Is there access control checked via a modifier?
- Is there input validation in a function called earlier in the chain?
- Does SafeERC20 or a similar library handle the edge case?
If protection exists → DEMOTE to "Candidate" with explanation.

### Challenge 2: Economic Infeasibility
Calculate whether this exploit is profitable:
- What is the minimum capital required?
- What are the gas costs of the attack transaction?
- What is the maximum extractable value?
- Could the attack be sandwiched or front-run by others?
If cost exceeds profit → DEMOTE.

### Challenge 3: Attack Dry Run
Construct a step-by-step attack scenario:
1. Attacker starts with X ETH/tokens
2. Attacker calls function A with parameters B
3. Contract state changes to C
4. Attacker calls function D
5. Attacker ends with Y ETH/tokens (profit = Y - X)
If you cannot complete the dry run → DEMOTE.

### Challenge 4: Design Intent
Review code comments, variable names, and README for evidence this behavior is intentional.
- Does a comment say "by design" or "intentional"?
- Is there a documented tradeoff being made?
- Is this a known limitation mentioned in docs?
If intentional → DEMOTE.

### Challenge 5: Severity Validation
For HIGH/CRITICAL findings, the bar is higher:
- Can you name a similar real-world exploit? (reference Solodit, rekt.news)
- Would this be reported as H/M in a Code4rena contest?
- Is the impact quantifiable (not just "could lead to loss")?
If severity cannot be justified → DOWNGRADE.

### Challenge 6: Inversion
Try to PROVE the code is SAFE. Argue the opposite of the finding.
What would need to be true for this NOT to be a vulnerability?
If you can convincingly prove safety → KILL.

### Classification After Challenge:
- **Proved**: Survived all 6 challenges with concrete evidence → Include in report
- **Confirmed Unproven**: Likely real but cannot fully prove → Include with caveat
- **Candidate**: Failed 1-2 challenges → Include only if HIGH+ severity
- **Discarded**: Failed 3+ challenges → Kill with reasoning logged

# Choose Your Tool Integration

## AI Only
No external dependencies. Your skill is a single markdown file. It works anywhere Claude Code runs.

Pros: Zero setup for users. No dependencies to install or maintain. Portable across machines. Cons: Misses patterns that static analysis catches trivially (dead code, shadowing, unused returns). Cannot prove findings with formal verification.

## Static Analysis Integration

Run Slither and/or Aderyn first, feed their output to your AI. The AI reasons over tool findings, filters false positives, and enriches reports with context.

Pros: Catches structural issues. Cross-validates AI findings with deterministic tools. Reduces false positives (if Slither agrees, it is more likely real). Cons: Requires Slither/Aderyn installed. Adds complexity. Token usage increases (feeding tool output to AI).

## Full Stack (Static + Fuzzing + Formal)

Everything in static analysis plus fuzzers (Echidna, Medusa) for property testing and Halmos for formal verification. Findings that can be proven with a test or formal proof are dramatically more credible.

Pros: Highest confidence findings. Provable vulnerabilities. Catches complex multi-step bugs that pure analysis misses. Cons: Heavy dependencies. Slow. Requires the project to have a working build setup. Complex MCP configuration.

*** Adding Static Analysis ***

## Prerequisites

Before running this audit skill, run static analysis tools on the project:

### Option 1: Slither (Python)
Install: `pip install slither-analyzer`
Run: `slither . --json .audit/slither-output.json`

### Option 2: Aderyn (Rust)
Install: `cargo install aderyn`
Run: `aderyn . --output .audit/aderyn-report.md`

If you have run either tool, this skill will automatically read the output and cross-reference findings.

## Tool Integration

When starting the audit, check for static analysis results:

1. Look for `.audit/slither-output.json` in the project root
   - If found: read and extract detector findings (name, severity, file, line)
   - Use these as additional signals during detection
2. Look for `.audit/aderyn-report.md` in the project root
   - If found: read and extract issues
   - Use these as additional signals during detection

### How to Use Tool Findings
Static analysis findings serve as HINTS, not final results:
- If a tool finding overlaps with an AI finding → BOOST confidence
- If a tool finds something the AI missed → Investigate further
- If the AI finds something tools missed → Keep it (tools miss semantic issues)
- Tool-only findings without AI corroboration → Include but mark as "Tool-detected, needs review"

If no tool output files are found, proceed with AI-only analysis and note in the report: "Static analysis results not provided. Run Slither or Aderyn for enhanced detection."

