### **✅ 1. Missing `onlyOwner` on Critical Functions**  
#### 📖 **Description**  
Functions like `mint()`, `withdraw()`, or `pause()` should be restricted to the contract owner. If unprotected, attackers can drain funds or manipulate the contract.  

#### 🐞 **Vulnerable Code**  
```solidity
function withdrawAll() public {
    payable(msg.sender).transfer(address(this).balance);
}
```
**🔴 Problem:** Anyone can call this and steal all ETH.  

#### 🔐 **Fixed Code**  
```solidity
address owner;

function withdrawAll() public {
    require(msg.sender == owner, "Not owner!");
    payable(msg.sender).transfer(address(this).balance);
}
```
**🟢 Detection:**  
- Search for critical functions (`withdraw`, `mint`, `setAdmin`) without `onlyOwner`.  
- Use Slither: `slither --detect unprotected-upgrade .`  

---

### **✅ 2. Using `tx.origin` for Authorization**  
#### 📖 **Description**  
`tx.origin` returns the **original transaction sender**, not the immediate caller. Attackers can trick users into calling malicious contracts that exploit this.  

#### 🐞 **Vulnerable Code**  
```solidity
function withdraw() public {
    require(tx.origin == owner, "Not owner!");
    payable(msg.sender).transfer(100 ether);
}
```
**🔴 Problem:** A phishing contract can call this on behalf of the victim.  

#### 🔐 **Fixed Code**  
```solidity
function withdraw() public {
    require(msg.sender == owner, "Not owner!");
    payable(msg.sender).transfer(100 ether);
}
```
**🟢 Detection:**  
- `grep -r "tx.origin" *.sol`  
- Mythril: `myth analyze --truffle`  

---

### **✅ 3. Reentrancy Attack (Checks-Effects-Interactions Violation)**  
#### 📖 **Description**  
If a contract sends ETH before updating state, an attacker can recursively re-enter the function and drain funds.  

#### 🐞 **Vulnerable Code**  
```solidity
function withdraw() public {
    uint bal = balances[msg.sender];
    (bool sent, ) = msg.sender.call{value: bal}("");
    require(sent, "Failed");
    balances[msg.sender] = 0; // Too late!
}
```
**🔴 Problem:** Attacker re-enters `withdraw()` before balance is zeroed.  

#### 🔐 **Fixed Code**  
```solidity
function withdraw() public {
    uint bal = balances[msg.sender];
    balances[msg.sender] = 0; // Update first!
    (bool sent, ) = msg.sender.call{value: bal}("");
    require(sent, "Failed");
}
```
**🟢 Detection:**  
- Slither: `slither --detect reentrancy .`  
- Look for `call.value()` before state changes.  

---

### **✅ 4. Uninitialized Storage Pointer**  
#### 📖 **Description**  
Uninitialized `storage` pointers in Solidity can overwrite critical data.  

#### 🐞 **Vulnerable Code**  
```solidity
contract Wallet {
    address[] wallets;

    function addWallet(address wallet) public {
        wallets.push(wallet);
    }

    function clearWallets() public {
        address[] storage walletList; // Points to slot 0!
        walletList = wallets; // Overwrites data.
    }
}
```
**🔴 Problem:** `walletList` corrupts storage.  

#### 🔐 **Fixed Code**  
```solidity
function clearWallets() public {
    delete wallets; // Proper cleanup
}
```
**🟢 Detection:**  
- Slither: `slither --detect uninitialized-storage`  

---

### **✅ 5. Integer Overflow/Underflow (Pre-Solidity 0.8.0)**  
#### 📖 **Description**  
Before Solidity 0.8.0, arithmetic operations could silently overflow (e.g., `0 - 1 = 2^256 - 1`).  

#### 🐞 **Vulnerable Code**  
```solidity
function withdraw(uint amount) public {
    balances[msg.sender] -= amount; // Underflow if amount > balance
}
```
**🔴 Problem:** Attacker can underflow to get a huge balance.  

#### 🔐 **Fixed Code**  
```solidity
// Solidity 0.8+ auto-checks
function withdraw(uint amount) public {
    balances[msg.sender] -= amount;
}
```
**🟢 Detection:**  
- Check for `unchecked { ... }` blocks.  
- Use Slither: `slither --detect overflow`  

---

