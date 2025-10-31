## Common Vulnerabilities & Risks

Smart contracts can fail in countless ways, but certain patterns recur frequently. Below are some of the most prominent vulnerabilities we encounter, along with notes on how they manifest in practice:

1. Reentrancy Attacks

Reentrancy attacks are among the oldest yet still highly impactful types of exploits, with the most famous example being The DAO hack. When a contract fails to manage external calls correctly, attackers can repeatedly invoke a function before the initial contract call has finished its initial logic. This can lead to inconsistent state and eventually to draining of funds or other unintended state changes. In modern variants, even functions that appear “read-only” can be leveraged for reentrancy if the code relies on state-dependent logic or external calls without proper checks.

Example: A withdraw function that updates balances only after sending tokens, coupled with a malicious token contract that invokes back the calling contract, allowing an attacker to call withdraw multiple times.

2. Integer Overflows & Underflows

In languages like Solidity (prior to certain version updates) and in some custom libraries, calculations that exceed numeric bounds can “wrap around” in unexpected ways.

Impact: Attackers might artificially inflate their token balances or cause token supply manipulations.

3. Access Control Misconfigurations

Failure to restrict privileged functions or overlooking necessary checks for admin roles can hand attackers or malicious insiders unrestricted power.

Impact: Funds can be frozen, minted, or drained if the wrong party obtains these permissions.

4. Price Oracle Manipulation & MEV Attacks

Smart contracts often rely on on-chain oracles for data like token prices. If these oracles are manipulated, attackers can exploit the contract’s trust in the data. Similarly, Miner Extractable Value (MEV) or front-running can reorder transactions to an attacker’s advantage.

Impact: Unfair token swaps, manipulated lending/borrowing rates, or sandwich attacks in DEXes.

5. Unprotected External Calls

Contracts that call external untrusted contracts or handle user-supplied addresses without proper checks can inadvertently trigger fallback functions, reentrancy, or other malicious code paths.

Example: A contract awarding tokens to a user-supplied address without verifying if it’s a contract with potential reentrancy triggers.

6. Lack of Input Validation

Smart contracts that fail to adequately validate user input, such as function parameters or data payloads, risk enabling a wide range of attacks, from logical errors to injection-based vulnerabilities.

Impact: Malicious or malformed inputs can bypass intended restrictions, leading to partial or full contract compromise.

7. Logic & Business Rule Errors

Even well-known security pitfalls don’t cover all the ways a project’s custom logic can fail. Incorrect assumptions, poorly structured code, or edge cases in business rules can break functionality or cause financial damage.

