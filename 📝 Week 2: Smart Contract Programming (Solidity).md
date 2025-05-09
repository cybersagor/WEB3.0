### **📝 Week 2: Smart Contract Programming (Solidity)**

This week, you’ll dive into the heart of Web3 – **Solidity**, the primary language for writing smart contracts on the Ethereum blockchain. You'll start by learning the basics, then progress to more advanced features, building a strong foundation for finding and exploiting Web3 vulnerabilities.

---

#### **🔍 Day 1-2: Getting Started with Solidity**

* 📌 **Introduction to Smart Contracts**

  * What are smart contracts?
  * How smart contracts differ from traditional programs
  * Real-world use cases (DeFi, NFTs, DAOs, Tokenomics)

* 📌 **Basic Solidity Syntax**

  * Data types (uint, int, bool, address, string, bytes)
  * Variables (state, local, global)
  * Visibility (public, private, internal, external)
  * Functions, function modifiers, and constructors

* 📌 **Reading:**

  * [Solidity Documentation (Basics)](https://docs.soliditylang.org/en/latest/)
  * [Mastering Ethereum (Chapters 6-8)](https://github.com/ethereumbook/ethereumbook)

* 📌 **Practical:**

  * Create your first "Hello World" smart contract in Remix.
  * Deploy it on the Goerli testnet.

---

#### **🔍 Day 3-4: Understanding Smart Contract Structure**

* 📌 **Contract Structure and Inheritance**

  * Contract definition and constructors
  * Inheritance and multiple inheritance
  * Using interfaces and abstract contracts
* 📌 **Events and Logging**

  * How events work in Solidity
  * Using **emit** to trigger events
  * Retrieving past events on Etherscan
* 📌 **Reading:**

  * [Solidity by Example](https://solidity-by-example.org/)
  * [Solidity Events and Logging](https://docs.soliditylang.org/en/latest/contracts.html#events)
* 📌 **Practical:**

  * Build a simple ERC-20 token with minting and burning capabilities.
  * Use events to log transactions.

---

#### **🔍 Day 5-6: Control Structures and Functions**

* 📌 **Control Flow in Solidity**

  * if-else, for loops, while loops, and break/continue
  * Error handling and custom error messages (revert, require, assert)
* 📌 **Function Modifiers and Gas Optimization**

  * Create reusable function modifiers
  * Understand gas costs and optimization strategies
* 📌 **Reading:**

  * [Solidity Control Structures](https://docs.soliditylang.org/en/latest/control-structures.html)
  * [Gas Optimization in Solidity](https://ethereum.stackexchange.com/questions/34381/how-to-optimize-gas-in-solidity-smart-contracts)
* 📌 **Practical:**

  * Build a basic wallet contract with deposit, withdraw, and balance check functions.
  * Optimize the contract for lower gas fees.

---

#### **🔍 Day 7: Access Control and Security Basics**

* 📌 **Access Control in Smart Contracts**

  * Role-based access control (RBAC)
  * Using **msg.sender** and **tx.origin** safely
* 📌 **Security Basics**

  * Common pitfalls (reentrancy, unchecked transfers, integer overflow/underflow)
  * Use OpenZeppelin's libraries for secure contract development
* 📌 **Reading:**

  * [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/)
  * [Solidity Security Patterns](https://fravoll.github.io/solidity-patterns/)
* 📌 **Practical:**

  * Build a simple **Ownable** contract with role-based permissions.
  * Implement basic security checks and access control.

---

#### **🛠️ Essential Tools for Week 2**

* **Remix IDE** - Online IDE for writing and deploying smart contracts
* **MetaMask** - For interacting with deployed contracts
* **Ganache** - Personal Ethereum blockchain for local testing
* **OpenZeppelin Contracts** - Standard secure smart contract libraries
* **Hardhat** - Ethereum development environment and testing framework

---

#### **💡 Key Takeaways by the End of Week 2**

* You should be comfortable writing basic smart contracts.
* You should understand the structure of a smart contract, including functions, modifiers, and events.
* You should know how to deploy a contract to a testnet and interact with it using MetaMask.