### **✅ 6. Public `selfdestruct()` Function**  
#### 📖 **Description**  
If `selfdestruct()` is callable by anyone, the contract can be killed.  

#### 🐞 **Vulnerable Code**  
```solidity
function kill() public {
    selfdestruct(payable(msg.sender));
}
```
**🔴 Problem:** Anyone can destroy the contract.  

#### 🔐 **Fixed Code**  
```solidity
function kill() public onlyOwner {
    selfdestruct(payable(owner));
}
```
**🟢 Detection:**  
- `grep -r "selfdestruct" *.sol`  

---

### **✅ 7. Unchecked `call()` Return Value**  
#### 📖 **Description**  
Ignoring the return value of low-level calls can hide failed transfers.  

#### 🐞 **Vulnerable Code**  
```solidity
function sendETH(address to, uint amount) public {
    to.call{value: amount}(""); // No success check
}
```
**🔴 Problem:** Failed transfers go unnoticed.  

#### 🔐 **Fixed Code**  
```solidity
function sendETH(address to, uint amount) public {
    (bool sent, ) = to.call{value: amount}("");
    require(sent, "Transfer failed");
}
```
**🟢 Detection:**  
- Look for unchecked `call()`, `send()`, `transfer()`.  

---

### **✅ 8. Hardcoded Owner Address**  
#### 📖 **Description**  
Hardcoding an owner address prevents contract recovery if keys are lost.  

#### 🐞 **Vulnerable Code**  
```solidity
address owner = 0x123...;
```
**🔴 Problem:** No way to change ownership.  

#### 🔐 **Fixed Code**  
```solidity
address owner;
constructor() {
    owner = msg.sender;
}
```
**🟢 Detection:**  
- Search for hardcoded `address` assignments.  

---

### **✅ 9. Missing `receive()` or `fallback()`**  
#### 📖 **Description**  
Contracts without a `receive()` function may lock ETH sent via `transfer()`.  

#### 🐞 **Vulnerable Code**  
```solidity
contract NoReceive {} // No fallback
```
**🔴 Problem:** ETH transfers fail.  

#### 🔐 **Fixed Code**  
```solidity
receive() external payable {}
```
**🟢 Detection:**  
- Check if contract handles ETH.  

---

### **✅ 10. Delegatecall to Untrusted Contracts**  
#### 📖 **Description**  
`delegatecall` executes code from another contract in the caller’s context, risking storage hijacking.  

#### 🐞 **Vulnerable Code**  
```solidity
function execute(address _contract, bytes calldata data) public {
    _contract.delegatecall(data); // Dangerous!
}
```
**🔴 Problem:** Attacker can modify storage.  

#### 🔐 **Fixed Code**  
```solidity
// Avoid delegatecall unless absolutely necessary
```
**🟢 Detection:**  
- `grep -r "delegatecall" *.sol`  

---

### **✅ 11. Unprotected Ether Withdrawal (Missing Access Control)**  
#### 📖 **Description**  
If a contract allows arbitrary addresses to withdraw ETH (e.g., via `selfdestruct` or direct transfers), funds can be stolen.  

#### 🐞 **Vulnerable Code**  
```solidity
function withdraw() public {
    payable(msg.sender).transfer(address(this).balance);
}
```
**🔴 Problem:** Anyone can drain the contract.  

#### 🔐 **Fixed Code**  
```solidity
address owner;
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```
**🟢 Detection:**  
- Search for `transfer()`/`send()` calls without access checks.  

---

### **✅ 12. ERC20 Return Values Not Checked**  
#### 📖 **Description**  
Some ERC20 tokens (like USDT) don’t revert on failure but return `false`. Ignoring this can break logic.  

#### 🐞 **Vulnerable Code**  
```solidity
IERC20(token).transfer(user, amount); // No return check
```
**🔴 Problem:** Failed transfers go unnoticed.  

#### 🔐 **Fixed Code**  
```solidity
bool success = IERC20(token).transfer(user, amount);
require(success, "Transfer failed");
```
**🟢 Detection:**  
- `grep -r "\.transfer(" *.sol` (missing `require`)  

---

### **✅ 13. Frontrunning with `block.timestamp`**  
#### 📖 **Description**  
Using `block.timestamp` for randomness or deadlines is manipulable by miners.  

