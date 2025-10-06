# Part 1: Core Architecture and Philosophy - Mina Protocol, o1js, and Zeko Labs

> **AI Agent Guidance**: This document provides foundational knowledge about the Mina ecosystem. Use this information to explain architectural concepts to users and guide technology stack decisions.

## Overview of the Complete Ecosystem

### The Three-Layer Architecture

**Stack Overview for AI Agents:**

- **Development Layer**: o1js (TypeScript SDK for creating zk-SNARK circuits)
- **Base Protocol**: Mina L1 (22KB blockchain with recursive proofs)
- **Performance Layer**: Zeko L2 (High throughput while maintaining compatibility)

```
┌─────────────────────────────────┐
│          Zeko L2                │  ← High throughput, instant finality
│     (Enhanced Performance)      │     Compatible with all Mina tooling
├─────────────────────────────────┤
│          Mina L1                │  ← 22KB blockchain, zk-recursive proofs
│      (Base Security Layer)      │     True decentralization
├─────────────────────────────────┤
│           o1js                  │  ← TypeScript → zk-SNARK circuits
│    (Development Framework)      │     Constraint system programming
└─────────────────────────────────┘
```

## Mina Protocol: Fundamental Architecture

### Core Philosophy and Unique Advantages

**Mina is fundamentally different from all other blockchains:**

1. **Constant Size Blockchain**: 22KB forever through recursive zk-SNARKs
2. **Off-chain Execution Model**: zkApps execute client-side, generate proofs verified on-chain
3. **Privacy by Design**: Zero-knowledge execution with selective disclosure
4. **Scalability via Recursion**: Infinite recursion capability unique among blockchains

### zkApps: Zero Knowledge Applications

**Definition**: zkApps are zero knowledge applications built on Mina Protocol using zk-SNARKs that execute off-chain and prove their correctness on-chain.

**Key Architectural Advantages:**

- **Scalability**: Fixed 22KB blockchain size through recursive proofs
- **Privacy**: Zero-knowledge execution with optional privacy
- **Cost Efficiency**: Constant transaction costs vs. variable gas fees
- **Developer Experience**: TypeScript-based development using familiar tools

**Execution Model Comparison:**

| Aspect        | Ethereum                  | Mina zkApps                          |
| ------------- | ------------------------- | ------------------------------------ |
| **Language**  | Solidity                  | TypeScript (o1js)                    |
| **Execution** | On every node             | Client-side with proof verification  |
| **Costs**     | Variable gas fees         | Constant small fees                  |
| **Storage**   | All state on-chain        | Flexible on/off-chain storage        |
| **Scaling**   | Limited by node execution | Exponential via recursive proofs     |
| **Privacy**   | Public by default         | Private inputs, selective disclosure |
| **Consensus** | ~700GB blockchain         | 22KB recursive proof                 |

### Proof System Architecture

**Kimchi + Pickles System:**

- **Kimchi**: Step proof system (Rust implementation)
- **Pickles**: Wrap proof system enabling recursion (OCaml implementation)
- **Infinite Recursion**: Only blockchain supporting unbounded recursive proof composition

**Recursive Proof Capabilities:**

```mermaid
graph TD
    A[Base Proof] --> B[Proof of Proof 1]
    A --> C[Proof of Proof 2]
    B --> D[Combined Proof]
    C --> D
    D --> E[Higher-level Proof]
    E --> F[Blockchain State Proof]
```

### Network Architecture and Consensus

**Consensus Mechanism:**

- **Ouroboros Proof-of-Stake**: Energy-efficient consensus
- **SNARK Workers**: Specialized nodes for proof generation
- **Block Producers**: Nodes that create and propose blocks
- **Archive Nodes**: Full historical data storage

**Network Topology:**

- **Mainnet**: Production L1 with full decentralization
- **Devnet/Berkeley**: Testing network with test MINA tokens
- **Local Blockchain**: Development environment for testing

### Protocol State Anatomy

