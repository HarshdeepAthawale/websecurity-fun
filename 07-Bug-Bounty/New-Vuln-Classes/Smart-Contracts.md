# Smart Contract Security

← [[VRT-Overview]]

---

Smart contract security is its own deep field, but as a web security researcher you'll encounter it on bug bounty programs for DeFi protocols, NFT platforms, and crypto exchanges. This note covers the VRT-relevant attack classes at an overview level.

> [!NOTE]
> This is a web security vault — this note covers smart contract vulns as they appear in the VRT for bug bounty context. For deep smart contract auditing, see dedicated resources like Secureum, Trail of Bits, and the Ethereum Foundation's security documentation.

---

## VRT Severity Reference

| Vulnerability | P-Rating |
|--------------|---------|
| Reentrancy Attack | P1 |
| Smart Contract Owner Takeover | P1 |
| Unauthorized Transfer of Funds | P1 |
| Uninitialized Variables | P1 |
| Integer Overflow / Underflow | P2 |
| Unauthorized Smart Contract Approval | P2 |
| Function-Level Denial of Service | P3 |
| Improper Fee Implementation | P3 |
| Irreversible Function Call | P3 |
| Malicious Superuser Risk | P3 |
| Bypass of Function Modifiers and Checks | Varies |
| Flash Loan Attack | Varies |
| Pricing Oracle Manipulation | Varies |
| Improper Implementation of Governance | Varies |
| Improper Decimals Implementation | P4 |
| Improper Use of Modifier | P4 |
| Inaccurate Rounding Calculation | Varies |

---

## Reentrancy Attack (P1)

The most famous smart contract vulnerability. The DAO hack (2016, $60M stolen) was a reentrancy attack.

**How it works:**

```solidity
// Vulnerable withdrawal function
function withdraw(uint amount) public {
    require(balances[msg.sender] >= amount);
    
    // 1. Send ETH to caller — this can trigger caller's fallback function
    msg.sender.call{value: amount}("");
    
    // 2. Update balance — but attacker re-entered before we got here!
    balances[msg.sender] -= amount;
}
```

**Attack:**
The attacker deploys a contract whose fallback function calls `withdraw()` again before the balance is updated. Because step 1 (send ETH) happens before step 2 (update balance), the attacker can drain the contract.

**Fix:** Update state before external calls, or use a reentrancy guard (`nonReentrant` modifier in OpenZeppelin).

---

## Integer Overflow / Underflow (P2)

Solidity versions before 0.8.0 don't have built-in overflow checks:

```solidity
// Vulnerable in Solidity <0.8.0
uint8 balance = 0;
balance -= 1;  // Underflow: wraps to 255
```

**Real-world impact:** An attacker with 0 balance calls withdraw, underflows to a massive number, and drains the contract.

**Fix:** Use SafeMath library (pre-0.8.0) or upgrade to Solidity 0.8+ which has built-in overflow protection.

---

## Uninitialized Variables / Storage Pointers (P1)

```solidity
// Vulnerable: uninitialized storage pointer
function vulnerable() public {
    address[] storage arr;  // points to storage slot 0 by default
    arr.push(msg.sender);   // overwrites contract owner stored at slot 0
}
```

---

## Flash Loan Attacks (Varies)

Flash loans let you borrow unlimited funds within a single transaction — you must return them before the transaction ends. Attackers use flash loans to manipulate prices, exploit arbitrage, or meet collateral requirements temporarily.

**General pattern:**
1. Borrow huge amount via flash loan
2. Manipulate a price oracle by dumping tokens into a pool
3. Use the manipulated price to exploit another protocol
4. Repay the flash loan in the same transaction

---

## Oracle Manipulation (Varies)

DeFi protocols rely on price feeds ("oracles") to know the current price of assets. If the oracle uses an on-chain DEX price that can be manipulated:

1. Attacker takes a flash loan
2. Dumps large amount into the DEX → price drops dramatically
3. Protocol reads the manipulated price and allows under-collateralized borrowing
4. Attacker borrows against the depressed price
5. Repays flash loan — profits from the difference

---

## Testing Smart Contracts in Bug Bounty

Most smart contract bug bounty programs want you to submit issues with:
- The exact vulnerable function/line in the code
- A proof-of-concept (PoC) in Solidity or Python using `web3.py`/`ethers.js`
- Exact reproduction steps on a testnet fork

**Tools:**
```bash
# Slither — static analysis for Solidity
slither ./contracts/

# Mythril — symbolic execution
myth analyze ./contract.sol

# Foundry — testing framework
forge test -vvv    # run PoC tests

# Hardhat — local fork for testing
npx hardhat test --network hardhat
```

**Quick wins to look for on any DeFi contract:**
- `transfer()` vs `call()` — `transfer` has 2300 gas limit, `call` doesn't
- No reentrancy guard on functions that send ETH
- `tx.origin` used for auth (should be `msg.sender`)
- Owner functions callable by anyone
- Hardcoded addresses that could be changed via proxy
- Missing `require` checks on return values

---

*Related: [[VRT-Overview]] | [[Business-Logic-Flaws]]*

*Sources: Bugcrowd VRT v1.18, OpenZeppelin Security, Secureum, Trail of Bits*