#### 🐞 **Vulnerable Code**  
```solidity
function claimReward() public {
    require(block.timestamp > deadline, "Too early");
    _mint(msg.sender, reward);
}
```
**🔴 Problem:** Miners can adjust timestamps (±30 sec).  

#### 🔐 **Fixed Code**  
```solidity
function claimReward() public {
    require(block.number > deadlineBlock, "Too early");
    _mint(msg.sender, reward);
}
```
**🟢 Detection:**  
- `grep -r "block.timestamp" *.sol`  

---

### **✅ 14. Unbounded Loops (Gas Limit DoS)**  
#### 📖 **Description**  
Loops over dynamic arrays can run out of gas if the array grows too large.  

#### 🐞 **Vulnerable Code**  
```solidity
address[] holders;
function payDividends() public {
    for (uint i = 0; i < holders.length; i++) {
        payable(holders[i]).transfer(1 ether);
    }
}
```
**🔴 Problem:** Attackers can spam `holders` to block payments.  

#### 🔐 **Fixed Code**  
```solidity
// Use pull-over-push (let users withdraw individually)
```
**🟢 Detection:**  
- Slither: `slither --detect loops`  

---

### **✅ 15. Signature Replay Attacks**  
#### 📖 **Description**  
Signed messages can be reused (replayed) on other chains or contracts.  

#### 🐞 **Vulnerable Code**  
```solidity
function claimAirdrop(bytes memory sig) public {
    bytes32 hash = keccak256(abi.encodePacked(msg.sender));
    address signer = ECDSA.recover(hash, sig);
    require(signer == owner, "Invalid sig");
    _mint(msg.sender, 1000 tokens);
}
```
**🔴 Problem:** Same `sig` can be reused.  

#### 🔐 **Fixed Code**  
```solidity
bytes32 hash = keccak256(abi.encodePacked(msg.sender, nonce));
```
**🟢 Detection:**  
- Check for missing nonce/chainID in signatures.  

---

### **✅ 16. Storage Collision in Proxies**  
#### 📖 **Description**  
Upgradeable proxies can corrupt storage if logic/state layouts mismatch.  

#### 🐞 **Vulnerable Code**  
```solidity
// Proxy storage slot: keccak("implementation")
address public implementation; 
// Logic contract uses same slot for unrelated data!
uint public implementation; 
```
**🔴 Problem:** Storage clash breaks the proxy.  

#### 🔐 **Fixed Code**  
```solidity
// Use unstructured storage pattern
bytes32 private constant _IMPLEMENTATION_SLOT = 
    keccak256("some.unique.identifier");
```
**🟢 Detection:**  
- Slither: `slither --detect storage-collision`  

---

### **✅ 17. Missing Event Emission for Critical Actions**  
#### 📖 **Description**  
Without events, off-chain monitors can’t track admin actions (e.g., ownership changes).  

#### 🐞 **Vulnerable Code**  
```solidity
function setOwner(address newOwner) public onlyOwner {
    owner = newOwner; // No event
}
```
**🔴 Problem:** Stealth ownership takeover.  

#### 🔐 **Fixed Code**  
```solidity
event OwnerChanged(address old, address new);
function setOwner(address newOwner) public onlyOwner {
    emit OwnerChanged(owner, newOwner);
    owner = newOwner;
}
```
**🟢 Detection:**  
- `grep -r "owner =" *.sol` (missing events)  

---

### **✅ 18. Incorrect ERC721 `safeTransferFrom` Usage**  
#### 📖 **Description**  
Using `transferFrom` instead of `safeTransferFrom` can send NFTs to non-receiver contracts.  

#### 🐞 **Vulnerable Code**  
```solidity
function transferNFT(address to, uint id) public {
    IERC721(nft).transferFrom(msg.sender, to, id);
}
```
**🔴 Problem:** NFT locked if `to` can’t handle it.  

#### 🔐 **Fixed Code**  
```solidity
IERC721(nft).safeTransferFrom(msg.sender, to, id);
```
**🟢 Detection:**  
- `grep -r "transferFrom(" *.sol` (missing `safe`)  

---

### **✅ 19. Floating Pragma**  
#### 📖 **Description**  
`^0.8.0` allows any version ≥0.8.0, which may include bugs.  