A Mina block bundles the entire protocol state and a proof of its validity. Each transition carries a previous-state hash and a body containing the blockchain state, consensus state, and consensus constants. The blockchain state commits to the staged ledger hash, genesis ledger hash, ledger proof statement, and timestamp, while the consensus state tracks global slot, epoch data, total currency, and ancestor density metrics (`docs2/docs/mina-protocol/whats-in-a-block.mdx:1`). Because the protocol state hash chains these fields, any tampering breaks the recursive proof supplied with the block.

**Key protocol state fields to keep in mind:**
- *Staged ledger hash* combines the pending ledger, scan state, and pending coinbase stack to describe the next ledger snapshot.
- *Ledger proof statement* encapsulates the snarked ledger hash, fee excess, pending coinbase stack, and local state commitment used by the proof system to attest to all included transactions.
- *Consensus constants* such as `k`, `delta`, `slots_per_window`, and `slot_duration_ms` govern finality depth and the admissible network delay window.

### Scan State, SNARK Workers, and Throughput

Mina decouples transaction execution from proof generation by maintaining a scan state: a forest of full binary trees whose leaves are base jobs and whose internal nodes are merge jobs (`docs2/docs/mina-protocol/scan-state.mdx:1`). Block producers add transactions to the scan state and must include an equivalent amount of completed proofs, which can be purchased from the snarketplace. This design guarantees a single ledger proof emerges per block while allowing transaction throughput to scale logarithmically with available SNARK work rather than block time. Important constants include:
- `transaction_capacity_log_2`: sets the theoretical upper bound on transactions per block (`2^{log_2}` slots).
- `work_delay`: enforces a minimum number of blocks before referencing newly enqueued jobs, giving SNARK workers time to respond.
- `max_number_of_trees` / `max_number_of_proofs`: derived values that bound concurrent SNARK jobs and guarantee only one ledger proof is emitted per block.

In steady state the scan state emits one proof per block; when proof supply lags, block producers fall back to lower transaction counts until the forest catches up, preserving determinism.

### Consensus Mechanics in Practice

Mina implements a succinct adaptation of Ouroboros Praos. Validators sample slots using a VRF threshold that is proportional to stake recorded in the epoch ledger. Instead of materializing historical ledgers, Mina nodes snapshot only the account records and Merkle paths needed for their own keys; the SNARK proves that the VRF output is beneath the correct stake-weighted threshold (`docs2/docs/mina-protocol/proof-of-stake.mdx:1`). Finality parameters (`delta`, `k`) guarantee that adversaries cannot extend forks beyond the checkpoint window before proofs reveal the inconsistency.

Slot winners must also satisfy the delta transition chain proof (ensuring the block was produced inside the allowed network latency window) and include SNARK work matching the staged ledger diff, so honest block producers cannot be starved by withholding proofs.

### Account Model, Preconditions, and Action History

Every Mina account update carries explicit preconditions over account state, network state, and global slots. These preconditions are enforced during validation—not while the prover constructs the transaction—so any zkApp that fetches state with `getAndRequireEquals()` is effectively pinning its proof to that specific ledger snapshot. Use inequality checks (`requireBetween`, `requireGreaterThan`) when collaborating with other updates in the same block to avoid unnecessary replays.

Mina stores an `events` commitment and an `actionState` commitment per account. The action state maintains the last five commitments (current plus four historical values) so zkApps can safely process concurrent actions dispatched across multiple transactions, forming the foundation of the reducer patterns highlighted later. Archive nodes expose the full action history while the base ledger retains only the rolling commitments.

## o1js Framework: The Core Development Layer

### Framework Architecture and Philosophy

**o1js is NOT JavaScript**: It's a TypeScript SDK with bindings to Rust and OCaml that creates constraint systems by executing TypeScript code.

```mermaid
graph TD
    A["o1js SDK (TypeScript)"] -- "Compile Program" --> B["Bindings Layer"]
    B --> C["Intermediate Representation of Constraints (OCaml)"]
    C --> D1["Pickles: Wrap Proof System (OCaml)"]
    C --> D2["Kimchi: Step Proof System (Rust)"]
    A -- "Execute Program" --> E["Dry-run Execution (TypeScript)"]
    E --> F{"Any Issues?"}
    F -- "No" --> G["Execute in Bindings Layer"]
    F -- "Yes" --> H["Throw Error / Debug"]
```

### Two-Phase Execution Model

