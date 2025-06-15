### 🔧 **Static Analysis Tools**  
*(Scan code without execution)*  

#### 1. **[Slither](https://github.com/crytic/slither)**  
- **Functionality**:  
  - Detects 100+ vulnerabilities (reentrancy, uninitialized storage, etc.).  
  - Generates inheritance graphs and CFGs.  
- **Usage**:  
  ```bash
  slither . --detect all  # Full analysis
  slither Contract.sol --triage-mode  # Interactive findings
  ```

#### 2. **[Mythril](https://github.com/ConsenSys/mythril)**  
- **Functionality**:  
  - Symbolic execution to find security flaws.  
  - Detects reentrancy, integer overflows, and unprotected SELFDESTRUCT.  
- **Usage**:  
  ```bash
  myth analyze Contract.sol --solc-json remappings.json
  ```

#### 3. **[Semgrep for Solidity](https://semgrep.dev/)**  
- **Functionality**:  
  - Pattern-matching for custom rules (e.g., `tx.origin`, `delegatecall`).  
- **Usage**:  
  ```bash
  semgrep --config p/solidity Contract.sol
  ```

---

### 🧪 **Dynamic Analysis Tools**  
*(Test contracts during execution)*  

#### 4. **[Foundry](https://github.com/foundry-rs/foundry) (Forge + Cast)**  
- **Functionality**:  
  - Fuzz testing (property-based testing).  
  - Invariant testing (e.g., "totalSupply never decreases").  
- **Usage**:  
  ```solidity
  // In a test file
  function test_WithdrawFuzz(uint256 amount) public {
      vm.assume(amount <= userBalance);
      withdraw(amount);
      assert(balances[user] == userBalance - amount);
  }
  ```
  Run with:  
  ```bash
  forge test --match-test test_WithdrawFuzz -vvv
  ```

#### 5. **[Echidna](https://github.com/crytic/echidna)**  
- **Functionality**:  
  - Property-based fuzzer for smart contracts.  
  - Detects reentrancy, assertion violations, and more.  
- **Usage**:  
  ```bash
  echidna-test Contract.sol --config config.yaml
  ```

#### 6. **[Hardhat](https://hardhat.org/) + Plugins**  
- **Functionality**:  
  - Network forking and console.log debugging.  
  - Integrates with Slither and Mythril.  
- **Usage**:  
  ```bash
  npx hardhat test
  npx hardhat coverage
  ```

---

### 🔍 **Linting & Formatting**  

#### 7. **[Solhint](https://github.com/protofire/solhint)**  
- **Functionality**:  
  - Linter for style and security best practices.  
- **Usage**:  
  ```bash
  solhint Contract.sol
  ```

#### 8. **[Prettier Plugin for Solidity](https://github.com/prettier-solidity/prettier-plugin-solidity)**  
- **Functionality**:  
  - Auto-formatting for consistent code style.  
- **Usage**:  
  ```bash
  prettier --write Contract.sol
  ```

---

### 🌐 **DeFi-Specific Tools**  

#### 9. **[Certora Prover](https://www.certora.com/)**  
- **Functionality**:  
  - Formal verification for complex invariants (e.g., "no inflation bugs").  
- **Usage**:  
  ```bash
  certoraRun Contract.sol --verify Contract:Spec.spec
  ```

#### 10. **[Diligence Fuzzing](https://consensys.net/diligence/fuzzing/)**  
- **Functionality**:  
  - Cloud-based fuzzing for DeFi protocols.  

---

### 🛠️ **Utility Tools**  

#### 11. **[4naly3er](https://github.com/Picodes/4naly3er)**  
- **Functionality**:  
  - Gas optimizer and security analyzer.  
- **Usage**:  
  ```bash
  4naly3er Contract.sol
  ```

#### 12. **[Surya](https://github.com/ConsenSys/surya)**  
- **Functionality**:  
  - Generates call graphs and inheritance diagrams.  
- **Usage**:  
  ```bash
  surya graph Contract.sol | dot -Tpng > graph.png
  ```

---

### 📜 **Comprehensive Suites**  

#### 13. **[Brownie](https://github.com/eth-brownie/brownie)**  
- **Functionality**:  
  - Python-based framework with built-in vulnerability checks.  
- **Usage**:  
  ```bash
  brownie test --coverage
  ```

#### 14. **[Scaffold-Eth](https://github.com/scaffold-eth/scaffold-eth)**  
- **Functionality**:  
  - Rapid prototyping with security checks.  

---

### 🎯 **Pro Tips for Automation**  
1. **CI/CD Integration**:  
   ```yaml
   # GitHub Actions example
   - name: Run Slither
     run: slither . --detect all --json slither-report.json
   ```
2. **Custom Scripts**:  
   ```bash
   # Grep for critical patterns
   grep -rE "delegatecall|selfdestruct|unchecked" --include="*.sol"
   ```
3. **Combine Tools**:  
   ```bash
   slither . && myth analyze . && forge test
   ```

---

### 📚 **Further Learning**  
- **[Solidity Patterns](https://github.com/fravoll/solidity-patterns)**  
- **[Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/)**