#### 🐞 **Vulnerable Code**  
```solidity
pragma solidity ^0.8.0;
```
**🔴 Problem:** Unintended compiler bugs.  

#### 🔐 **Fixed Code**  
```solidity
pragma solidity 0.8.17; // Lock to a specific version
```
**🟢 Detection:**  
- `grep -r "pragma solidity ^" *.sol`  

---

### **✅ 20. Uninitialized Local Variables**  
#### 📖 **Description**  
Unset `memory` variables default to zero, causing unintended behavior.  

#### 🐞 **Vulnerable Code**  
```solidity
function sum(uint[] calldata arr) public pure returns (uint) {
    uint total; // Uninitialized (defaults to 0)
    for (uint i = 0; i < arr.length; i++) {
        total += arr[i];
    }
    return total;
}
```
**🔴 Problem:** Works but unclear intent.  

#### 🔐 **Fixed Code**  
```solidity
uint total = 0; // Explicit initialization
```
**🟢 Detection:**  
- Slither: `slither --detect uninitialized-local`  

---

### 🐞 **21. ERC777 Reentrancy via `tokensToSend` Hook**
#### 📖 **Why Dangerous**  
ERC777's `tokensToSend`/`tokensReceived` hooks allow reentrancy during transfers, similar to ETH reentrancy.

#### 🔥 Vulnerable Code
```solidity
contract Vulnerable {
    mapping(address => uint) balances;
    
    function withdraw(uint amount) public {
        require(balances[msg.sender] >= amount);
        IERC777(token).send(msg.sender, amount, ""); // Triggers hook
        balances[msg.sender] -= amount; // Too late!
    }
}
```
**Attack:** Malicious contract re-enters `withdraw()` before balance update.

#### 🛡️ Fixed Code
```solidity
function withdraw(uint amount) public {
    balances[msg.sender] -= amount; // CEI pattern
    IERC777(token).send(msg.sender, amount, "");
}
```
**Detection:**  
- Slither: `slither --detect erc777-reeentrancy`
- Look for ERC777 transfers with external calls

---

### 🐞 **22. Delegatecall Storage Collision**
#### 📖 **Why Dangerous**  
`delegatecall` preserves caller's storage layout. Mismatched contracts corrupt storage.

#### 🔥 Vulnerable Code
```solidity
contract Proxy {
    address implementation;
    
    fallback() external {
        implementation.delegatecall(msg.data);
    }
}

contract Logic {
    address public owner; // Storage slot 0 vs Proxy's implementation
}
```
**Attack:** `Logic` overwrites Proxy's `implementation` variable.

#### 🛡️ Fixed Code
```solidity
// Proxy uses unstructured storage
bytes32 constant IMPL_SLOT = keccak256("impl");
function upgrade(address newImpl) internal {
    bytes32 slot = IMPL_SLOT;
    assembly { sstore(slot, newImpl) }
}
```
**Detection:**  
- Check storage layouts in upgradeable contracts
- MythX: `detect storage-collision`

---

### 🐞 **23. Oracle Manipulation (Spot Price Slippage)**
#### 📖 **Why Dangerous**  
Using single DEX spot prices allows price manipulation via flash loans.

#### 🔥 Vulnerable Code
```solidity
function getPrice() public view returns (uint) {
    (uint reserve0, uint reserve1,) = UNI_PAIR.getReserves();
    return reserve0 / reserve1; // Easy to manipulate
}
```
**Attack:** Flash loan to skew reserves before price query.

#### 🛡️ Fixed Code
```solidity
// Use TWAP (Time-Weighted Average Price)
interface IUniswapV2Oracle {
    consult(address token, uint amountIn) external view returns (uint);
}
```
**Detection:**  
- Look for `getReserves()` without TWAP
- Chainlink: Check for `latestAnswer()` without heartbeat

---

### 🐞 **24. Griefing with Failed Transfers**
#### 📖 **Why Dangerous**  
Some tokens (e.g., USDT) revert on failure, others return `false`. Mixed handling breaks contracts.

#### 🔥 Vulnerable Code
```solidity
function distribute(address[] calldata users) public {
    for (uint i = 0; i < users.length; i++) {
        IERC20(token).transfer(users[i], amount); // May revert
    }
}
```
**Problem:** One failed transfer blocks entire distribution.