**Critical Concept**: o1js operates in two distinct phases that developers must understand:

1. **Compile Time (Circuit Structure Determination)**:

   - Uses regular JavaScript/TypeScript execution
   - Determines the structure of the constraint system
   - Optimizations and conditional circuit inclusion happen here

2. **Prove Time (Fixed Circuit Execution)**:
   - Circuit structure is fixed and cannot change
   - Values flow through predetermined constraint paths
   - All conditional logic must use provable operations

**Example of Two-Phase Design:**

```typescript
// COMPILE TIME: Circuit structure determination
if (keyLength > 64) {
  // Add SHA-256 constraints to circuit
  keyHash = sha256Hash(keyBytes);
} else {
  // Add padding-only constraints to circuit
  keyHash = padWithZeros(keyBytes);
}

// PROVE TIME: Value selection within fixed circuit
const result = Provable.if(condition, valueA, valueB);
```

### Constraint System Programming Model

**Fundamental Principle**: Every operation in o1js must be expressible as polynomial constraints over finite fields.

**Key Constraints:**

- **No Dynamic Programming**: Loop bounds and conditional paths must be static at compile time
- **No Traditional Branching**: Must use `Provable.if()` for conditional values
- **Fixed Circuit Structure**: Same constraints must be generated on every execution
- **Size Limitations**: Larger circuits increase proof generation time exponentially

**Valid vs Invalid Patterns:**

```typescript
// ❌ INVALID - Traditional conditionals don't work in provable code
if (condition) {
  x.assertEquals(y);
}

// ✅ VALID - Use Provable.if for conditional values
const result = Provable.if(condition, valueA, valueB);

// ❌ INVALID - Dynamic loops aren't supported
for (let i = 0; i < dynamicLength; i++) {
  // This breaks circuit consistency
}

// ✅ VALID - Fixed-size arrays and loops
const MyArray = Provable.Array(Field, 10);
for (let i = 0; i < 10; i++) {
  // Always executes exactly 10 times
}

// ❌ INVALID - Side effects in conditional paths
Provable.if(
  condition,
  () => {
    counter.increment();
    return valueA;
  }, // Both paths execute!
  () => {
    counter.increment();
    return valueB;
  }
);

// ✅ VALID - Pure value selection
const value = Provable.if(condition, valueA, valueB);
value.assertEquals(expectedValue);
```

## Zeko Labs: L2 Performance Layer

### What is Zeko?

Zeko is a Layer 2 ecosystem specifically designed for Mina's zero-knowledge applications (zkApps) that provides:

- **Fully Mina-equivalent application layer**
- **Improved throughput and quick confirmation times**
- **Seamless compatibility with existing Mina tooling**
- **Modular architecture for customization**

### L1/L2 Integration Architecture

**Nested Ledger Model**: Zeko operates as a nested Mina ledger contained within a zkApp's account state on L1.

**Core Components:**

- **Outer Account (L1)**: Tracks the ledger hash of the inner ledger and manages withdrawals
- **Inner Account (L2)**: Mirrors the outer account and manages deposits
- **Special Bridge Account**: Connects L1 and L2 with a unique public key

### Performance Characteristics Comparison

| Metric               | Mina L1             | Zeko L2                          |
| -------------------- | ------------------- | -------------------------------- |
| **Finality**         | 3-5 minutes         | ~10 seconds                      |
| **Throughput**       | 24 zkApp tx/block   | No practical limit               |
| **Cost**             | ~0.1 MINA           | Fraction of L1 cost              |
| **Decentralization** | Full                | Sequencer-based                  |
| **Security**         | zk-recursive proofs | Inherits from L1                 |
| **Account Updates**  | ~7 per transaction  | No limit (centralized sequencer) |

### Key Architectural Components

#### 1. Sequencer

- **Role**: Acts as the "conductor" of the Zeko ecosystem
- **Functions**:
  - Transaction collection and state application
  - Zero-knowledge proof verification
  - Batch processing for efficient L1 settlement
  - L1 bridge communication via smart contracts
- **Performance**: Sub-second transaction processing
- **API**: Exposes GraphQL API (subset of L1 GraphQL + L1 API for actions/events)

