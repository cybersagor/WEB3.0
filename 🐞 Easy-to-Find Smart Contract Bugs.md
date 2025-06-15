## 🐞 Easy-to-Find Smart Contract Bugs

### 1. **Reentrancy**

**Impact**: Allows attacker to drain funds by calling back into the contract before the first function call finishes.

**How to spot in `.sol` code**:

```solidity
function withdraw() public {
    uint amount = balances[msg.sender];
    (bool sent, ) = msg.sender.call{value: amount}("");  // ⚠️ External call before state change
    require(sent, "Failed to send Ether");
    balances[msg.sender] = 0;  // Should come before external call!
}
```

**How to fix**:

```solidity
balances[msg.sender] = 0;
(bool sent, ) = msg.sender.call{value: amount}("");
```

**Tools to detect**:

* Manual grep for `call`, `transfer`, `send`
* Static analyzers like [Mythril](https://github.com/ConsenSys/mythril)

---

### 2. **Uninitialized Storage Variables**

**Impact**: Can overwrite important storage variables if not declared properly.

**How to spot**:

```solidity
contract Example {
    struct User {
        uint balance;
    }

    User user;

    function setBalance(uint _balance) public {
        User storage u;
        u.balance = _balance;  // ⚠️ Uninitialized storage pointer
    }
}
```

**Fix**:

```solidity
User storage u = user;
```

---

### 3. **Tx.origin Authentication**

**Impact**: Allows phishing-like attacks because `tx.origin` includes the original sender (could be a contract).

**Bad code**:

```solidity
function adminFunction() public {
    require(tx.origin == owner);  // ⚠️ Vulnerable to phishing
    // perform admin action
}
```

**Fix**:

```solidity
require(msg.sender == owner);  // Only direct caller is allowed
```

---

### 4. **Integer Overflows / Underflows** (mostly fixed in >=0.8.0)

**Impact**: Attacker causes math operations to wrap around.

**Old vulnerable code**:

```solidity
uint8 x = 255;
x += 1;  // ⚠️ Overflow to 0
```

**Fix**: Use Solidity 0.8.0+ (automatic checks) or use `SafeMath` in older versions.

---

### 5. **Unrestricted Access to `selfdestruct()`**

**Impact**: Anyone can kill the contract.

**Bad code**:

```solidity
function destroy() public {
    selfdestruct(payable(msg.sender));  // ⚠️ Anyone can call this
}
```

**Fix**:

```solidity
function destroy() public onlyOwner {
    selfdestruct(payable(owner));
}
```

---

### 6. **Hardcoded Owner Addresses**

**Impact**: Backdoors or poor upgradeability.

**Bad practice**:

```solidity
address owner = 0x123...;  // ⚠️ Hardcoded address
```

**Fix**:
Use constructor to set it:

```solidity
constructor() {
    owner = msg.sender;
}
```

---

### 7. **Missing Input Validation**

**Impact**: Can pass 0 or huge values to logic that assumes otherwise.

**Bad code**:

```solidity
function burn(uint amount) public {
    balance[msg.sender] -= amount;  // ⚠️ No check if balance >= amount
}
```

**Fix**:

```solidity
require(balance[msg.sender] >= amount, "Not enough balance");
```

---

### 🔎 How to Practice Finding These Bugs

* Open-source contracts on [Etherscan](https://etherscan.io/)
* Use `Ctrl+F` / grep to search for risky functions like `call`, `tx.origin`, `selfdestruct`, `delegatecall`
* Run [Mythril](https://github.com/ConsenSys/mythril) or [Slither](https://github.com/crytic/slither) on `.sol` code
* Try solving [Ethernaut](https://ethernaut.openzeppelin.com/) levels that relate to each bug