#### 🛡️ Fixed Code
```solidity
for (uint i = 0; i < users.length; i++) {
    bool success = IERC20(token).transfer(users[i], amount);
    if (!success) emit TransferFailed(users[i]);
}
```
**Detection:**  
- `grep -r "\.transfer(" --include="*.sol"`

---

### 🐞 **25. Unchecked `call()` Return Data**
#### 📖 **Why Dangerous**  
Ignoring return data from low-level calls hides failures.

#### 🔥 Vulnerable Code
```solidity
function callContract(address target) public {
    target.call(""); // No return check
}
```
**Attack:** Callee fails silently.

#### 🛡️ Fixed Code
```solidity
(bool success, bytes memory data) = target.call("");
require(success, "Call failed");
```
**Detection:**  
- Slither: `slither --detect unchecked-call`

---

### 🐞 **26. Constructor Shadowing (Pre-0.4.22)**
#### 📖 **Why Dangerous**  
Pre-Solidity 0.4.22, constructors were defined as functions matching the contract name. Typos created public functions.

#### 🔥 Vulnerable Code
```solidity
contract Token {
    function Token() public { // Constructor 
        initialize();
    }
    
    function initialize() public {} // Accidental public init
}
```
**Attack:** Anyone can call `initialize()`.

#### 🛡️ Fixed Code
```solidity
constructor() public {
    initialize();
}
```
**Detection:**  
- Check legacy contracts for function-named constructors

---

### 🐞 **27. Signature Malleability**
#### 📖 **Why Dangerous**  
ECDSA signatures can be modified without invalidating them.

#### 🔥 Vulnerable Code
```solidity
function verify(bytes32 hash, uint8 v, bytes32 r, bytes32 s) public {
    require(ecrecover(hash, v, r, s) == owner);
}
```
**Attack:** Signature can be altered while remaining valid.

#### 🛡️ Fixed Code
```solidity
require(uint256(s) <= 0x7FFFFFFFF...);
```
**Detection:**  
- Look for missing `s` checks in `ecrecover`

---

### 🐞 **28. Unprotected Initializer**
#### 📖 **Why Dangerous**  
Upgradeable contracts with publicly callable `initialize()` functions.

#### 🔥 Vulnerable Code
```solidity
function initialize() public {
    owner = msg.sender; // No protection
}
```
**Attack:** Anyone can become owner.

#### 🛡️ Fixed Code
```solidity
function initialize() public initializer {
    __Ownable_init();
}
```
**Detection:**  
- `grep -r "function initialize(" --include="*.sol"`

---

### 🐞 **29. Gas Limit in Loops**
#### 📖 **Why Dangerous**  
Loops may hit gas limits with large iterations.

#### 🔥 Vulnerable Code
```solidity
for (uint i = 0; i < array.length; i++) {
    process(array[i]); // May run out of gas
}
```
**Fix:** Use pagination or pull payments.

#### 🛡️ Fixed Code
```solidity
uint constant MAX_BATCH = 50;
for (uint i = 0; i < Math.min(array.length, MAX_BATCH); i++) {
    process(array[i]);
}
```
**Detection:**  
- Slither: `slither --detect loops`

---

### 🐞 **30. Unbounded Array Growth**
#### 📖 **Why Dangerous**  
Push operations without limits can brick contracts.

#### 🔥 Vulnerable Code
```solidity
address[] public allUsers;
function register() public {
    allUsers.push(msg.sender); // No limit
}
```
**Attack:** Spam registrations to make reads fail.

#### 🛡️ Fixed Code
```solidity
uint constant MAX_USERS = 10_000;
function register() public {
    require(allUsers.length < MAX_USERS);
    allUsers.push(msg.sender);
}
```
**Detection:**  
- Look for an unbounded `array.push()`

---

### 🐞 **31. Flash Loan Price Oracle Manipulation**
#### 📖 **Why Dangerous**  
Attackers borrow massive assets to skew DEX prices, tricking protocols into incorrect valuations.