#### 2. Modular Data Availability Layer

- **Current Implementation**: Fork of EVM chain (Ethermint) with instant finality consensus
- **Modularity Benefits**:
  - Flexibility to choose different DA solutions
  - Customization for specific application needs
  - Future integration with solutions like Celestia
- **Security**: Prevents data withholding attacks and ensures transaction data availability

#### 3. Circuit Architecture

- **Outer Circuit**: For L1 zkApp operations
- **Inner Circuit**: For L2 transfer handling
- **Transaction Wrapper Circuit**: For wrapping transaction SNARKs
- **Action State Extension Circuit**: For proving action state extensions
- **Helper Token Owner Circuit**: For tracking processed transfers

### Bridge Operations and Transfer Mechanism

**Deposit Process (L1 → L2):**

1. Actions submitted to outer account with MINA
2. Recipients finalize deposits by claiming from L1 account
3. Transfer state managed via token-based approach
4. Action state tracking maintains processed vs. pending transfers

**Withdrawal Process (L2 → L1):**

1. Reverse process with similar action-based mechanism
2. Users prove transfer hasn't been processed before
3. Async API for creating transfer requests
4. Polling mechanism for proved account updates

### Development Experience

**Seamless Integration**: Existing Mina tools, libraries, and wallets work out-of-the-box with Zeko.

**Network Configuration Example:**

```typescript
// Same codebase works on both layers!
class MyDapp extends SmartContract {
  @method async execute() {
    // This code runs identically on L1 and L2
  }
}

// L1 Configuration
const minaL1 = Mina.Network({
  networkId: "mainnet",
  mina: "https://api.minascan.io/node/mainnet/v1/graphql",
});

// L2 Configuration
const zekoL2 = Mina.Network({
  networkId: "zeko",
  mina: "https://devnet.zeko.io/graphql",
  archive: "https://devnet.zeko.io/graphql",
});
```

## Mina Transaction Model and Account Updates

**Critical AI Agent Knowledge**: Understanding Mina's unique transaction model is essential for building any zkApp.

### **AccountUpdate: The Transaction Building Block**

Every Mina transaction consists of one or more **AccountUpdates** - atomic state changes to accounts:

```typescript
// Basic transaction structure
const tx = await Mina.transaction(senderPublicKey, () => {
  // Each contract method call creates AccountUpdates
  contract.myMethod(args);

  // You can also create manual AccountUpdates
  AccountUpdate.fundNewAccount(senderPublicKey); // Pay for new account creation
});

// Prove and send
await tx.prove();
await tx.send();
```

### **Key Transaction Concepts**

```typescript
import { Mina, PrivateKey, PublicKey, AccountUpdate } from "o1js";

// 1. Network Connection (Essential First Step)
const Local = Mina.LocalBlockchain({ proofsEnabled: true });
Mina.setActiveInstance(Local);

// Or connect to live network
const network = Mina.Network({
  mina: "https://api.minascan.io/node/devnet/v1/graphql",
  archive: "https://api.minascan.io/archive/devnet/v1/graphql",
});
Mina.setActiveInstance(network);

// 2. Key Management
const deployerKey = PrivateKey.random();
const deployerAccount = deployerKey.toPublicKey();

// Fund account (for local blockchain)
const localAccounts = Local.testAccounts;
const fundingAccount = localAccounts[0].privateKey;

// 3. Contract Deployment Pattern
const contractKey = PrivateKey.random();
const contractAddress = contractKey.toPublicKey();
const contract = new MyContract(contractAddress);

const deployTx = await Mina.transaction(deployerAccount, () => {
  AccountUpdate.fundNewAccount(deployerAccount); // Pay for account creation
  contract.deploy({ zkappKey: contractKey });
});

await deployTx.prove();
await deployTx.sign([deployerKey, contractKey]).send();

// 4. Contract Interaction Pattern
const interactionTx = await Mina.transaction(deployerAccount, () => {
  contract.myMethod(arg1, arg2);
});

await interactionTx.prove();
await interactionTx.sign([deployerKey]).send();
```

### **Account Update Permissions and Security**

```typescript
class SecureContract extends SmartContract {
  init() {
    super.init();

    // Set account permissions for security
    this.account.permissions.set({
      ...Permissions.default(),
      editState: Permissions.proofOrSignature(),
      send: Permissions.proofOrSignature(),
      receive: Permissions.none(),
      incrementNonce: Permissions.proofOrSignature(),
      setDelegate: Permissions.impossible(),
      setPermissions: Permissions.impossible(),
      setVerificationKey: Permissions.impossible(),
      setZkappUri: Permissions.impossible(),
      setTokenSymbol: Permissions.impossible(),
      setVotingFor: Permissions.impossible(),
    });
  }
}
```

### **Network Configuration Patterns**

```typescript
// AI Agent Helper: Network configuration for different environments
export class NetworkManager {
  static setupLocal(): void {
    const Local = Mina.LocalBlockchain({
      proofsEnabled: true,
      enforceTransactionLimits: false, // For testing
    });
    Mina.setActiveInstance(Local);
  }

  static setupDevnet(): void {
    const network = Mina.Network({
      mina: "https://api.minascan.io/node/devnet/v1/graphql",
      archive: "https://api.minascan.io/archive/devnet/v1/graphql",
    });
    Mina.setActiveInstance(network);
  }

  static setupMainnet(): void {
    const network = Mina.Network({
      mina: "https://api.minascan.io/node/mainnet/v1/graphql",
      archive: "https://api.minascan.io/archive/mainnet/v1/graphql",
    });
    Mina.setActiveInstance(network);
  }

  static async waitForTransaction(txHash: string): Promise<void> {
    console.log("Waiting for transaction confirmation...");

    for (let attempt = 0; attempt < 50; attempt++) {
      try {
        const status = await Mina.getTransactionStatus(txHash);
        if (status === "INCLUDED") {
          console.log("Transaction confirmed!");
          return;
        }
      } catch (error) {
        // Transaction not yet included
      }

      await new Promise((resolve) => setTimeout(resolve, 10000)); // Wait 10 seconds
    }

    throw new Error("Transaction confirmation timeout");
  }
}
```

## Strategic Positioning and Use Cases

**AI Agent Decision Framework**: Use this guidance to recommend the appropriate layer based on user requirements.

### When to Use Each Layer

#### **Mina L1 - Recommend for:**

- **Security-first applications**: Maximum decentralization and security
- **Long-term value storage**: DeFi protocols, treasury management
- **Cross-ecosystem bridges**: Interoperability with other blockchains
- **Governance systems**: Protocol upgrades and DAOs
- **Compliance-critical applications**: Regulatory environments requiring maximum security
- **Privacy-preserving applications**: Zero-knowledge proofs with highest security guarantees

**Trade-offs**: 3-5 minute finality, limited throughput

#### **Zeko L2 - Recommend for:**

- **High-frequency applications**: DEX trading, gaming, real-time interactions
- **User experience focused apps**: Applications needing instant feedback
- **Cost-sensitive use cases**: Micro-transactions, frequent interactions
- **Development and prototyping**: Faster iteration cycles
- **Consumer applications**: Social apps, content platforms
- **Enterprise applications**: Business workflows requiring high throughput

**Trade-offs**: Reduced decentralization, sequencer dependency

### Market Opportunities and Applications

**Privacy-preserving DeFi:**

- Hidden transaction amounts using zk-proofs
- Private order books and MEV protection
- Compliance-friendly privacy solutions

**Scalable Gaming:**

- Private game state with zk-proof verification
- Fast state updates via Zeko L2
- Provable randomness and fair gameplay

**Identity Solutions:**

- zk-proof based authentication
- Selective disclosure of credentials
- Privacy-preserving KYC/AML

**Compliance Tools:**

- Private regulatory reporting
- Audit-friendly transaction logs
- Zero-knowledge compliance verification

### Ecosystem Competitive Advantages

**Unique Technical Capabilities:**

1. **Only blockchain with infinite recursion** (Kimchi + Pickles)
2. **Client-side proof generation** (no network congestion)
3. **Constant-size blockchain** (22KB forever)
4. **Privacy by design** (zk-native)
5. **TypeScript development** (familiar to web developers)
6. **Unified L1/L2 development experience**