#### 🔥 Vulnerable Code
```solidity
function getETHPrice() public view returns (uint) {
    (uint reserve0, uint reserve1,) = UNI_ETH_USDC.getReserves();
    return reserve1 / reserve0; // USDC per ETH (manipulable)
}
```
**Real-World Exploit:** [bZx (2020)](https://rekt.news/bzx-rekt/) lost $350k.

#### 🛡️ Fixed Code
```solidity
// Use Chainlink or TWAP
function getETHPrice() public view returns (uint) {
    return chainlinkFeed.latestAnswer(); // Decentralized oracle
}
```
**Detection:**  
- Look for `getReserves()` without time-weighted checks
- Slither: `slither --detect price-oracle`

---

### 🐞 **32. ERC4626 Inflation Attack**
#### 📖 **Why Dangerous**  
First depositor can inflate share prices by donating assets.

#### 🔥 Vulnerable Code
```solidity
function deposit(uint assets) public returns (uint shares) {
    shares = assets * totalSupply / totalAssets(); // Before transfer
    _mint(msg.sender, shares);
    asset.transferFrom(msg.sender, address(this), assets);
}
```
**Attack:** Donate 1M tokens → shares become worthless.

#### 🛡️ Fixed Code
```solidity
shares = totalSupply == 0 ? assets : assets * totalSupply / totalAssets();
```
**Detection:**  
- Check ERC4626 vaults for donation protection

---

### 🐞 **33. Cross-Chain Replay Attacks**
#### 📖 **Why Dangerous**  
Signatures valid on multiple chains enable replay attacks.

#### 🔥 Vulnerable Code
```solidity
function bridgeWithdraw(bytes memory sig) public {
    bytes32 hash = keccak256(abi.encodePacked(msg.sender, amount));
    require(!usedHashes[hash], "Already spent");
    usedHashes[hash] = true;
    // No chainID in hash
}
```
**Attack:** Reuse sig on another EVM chain.

#### 🛡️ Fixed Code
```solidity
bytes32 hash = keccak256(abi.encodePacked(block.chainid, msg.sender, amount));
```
**Detection:**  
- `grep -r "abi.encodePacked(" --include="*.sol" | grep -v "chainid"`

---

### 🐞 **34. Storage Collision in Proxy Patterns**
#### 📖 **Why Dangerous**  
Upgraded contracts with mismatched storage layouts corrupt data.

#### 🔥 Vulnerable Code
```solidity
// Proxy storage slot 0: implementation address
// Upgraded contract slot 0: accidentally overlaps
uint public totalSupply; // Now overwrites proxy's implementation
```
**Real-World Exploit:** [Parity Wallet (2017)](https://www.parity.io/security-alert-2/)

#### 🛡️ Fixed Code
```solidity
// Use EIP-1967 storage slots
bytes32 private constant _IMPLEMENTATION_SLOT = 
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```
**Detection:**  
- Slither: `slither --detect storage-collision`

---

### 🐞 **35. Griefing with Gas Limits**
#### 📖 **Why Dangerous**  
Loops or unbounded operations can be DoS'd via gas limits.

#### 🔥 Vulnerable Code
```solidity
function distribute(address[] calldata recipients) public {
    for (uint i; i < recipients.length; i++) {
        payable(recipients[i]).transfer(1 ether); // May fail
    }
}
```
**Attack:** One recipient reverts → blocks entire batch.

#### 🛡️ Fixed Code
```solidity
// Use pull payments (recipients withdraw individually)
mapping(address => uint) public balances;
function withdraw() public {
    uint amount = balances[msg.sender];
    balances[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```
**Detection:**  
- Look for loops with external calls

---

### 🐞 **36. ERC20 Permit Frontrunning**
#### 📖 **Why Dangerous**  
Unprotected `permit()` allows signature reuse.

#### 🔥 Vulnerable Code
```solidity
function permit(address owner, address spender, uint value, uint deadline, bytes memory sig) public {
    // No nonce check
    bytes32 digest = keccak256(abi.encodePacked(owner, spender, value, deadline));
    address signer = ecrecover(digest, sig);
    require(signer == owner, "Invalid sig");
    _approve(owner, spender, value);
}
```
**Attack:** Reuse the same permit multiple times.

#### 🛡️ Fixed Code
```solidity
bytes32 digest = keccak256(abi.encodePacked(
    "\x19\x01", 
    DOMAIN_SEPARATOR, 
    keccak256(abi.encode(
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"),
        owner, spender, value, nonces[owner]++, deadline
    ))
));
```
**Detection:**  
- Check for missing nonces in `permit()`

---

### 🐞 **37. Unchecked `create2` Preimage**
#### 📖 **Why Dangerous**  
Attackers can frontrun `create2` deployments with malicious contracts.

#### 🔥 Vulnerable Code
```solidity
address deployed = address(new Counter{salt: salt}());
```
**Attack:** Precompute salt to deploy attacker's contract.

#### 🛡️ Fixed Code
```solidity
bytes32 salt = keccak256(abi.encode(msg.sender, nonce));
address deployed = address(new Counter{salt: salt}());
```
**Detection:**  
- Review all `create2` salt generation logic

---

### 🐞 **38. `selfdestruct` in Logic Contracts**
#### 📖 **Why Dangerous**  
Proxy logic contracts with `selfdestruct` can kill the entire system.

#### 🔥 Vulnerable Code
```solidity
function kill() public onlyOwner {
    selfdestruct(payable(owner)); // In logic contract
}
```
**Attack:** Proxy becomes useless after destruction.

#### 🛡️ Fixed Code
```solidity
// Remove selfdestruct from upgradeable contracts
```
**Detection:**  
- `grep -r "selfdestruct" --include="*.sol"`

---

### 🐞 **39. Signature Replay in Meta-Transactions**
#### 📖 **Why Dangerous**  
Missing chainID or nonce allows replay across networks.

#### 🔥 Vulnerable Code
```solidity
function executeMetaTx(
    address user,
    bytes memory callData,
    bytes memory sig
) public {
    bytes32 hash = keccak256(abi.encodePacked(user, callData));
    address signer = ecrecover(hash, sig);
    require(signer == user, "Invalid sig");
    (bool success,) = address(this).call(callData);
    require(success);
}
```
**Fix:** Include `block.chainid` and nonce in hash.

#### 🛡️ Fixed Code
```solidity
bytes32 hash = keccak256(abi.encodePacked(
    block.chainid, address(this), nonces[user]++, callData
));
```
**Detection:**  
- Check all meta-transaction signature schemes

---

### 🐞 **40. Unbounded `exp` Operations**
#### 📖 **Why Dangerous**  
Exponential functions can overflow and brick contracts.

#### 🔥 Vulnerable Code
```solidity
function calculateInterest(uint principal, uint rate, uint time) public pure returns (uint) {
    return principal * (rate ** time); // Overflows easily
}
```
**Attack:** Pass large `time` to overflow.

#### 🛡️ Fixed Code
```solidity
// Use fixed-point math libraries
return principal * (1e18 + rate).pow(time) / 1e18;
```
**Detection:**  
- Look for `**` operator in financial logic

---

### 🐞 **41. Arbitrary `delegatecall` in Proxies**
#### 📖 **Why Dangerous**  
Proxies allowing arbitrary `delegatecall` enable full storage hijacking.

#### 🔥 Vulnerable Code
```solidity
fallback() external {
    address impl = implementation;
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```
**Attack:** Call `selfdestruct` via malicious payload.

#### 🛡️ Fixed Code
```solidity
// Whitelist allowed function selectors
bytes4 sig = bytes4(msg.data[0:4]);
require(sig == 0x12345678 || sig == 0x87654321, "Not allowed");
```
**Detection:**  
- Check proxy contracts for unrestricted `delegatecall`

---

### 🐞 **42. ERC721 Reentrancy via `onERC721Received`**
#### 📖 **Why Dangerous**  
NFT transfer hooks can reenter vulnerable contracts.

#### 🔥 Vulnerable Code
```solidity
function buyNFT(uint id) public {
    require(!listed[id], "Already sold");
    nft.safeTransferFrom(address(this), msg.sender, id);
    listed[id] = true; // Too late!
}
```
**Attack:** Malicious NFT contract reenters before state update.

#### 🛡️ Fixed Code
```solidity
function buyNFT(uint id) public {
    listed[id] = true; // Update first
    nft.safeTransferFrom(address(this), msg.sender, id);
}
```
**Detection:**  
- Slither: `slither --detect erc721-reeentrancy`

---

### 🐞 **43. Uninitialized Proxy Contracts**
#### 📖 **Why Dangerous**  
Proxies without initialization can be hijacked.

#### 🔥 Vulnerable Code
```solidity
contract Proxy {
    address public implementation; // No initializer
    fallback() external { /* delegatecall */ }
}
```
**Attack:** Attacker sets malicious implementation.

#### 🛡️ Fixed Code
```solidity
constructor(address _impl) {
    implementation = _impl;
}
```
**Detection:**  
- Check proxy constructors

---

### 🐞 **44. `block.number` as Deadline**
#### 📖 **Why Dangerous**  
Miners can manipulate block numbers within small ranges.

#### 🔥 Vulnerable Code
```solidity
function lockTokens(uint blocks) public {
    unlockBlock = block.number + blocks;
}
```
**Attack:** Miners can stall blocks.

#### 🛡️ Fixed Code
```solidity
function lockTokens(uint time) public {
    unlockTime = block.timestamp + time;
}
```
**Detection:**  
- `grep -r "block.number" --include="*.sol"`

---

### 🐞 **45. Missing `receive()` in Payment Splitters**
#### 📖 **Why Dangerous**  
Contracts splitting ETH need `receive()` to handle direct transfers.

#### 🔥 Vulnerable Code
```solidity
contract Splitter {
    function split() public {
        // No receive() function
        payable(a).transfer(address(this).balance / 2);
        payable(b).transfer(address(this).balance / 2);
    }
}
```
**Problem:** ETH sent directly gets stuck.

#### 🛡️ Fixed Code
```solidity
receive() external payable {}
```
**Detection:**  
- `grep -r "transfer(" --include="*.sol" | grep -L "receive"`

---

### 🐞 **46. Unchecked `ecrecover` Return**
#### 📖 **Why Dangerous**  
`ecrecover` returns `address(0)` for invalid sigs.

#### 🔥 Vulnerable Code
```solidity
address signer = ecrecover(hash, v, r, s);
require(signer == owner); // May pass if signer=0 and owner=0
```
**Attack:** Spoof zero-address signatures.

#### 🛡️ Fixed Code
```solidity
require(signer != address(0) && signer == owner, "Invalid sig");
```
**Detection:**  
- `grep -r "ecrecover(" --include="*.sol"`

---

### 🐞 **47. `transfer()` Gas Stipend Limitation**
#### 📖 **Why Dangerous**  
`transfer()` limits gas to 2300, which may fail with gas-cost increases.

#### 🔥 Vulnerable Code
```solidity
payable(user).transfer(amount); // May fail post-EIP-1559
```
**Fix:** Use `call` with gas adjustment.

#### 🛡️ Fixed Code
```solidity
(bool success,) = payable(user).call{value: amount, gas: 30_000}("");
require(success);
```
**Detection:**  
- `grep -r "\.transfer(" --include="*.sol"`

---

### 🐞 **48. Unprotected Ether Withdrawal in Libraries**
#### 📖 **Why Dangerous**  
Library functions transferring ETH can be exploited if unprotected.

#### 🔥 Vulnerable Code
```solidity
library PaymentLib {
    function pay(address to, uint amount) public {
        payable(to).transfer(amount);
    }
}
```
**Attack:** Anyone can call library functions.

#### 🛡️ Fixed Code
```solidity
// Libraries shouldn't handle ETH directly
```
**Detection:**  
- Review all library functions

---

### 🐞 **49. Incorrect Interface Detection**
#### 📖 **Why Dangerous**  
Assuming interface compliance via `extcodesize` can be bypassed.

#### 🔥 Vulnerable Code
```solidity
function isContract(address addr) internal view returns (bool) {
    uint size;
    assembly { size := extcodesize(addr) }
    return size > 0;
}
```
**Attack:** Contracts in construction return `extcodesize=0`.

#### 🛡️ Fixed Code
```solidity
// Use try/catch for calls
```
**Detection:**  
- `grep -r "extcodesize" --include="*.sol"`

---

### 🐞 **50. Hardcoded Gas Limits in Cross-Chain Calls**
#### 📖 **Why Dangerous**  
Fixed gas limits may fail under network congestion.

#### 🔥 Vulnerable Code
```solidity
// Arbitrum L1→L2 message
Inbox.depositEth{gas: 200_000}(...);
```
**Problem:** Gas costs may increase.

#### 🛡️ Fixed Code
```solidity
// Use gas estimation or slippage
uint gas = gasleft() - 50_000; // Buffer
```
**Detection:**  
- Review all cross-chain messaging

---