**Ecosystem Positioning:**

- **Mina L1**: The "Bitcoin" of privacy (security, decentralization)
- **Zeko L2**: The "Lightning" of privacy (speed, scalability)
- **o1js**: The "React" of zk-development (developer experience)

## Version Information and Evolution

### o1js Evolution

- **Current Version**: o1js 2.10.0 (released September 27, 2025)
- **Previous Name**: SnarkyJS (49 versions, 43,141 downloads)
- **Recent Features (2.7.0 → 2.10.0)**:
  - **2.10.0**: `Core` namespace for low-level bindings, improved `RuntimeTable` API, Transaction.fromJSON string support, VK regression fix, cache artifact fix
  - **2.9.0**: `RuntimeTable` class API, `ForeignField.Unsafe.fromField`, performance improvements
  - **2.8.0**: `ZkFunction` API (via `Experimental.ZkFunction`), improved sourcemaps
  - **2.7.0**: Lazy mode for prover index computation, `IndexedMerkleMap` promoted to public API
  - DynamicArray reverse functionality and enhanced constraint system analysis tools

### Active Development Status

- **Monthly Updates**: Regular o1js releases with new features
- **Community**: Active Discord channels for developers
- **Documentation**: Comprehensive tutorials and API reference
- **Tooling**: Mature CLI and development workflow
- **Third-party Audits**: Veridise conducted external audit (Q3 2024)

### Browser Compatibility Requirements

- **WebAssembly**: Required for performance
- **SharedArrayBuffer**: Needs specific CORS headers:
  ```
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
  ```
- **Modern Browser Support**: Latest Chrome, Firefox, Safari, Edge

## Advanced: Core Namespace for Protocol-Level Access (2.10.0+)

**AI Agent Note**: The `Core` namespace is for advanced users only. Guide developers to use high-level APIs first. Only recommend Core for protocol development, custom cryptography, or deep integrations.

### Introduction to Core Namespace

As of o1js **2.10.0**, the library exposes internal protocol constants, hashes, and prefixes via the `Core` namespace. This provides low-level access to Mina Protocol bindings for developers building:

- Custom cryptographic primitives
- Protocol-level tools and infrastructure
- Deep blockchain integrations
- Advanced optimization scenarios

**⚠️ WARNING**: The Core namespace is a low-level API subject to change. Use high-level o1js APIs whenever possible.

### Core Namespace Usage

```typescript
import { Core } from 'o1js';

// Access to low-level bindings
// Example: Protocol constants and hashes
// (Actual API depends on internal bindings structure)

// ⚠️ Advanced use only - APIs may change between versions
```

### When to Use Core Namespace

**✅ Appropriate Use Cases:**

- Building custom proof system tooling
- Implementing novel cryptographic constructions
- Protocol research and development
- Performance-critical optimizations requiring low-level control
- Custom Mina Protocol integrations

**❌ Avoid Core for:**

- Standard zkApp development (use SmartContract, ZkProgram)
- Token contracts (use TokenContract or Silvana)
- Common cryptographic operations (use Poseidon, Signature, etc.)
- General application logic

### Best Practices

```typescript
// ✅ DO: Use high-level APIs first
import { Poseidon, Signature, Field } from 'o1js';

// Only drop to Core when necessary
import { Core } from 'o1js';

// ❌ DON'T: Use Core for standard operations
// This makes code brittle and harder to maintain
```

**Key Guidelines:**

1. **Prefer high-level APIs**: SmartContract, ZkProgram, Provable types
2. **Document Core usage**: Explain why low-level access is needed
3. **Test thoroughly**: Core APIs have fewer safety guarantees
4. **Monitor updates**: Core namespace may change between versions
5. **Seek community input**: Discuss Core usage on Discord before deployment

### Migration Strategy

If you find yourself needing Core namespace features:

1. **First**: Check if existing o1js APIs can solve your problem
2. **Second**: Ask in the o1js Discord #general or #zkapp-developers
3. **Third**: Consider opening a feature request for high-level API
4. **Last Resort**: Use Core with thorough testing and documentation

This completes Part 1 covering the core architecture and philosophy. The foundation is set for diving into the technical implementation details in the subsequent parts.
