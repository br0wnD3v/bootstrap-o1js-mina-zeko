# Part 4: Advanced Features and Recursion - ZkPrograms, Infinite Recursion, and Complex Proof Composition

> **AI Agent Guidance**: This document covers advanced o1js patterns including ZkPrograms and recursion. Use this for complex applications requiring proof composition and blockchain compression. The deprecated lookup table migration section has been removed as it was confusing and unused.

## ZkPrograms: Advanced Proof Systems
### Proof-System Building Blocks

Mina’s recursion stack splits into **step circuits** (Kimchi) and a **wrap circuit** (Pickles). Step circuits encode your business logic; the wrap circuit verifies the previous recursive proof and enforces feature flags such as `app_state` length, events, actions, and zkApp URI constraints (`o1js/src/lib/proof-system/zkprogram.ts:1`). Each recursive layer stores its verification key hash inside the proof so the wrap circuit can ensure you are verifying the correct program. Because the wrap circuit itself is fixed, upgrades that change the number of public inputs or proof indices require new verification keys and, if `allowUpdates=false`, a redeploy.

`Proof`s contain three core components: the public input vector, the previous proof commitments, and the deferred values. Deferred values like `plonk.alpha` or `bulletproof_challenges` are rechecked in the wrap circuit to guarantee honesty from the prover. Understanding these internals helps explain error messages such as `Invalid proof: scalar challenge did not match`—they indicate the wrap circuit rejected the deferred values of a recursive call.

### Feature Flags and Recursion Safety

The proof system uses feature flags to prevent incompatible circuits from being composed. Flags cover capabilities such as transaction slots, events, actions, and dynamic proofs. When you enable a feature (for example emitting actions), the compiler toggles the corresponding flag; any recursive proof that omits the flag cannot verify a proof that requires it. This prevents downgrade attacks where an adversary recursively verifies a proof under weaker rules than intended.

You can inspect the active feature set by calling `Proof.getFeatureFlags()` on a generated proof (`o1js/src/lib/proof-system/proof.ts:1`).

### Aggregation Patterns

Designing recursive systems typically follows one of three shapes:
1. **Linear recursion** (state machine): each proof feeds into the next, carrying over aggregate state such as a running hash or counter.
2. **Tree recursion** (batching): proofs verify siblings first and then combine via a parent proof, allowing logarithmic depth for large batches.
3. **Hybrid**: merge pairs in a tree and periodically checkpoint into a linear accumulator.

Tree recursion is ideal for reducing many independent proofs (payments, rollup batches), while linear recursion shines for evolving states like rollup ledgers or sequences of verifiable computations.

### Proof Caching and Key Management

`ZkProgram.compile()` and `SmartContract.compile()` output serialized prover key, verification key, and a cache identifier. Persist them through the `Cache` interface (filesystem or custom) so subsequent runs can read the keys instead of recompiling (`o1js/src/lib/proof-system/cache.ts:1`). As you rotate verification keys, remember that existing on-chain deployments expect the hash recorded during `deploy()`. If `allowUpdates` was set to `false`, you must redeploy a new contract and migrate state manually.


**Key Concept for AI Agents**: ZkPrograms enable infinite recursion and proof composition - unique capabilities in blockchain space. Use for scalability and complex verification patterns.

### Basic ZkProgram Structure

```typescript
import { ZkProgram, Field, Struct, SelfProof, verify } from "o1js";

// Define input/output types
class ProgramInput extends Struct({
  value: Field,
  multiplier: Field,
}) {}

class ProgramOutput extends Struct({
  result: Field,
  stepCount: Field,
}) {}

// Create ZkProgram
const MathProgram = ZkProgram({
  name: "math-program",
  publicInput: ProgramInput,
  publicOutput: ProgramOutput,

  methods: {
    // Base case method
    baseCase: {
      privateInputs: [],
      async method(input: ProgramInput): Promise<ProgramOutput> {
        // Simple base computation
        const result = input.value.mul(input.multiplier);
        return new ProgramOutput({
          result,
          stepCount: Field(1),
        });
      },
    },

    // Recursive method
    recursiveStep: {
      privateInputs: [SelfProof],
      async method(
        input: ProgramInput,
        earlierProof: SelfProof<ProgramInput, ProgramOutput>
      ): Promise<ProgramOutput> {
        // Verify the earlier proof
        earlierProof.verify();

        // Access previous computation results
        const previousResult = earlierProof.publicOutput.result;
        const previousSteps = earlierProof.publicOutput.stepCount;

        // Perform incremental computation
        const newResult = previousResult.add(input.value.mul(input.multiplier));
        const newStepCount = previousSteps.add(1);

        return new ProgramOutput({
          result: newResult,
          stepCount: newStepCount,
        });
      },
    },
  },
});

// Compile the program
await MathProgram.compile();

// Generate base proof
const baseInput = new ProgramInput({ value: Field(5), multiplier: Field(2) });
const baseProof = await MathProgram.baseCase(baseInput);

// Generate recursive proof
const recursiveInput = new ProgramInput({
  value: Field(3),
  multiplier: Field(4),
});
const recursiveProof = await MathProgram.recursiveStep(
  recursiveInput,
  baseProof
);

console.log("Final result:", recursiveProof.publicOutput.result.toString());
console.log("Steps taken:", recursiveProof.publicOutput.stepCount.toString());
```

### Advanced Recursion Patterns

#### **Linear Recursion: State Machine Progression**

```typescript
// State machine for a simple counter with constraints
class CounterState extends Struct({
  value: Field,
  maxValue: Field,
  isActive: Bool,
}) {}

const CounterProgram = ZkProgram({
  name: "counter-state-machine",
  publicInput: Field, // New value to add
  publicOutput: CounterState,

  methods: {
    // Initialize counter
    init: {
      privateInputs: [Field], // maxValue
      async method(increment: Field, maxValue: Field): Promise<CounterState> {
        // Validate initial increment
        increment.assertGreaterThan(Field(0));
        increment.assertLessThanOrEqual(maxValue);

        return new CounterState({
          value: increment,
          maxValue,
          isActive: Bool(true),
        });
      },
    },

    // Increment counter recursively
    increment: {
      privateInputs: [SelfProof],
      async method(
        increment: Field,
        prevProof: SelfProof<Field, CounterState>
      ): Promise<CounterState> {
        prevProof.verify();

        const prevState = prevProof.publicOutput;

        // Validate state
        prevState.isActive.assertTrue();

        const newValue = prevState.value.add(increment);

        // Check if we exceed maximum
        const exceedsMax = newValue.greaterThan(prevState.maxValue);
        const finalValue = Provable.if(
          exceedsMax,
          prevState.maxValue,
          newValue
        );
        const stillActive = exceedsMax.not();

        return new CounterState({
          value: finalValue,
          maxValue: prevState.maxValue,
          isActive: stillActive,
        });
      },
    },

    // Reset counter
    reset: {
      privateInputs: [SelfProof],
      async method(
        newIncrement: Field,
        prevProof: SelfProof<Field, CounterState>
      ): Promise<CounterState> {
        prevProof.verify();

        const prevState = prevProof.publicOutput;

        return new CounterState({
          value: newIncrement,
          maxValue: prevState.maxValue,
          isActive: Bool(true),
        });
      },
    },
  },
});
```

#### **Tree Recursion: Parallel Computation Merging**

```typescript
// Tree-based recursive computation for parallel processing
class ComputationNode extends Struct({
  value: Field,
  depth: Field,
  nodeCount: Field,
}) {}

const TreeProgram = ZkProgram({
  name: "tree-computation",
  publicInput: Field, // Input value
  publicOutput: ComputationNode,

  methods: {
    // Leaf node computation
    leaf: {
      privateInputs: [],
      async method(value: Field): Promise<ComputationNode> {
        // Process leaf value (e.g., square it)
        const processedValue = value.mul(value);

        return new ComputationNode({
          value: processedValue,
          depth: Field(0),
          nodeCount: Field(1),
        });
      },
    },

    // Merge two subtrees
    merge: {
      privateInputs: [SelfProof, SelfProof],
      async method(
        _input: Field, // Not used in merge, but required by type system
        leftProof: SelfProof<Field, ComputationNode>,
        rightProof: SelfProof<Field, ComputationNode>
      ): Promise<ComputationNode> {
        leftProof.verify();
        rightProof.verify();

        const leftNode = leftProof.publicOutput;
        const rightNode = rightProof.publicOutput;

        // Combine values (e.g., sum them)
        const combinedValue = leftNode.value.add(rightNode.value);

        // Calculate new depth (max of children + 1)
        const maxDepth = Provable.if(
          leftNode.depth.greaterThan(rightNode.depth),
          leftNode.depth,
          rightNode.depth
        );
        const newDepth = maxDepth.add(1);

        // Count total nodes
        const totalNodes = leftNode.nodeCount.add(rightNode.nodeCount).add(1);

        return new ComputationNode({
          value: combinedValue,
          depth: newDepth,
          nodeCount: totalNodes,
        });
      },
    },
  },
});

// Example usage: Build a tree computation
async function buildTreeComputation(values: Field[]): Promise<any> {
  await TreeProgram.compile();

  // Create leaf proofs
  const leafProofs = await Promise.all(
    values.map((value) => TreeProgram.leaf(value))
  );

  // Recursively merge pairs until we have one root
  let currentLevel = leafProofs;

  while (currentLevel.length > 1) {
    const nextLevel = [];

    for (let i = 0; i < currentLevel.length; i += 2) {
      if (i + 1 < currentLevel.length) {
        // Merge pair
        const mergedProof = await TreeProgram.merge(
          Field(0), // Dummy input
          currentLevel[i],
          currentLevel[i + 1]
        );
        nextLevel.push(mergedProof);
      } else {
        // Odd number, carry forward
        nextLevel.push(currentLevel[i]);
      }
    }

    currentLevel = nextLevel;
  }

  return currentLevel[0]; // Root proof
}
```

### Blockchain Compression and Rollups

#### **Transaction Batch Compression**

```typescript
// Transaction types for rollup
class Transaction extends Struct({
  from: PublicKey,
  to: PublicKey,
  amount: UInt64,
  nonce: UInt64,
}) {}

class RollupState extends Struct({
  stateRoot: Field,
  transactionCount: UInt64,
  totalVolume: UInt64,
}) {}

class BatchProof extends Struct({
  oldState: RollupState,
  newState: RollupState,
  transactions: Provable.Array(Transaction, 10), // Fixed batch size
}) {}

const RollupProgram = ZkProgram({
  name: "transaction-rollup",
  publicInput: RollupState, // Previous state
  publicOutput: RollupState, // New state

  methods: {
    // Process single transaction
    processTransaction: {
      privateInputs: [Transaction, MerkleWitness20],
      async method(
        prevState: RollupState,
        transaction: Transaction,
        stateWitness: MerkleWitness20
      ): Promise<RollupState> {
        // Verify current state
        const currentRoot = stateWitness.calculateRoot(
          Poseidon.hash([
            ...transaction.from.toFields(),
            transaction.nonce.value,
          ])
        );
        currentRoot.assertEquals(prevState.stateRoot);

        // Validate transaction
        transaction.amount.assertGreaterThan(UInt64.zero);

        // Update state root (simplified)
        const newAccountState = Poseidon.hash([
          ...transaction.from.toFields(),
          transaction.nonce.add(1).value,
        ]);
        const newStateRoot = stateWitness.calculateRoot(newAccountState);

        return new RollupState({
          stateRoot: newStateRoot,
          transactionCount: prevState.transactionCount.add(1),
          totalVolume: prevState.totalVolume.add(transaction.amount),
        });
      },
    },

    // Process batch of transactions recursively
    processBatch: {
      privateInputs: [SelfProof, Transaction, MerkleWitness20],
      async method(
        targetState: RollupState,
        prevProof: SelfProof<RollupState, RollupState>,
        transaction: Transaction,
        stateWitness: MerkleWitness20
      ): Promise<RollupState> {
        prevProof.verify();

        // Process the additional transaction
        const intermediateState = prevProof.publicOutput;

        // Apply transaction to intermediate state
        return this.processTransaction(
          intermediateState,
          transaction,
          stateWitness
        );
      },
    },
  },
});
```

#### **Cross-Chain Communication**

```typescript
// IBC-like messaging between chains
class ChainMessage extends Struct({
  sourceChain: Field,
  destinationChain: Field,
  messageData: Field,
  sequence: UInt64,
  timestamp: UInt64,
}) {}

class ChainState extends Struct({
  chainId: Field,
  latestHeight: UInt64,
  stateCommitment: Field,
  messageSequence: UInt64,
}) {}

const CrossChainProgram = ZkProgram({
  name: "cross-chain-messaging",
  publicInput: ChainMessage,
  publicOutput: ChainState,

  methods: {
    // Initialize chain state
    initChain: {
      privateInputs: [Field], // Chain ID
      async method(message: ChainMessage, chainId: Field): Promise<ChainState> {
        return new ChainState({
          chainId,
          latestHeight: UInt64.zero,
          stateCommitment: Field(0),
          messageSequence: UInt64.zero,
        });
      },
    },

    // Send message to another chain
    sendMessage: {
      privateInputs: [SelfProof],
      async method(
        message: ChainMessage,
        chainProof: SelfProof<ChainMessage, ChainState>
      ): Promise<ChainState> {
        chainProof.verify();

        const currentState = chainProof.publicOutput;

        // Validate message
        message.sourceChain.assertEquals(currentState.chainId);
        message.sequence.assertEquals(currentState.messageSequence.add(1));

        // Update state with outgoing message
        const newCommitment = Poseidon.hash([
          currentState.stateCommitment,
          message.messageData,
          message.sequence.value,
        ]);

        return new ChainState({
          chainId: currentState.chainId,
          latestHeight: currentState.latestHeight.add(1),
          stateCommitment: newCommitment,
          messageSequence: message.sequence,
        });
      },
    },

    // Receive and verify message from another chain
    receiveMessage: {
      privateInputs: [SelfProof, SelfProof], // Local state, remote chain proof
      async method(
        message: ChainMessage,
        localProof: SelfProof<ChainMessage, ChainState>,
        remoteProof: SelfProof<ChainMessage, ChainState>
      ): Promise<ChainState> {
        localProof.verify();
        remoteProof.verify();

        const localState = localProof.publicOutput;
        const remoteState = remoteProof.publicOutput;

        // Validate message routing
        message.destinationChain.assertEquals(localState.chainId);
        message.sourceChain.assertEquals(remoteState.chainId);

        // Verify message was sent from remote chain
        const expectedCommitment = Poseidon.hash([
          remoteState.stateCommitment,
          message.messageData,
          message.sequence.value,
        ]);
        // This would need more complex verification in practice

        // Update local state with received message
        const newLocalCommitment = Poseidon.hash([
          localState.stateCommitment,
          message.messageData,
          Field(1), // Received flag
        ]);

        return new ChainState({
          chainId: localState.chainId,
          latestHeight: localState.latestHeight.add(1),
          stateCommitment: newLocalCommitment,
          messageSequence: localState.messageSequence,
        });
      },
    },
  },
});
```

## Sideloaded Verification Keys and Dynamic Verification

### Dynamic ZkProgram Verification

```typescript
// Program that can verify other programs dynamically
class VerificationInput extends Struct({
  proof: Field, // Serialized proof
  publicInput: Field,
  programId: Field,
}) {}

const UniversalVerifier = ZkProgram({
  name: "universal-verifier",
  publicInput: VerificationInput,
  publicOutput: Bool, // Verification result

  methods: {
    // Verify a proof with sideloaded verification key
    verifyProof: {
      privateInputs: [DynamicProof], // Dynamic proof type
      async method(
        input: VerificationInput,
        proof: DynamicProof
      ): Promise<Bool> {
        // Verify the proof matches the input
        const proofIsValid = proof.verify();

        // Additional validation
        proof.publicInput.assertEquals(input.publicInput);

        return proofIsValid;
      },
    },

    // Compose multiple proof verifications
    verifyMultiple: {
      privateInputs: [SelfProof, DynamicProof],
      async method(
        input: VerificationInput,
        prevProof: SelfProof<VerificationInput, Bool>,
        newProof: DynamicProof
      ): Promise<Bool> {
        prevProof.verify();

        const prevResult = prevProof.publicOutput;
        const newResult = newProof.verify();

        // All proofs must be valid
        return prevResult.and(newResult);
      },
    },
  },
});
```

### Upgradeable ZkProgram Pattern

```typescript
// Proxy pattern for ZkProgram upgradability
class ProgramRegistry extends SmartContract {
  @state(Field) currentVersion = State<Field>();
  @state(Field) implementationHash = State<Field>();

  // Store verification keys for different versions
  private versionedKeys = new Map<string, any>();

  @method async upgradeImplementation(
    newVersion: Field,
    newImplementationHash: Field,
    adminSignature: Signature
  ) {
    const admin = this.getAdmin();

    adminSignature
      .verify(admin, [newVersion, newImplementationHash])
      .assertTrue();

    this.currentVersion.set(newVersion);
    this.implementationHash.set(newImplementationHash);
  }

  @method async verifyWithCurrentVersion(
    proof: DynamicProof,
    publicInput: Field
  ): Bool {
    const version = this.currentVersion.getAndRequireEquals();

    // Load verification key for current version
    const vk = this.getVerificationKey(version);

    // Verify proof with correct VK
    return proof.verifyWithKey(vk, publicInput);
  }

  private getVerificationKey(version: Field): any {
    // Implementation specific - load VK based on version
    return this.versionedKeys.get(version.toString());
  }

  private getAdmin(): PublicKey {
    // Implementation specific
    return PublicKey.empty();
  }
}
```

## Advanced Recursion Applications

### Fibonacci Sequence with Optimization

```typescript
// Optimized Fibonacci using matrix exponentiation in ZK
class FibMatrix extends Struct({
  a: Field, // [1,1]
  b: Field, // [1,0] position values
}) {}

const FibonacciProgram = ZkProgram({
  name: "fibonacci-optimized",
  publicInput: Field, // n (which Fibonacci number to compute)
  publicOutput: Field, // F(n)

  methods: {
    // Base cases
    fibBase: {
      privateInputs: [],
      async method(n: Field): Promise<Field> {
        // Handle F(0) = 0, F(1) = 1
        const isZero = n.equals(Field(0));
        const isOne = n.equals(Field(1));

        const result = Provable.if(
          isZero,
          Field(0),
          Provable.if(isOne, Field(1), Field(0))
        );

        // Ensure n is 0 or 1
        isZero.or(isOne).assertTrue();

        return result;
      },
    },

    // Recursive matrix multiplication approach
    fibRecursive: {
      privateInputs: [SelfProof],
      async method(
        n: Field,
        prevProof: SelfProof<Field, Field>
      ): Promise<Field> {
        prevProof.verify();

        const prevN = prevProof.publicInput;
        const prevResult = prevProof.publicOutput;

        // Verify we're computing the next Fibonacci number
        n.assertEquals(prevN.add(1));

        // For this simplified example, use addition
        // In practice, you'd implement matrix exponentiation
        const nextFib = this.computeNextFib(prevN, prevResult);

        return nextFib;
      },
    },

    // Jump computation for large Fibonacci numbers
    fibJump: {
      privateInputs: [SelfProof, Field], // Previous proof, jump size
      async method(
        n: Field,
        baseProof: SelfProof<Field, Field>,
        jumpSize: Field
      ): Promise<Field> {
        baseProof.verify();

        const baseN = baseProof.publicInput;
        const baseResult = baseProof.publicOutput;

        // Verify jump relationship
        n.assertEquals(baseN.add(jumpSize));

        // Use matrix exponentiation to compute F(n) from F(baseN)
        const result = this.fibonacciJump(baseResult, jumpSize);

        return result;
      },
    },
  },

  // Helper methods would be implemented as separate functions
});

// Helper functions (outside the ZkProgram)
function computeNextFib(prevN: Field, prevResult: Field): Field {
  // Simplified - in practice you'd implement proper Fibonacci logic
  return prevResult.add(Field(1));
}

function fibonacciJump(baseResult: Field, jumpSize: Field): Field {
  // Matrix exponentiation implementation would go here
  return baseResult.mul(jumpSize); // Simplified placeholder
}
```

### State Machine with Complex Transitions

```typescript
// Complex state machine for a game or workflow
enum GameState {
  WAITING = 0,
  PLAYING = 1,
  PAUSED = 2,
  FINISHED = 3,
}

class GameStateData extends Struct({
  state: Field,
  player1Score: UInt64,
  player2Score: UInt64,
  currentPlayer: Field, // 1 or 2
  moveCount: UInt64,
  gameId: Field,
}) {}

class GameAction extends Struct({
  actionType: Field, // 1=start, 2=move, 3=pause, 4=resume, 5=end
  player: Field,
  moveData: Field,
  signature: Signature,
}) {}

const GameProgram = ZkProgram({
  name: "game-state-machine",
  publicInput: GameAction,
  publicOutput: GameStateData,

  methods: {
    // Initialize new game
    initGame: {
      privateInputs: [Field], // Game ID
      async method(action: GameAction, gameId: Field): Promise<GameStateData> {
        // Validate initialization action
        action.actionType.assertEquals(Field(1)); // Start action

        return new GameStateData({
          state: Field(GameState.WAITING),
          player1Score: UInt64.zero,
          player2Score: UInt64.zero,
          currentPlayer: Field(1),
          moveCount: UInt64.zero,
          gameId,
        });
      },
    },

    // Execute game action
    executeAction: {
      privateInputs: [SelfProof],
      async method(
        action: GameAction,
        prevProof: SelfProof<GameAction, GameStateData>
      ): Promise<GameStateData> {
        prevProof.verify();

        const prevState = prevProof.publicOutput;

        // Validate action based on current state
        const newState = this.validateAndApplyAction(prevState, action);

        return newState;
      },
    },

    // Handle complex multi-step transitions
    complexTransition: {
      privateInputs: [SelfProof, SelfProof], // Two previous states
      async method(
        action: GameAction,
        state1Proof: SelfProof<GameAction, GameStateData>,
        state2Proof: SelfProof<GameAction, GameStateData>
      ): Promise<GameStateData> {
        state1Proof.verify();
        state2Proof.verify();

        const state1 = state1Proof.publicOutput;
        const state2 = state2Proof.publicOutput;

        // Validate states are from same game
        state1.gameId.assertEquals(state2.gameId);

        // Complex transition logic
        const mergedState = this.mergeGameStates(state1, state2, action);

        return mergedState;
      },
    },
  },
});

// State transition logic (helper methods)
function validateAndApplyAction(
  currentState: GameStateData,
  action: GameAction
): GameStateData {
  const currentStateField = currentState.state;
  const actionType = action.actionType;

  // Start game transition
  const canStart = currentStateField
    .equals(Field(GameState.WAITING))
    .and(actionType.equals(Field(1)));

  // Make move transition
  const canMove = currentStateField
    .equals(Field(GameState.PLAYING))
    .and(actionType.equals(Field(2)))
    .and(action.player.equals(currentState.currentPlayer));

  // Pause transition
  const canPause = currentStateField
    .equals(Field(GameState.PLAYING))
    .and(actionType.equals(Field(3)));

  // Resume transition
  const canResume = currentStateField
    .equals(Field(GameState.PAUSED))
    .and(actionType.equals(Field(4)));

  // End game transition
  const canEnd = actionType.equals(Field(5));

  // Validate at least one transition is valid
  const isValidTransition = canStart
    .or(canMove)
    .or(canPause)
    .or(canResume)
    .or(canEnd);
  isValidTransition.assertTrue();

  // Apply state changes based on action
  const newState = Provable.if(
    canStart,
    Field(GameState.PLAYING),
    Provable.if(
      canMove,
      Field(GameState.PLAYING), // Stays in playing state
      Provable.if(
        canPause,
        Field(GameState.PAUSED),
        Provable.if(
          canResume,
          Field(GameState.PLAYING),
          Field(GameState.FINISHED) // End game
        )
      )
    )
  );

  // Update scores based on move
  const scoreUpdate = Provable.if(
    canMove.and(action.player.equals(Field(1))),
    currentState.player1Score.add(1),
    currentState.player1Score
  );

  const score2Update = Provable.if(
    canMove.and(action.player.equals(Field(2))),
    currentState.player2Score.add(1),
    currentState.player2Score
  );

  // Switch players
  const nextPlayer = Provable.if(
    canMove,
    Provable.if(
      currentState.currentPlayer.equals(Field(1)),
      Field(2),
      Field(1)
    ),
    currentState.currentPlayer
  );

  return new GameStateData({
    state: newState,
    player1Score: scoreUpdate,
    player2Score: score2Update,
    currentPlayer: nextPlayer,
    moveCount: currentState.moveCount.add(canMove.toField().mul(1)),
    gameId: currentState.gameId,
  });
}

function mergeGameStates(
  state1: GameStateData,
  state2: GameStateData,
  action: GameAction
): GameStateData {
  // Complex merging logic for parallel game states
  // This could be used for handling complex game scenarios
  // where multiple state paths need to be combined

  return new GameStateData({
    state: Field(GameState.PLAYING),
    player1Score: state1.player1Score.add(state2.player1Score),
    player2Score: state1.player2Score.add(state2.player2Score),
    currentPlayer: action.player,
    moveCount: state1.moveCount.add(state2.moveCount),
    gameId: state1.gameId,
  });
}
```

## RuntimeTable: Improved Lookup Table API (Introduced 2.9.0, Enhanced 2.10.0)

### Introduction to RuntimeTable

**AI Agent Note**: RuntimeTable provides efficient provable lookup operations for runtime-defined tables. Use this for membership proofs of (index, value) pairs with automatic batching.

**RuntimeTable** is the current API for working with runtime lookup tables in o1js. It provides an ergonomic class-based interface for defining tables whose entries are determined at circuit construction time.

**Key Features (as of 2.10.0):**

- **Automatic batching**: Groups up to 3 lookup pairs per gate automatically
- **Better readability**: Intuitive `.insert()`, `.lookup()`, `.check()` API
- **Type safety**: Full TypeScript support with compile-time checks
- **Efficient gates**: ~`ceil(#pairs / 3)` lookup gates per table

**Replaces Deprecated APIs:**

```typescript
// ❌ DEPRECATED (DO NOT USE - will be removed)
Gates.addRuntimeTableConfig(id, indices);
Gadgets.inTable(id, index, value);

// ✅ CURRENT API (o1js 2.9.0+)
const table = new RuntimeTable(id, indices);
table.insert([[index, value]]);
table.lookup(index, value);
table.check();
```

**When to Use RuntimeTable:**

- ✅ Small/medium runtime-chosen sets of (index, value) pairs
- ✅ Membership proofs with known index space
- ✅ Precomputed lookup tables (hashes, arithmetic, etc.)
- ❌ NOT for: huge tables (>65K entries), mutable data, unknown indices

### Constraints and Requirements

**CRITICAL INVARIANTS:**

```typescript
// 1. Table ID must NOT be 0 or 1 (reserved for XOR and range-check tables)
const table = new RuntimeTable(2, [0n, 1n, 2n]); // ✅ OK
const badTable = new RuntimeTable(0, [0n, 1n]); // ❌ THROWS ERROR

// 2. Indices must be UNIQUE (duplicates rejected)
const uniqueTable = new RuntimeTable(3, [10n, 20n, 30n]); // ✅ OK
const dupTable = new RuntimeTable(3, [10n, 10n, 20n]); // ❌ THROWS ERROR

// 3. Indices must be KNOWN at construction time
const knownIndices = [0n, 1n, 2n];
const dynamicTable = new RuntimeTable(4, knownIndices); // ✅ OK

// 4. ALWAYS call check() to flush pending lookups (1-2 pairs)
dynamicTable.lookup(0n, Field(123));
dynamicTable.lookup(1n, Field(456));
dynamicTable.check(); // ✅ REQUIRED - flushes 2 pending lookups

// 5. zkApps using RuntimeTable MUST compile with withRuntimeTables flag
await MyContract.compile({ withRuntimeTables: true });
```

**Size Limits:**

- Maximum table size: **2^16 (65,536) entries**
- Maximum indices must be < circuit size limit
- Each lookup gate can batch **3 pairs** maximum

---

**⚠️ COMPILATION REQUIREMENT ⚠️**

**Any contract or ZkProgram using `RuntimeTable` MUST be compiled with the `withRuntimeTables: true` flag**, otherwise you'll get runtime errors when the circuit tries to use lookup gates.

```typescript
// SmartContract compilation
await MyContract.compile({
  withRuntimeTables: true  // REQUIRED for RuntimeTable usage
});

// ZkProgram compilation
await MyProgram.compile({
  withRuntimeTables: true  // REQUIRED for RuntimeTable usage
});
```

**What happens if you forget this flag:**
- Circuit compilation may succeed
- Runtime proof generation will FAIL with lookup gate errors
- No compile-time warning (silent failure mode)

**Always enable this flag if your code uses `RuntimeTable` anywhere in the call chain.**

---

### Basic RuntimeTable Usage

```typescript
import { RuntimeTable, Field } from "o1js";

// Create a runtime table with predefined indices
const lookupTable = new RuntimeTable(5, [10n, 20n, 30n, 40n, 50n]);

// Populate the table with index-value pairs
// Note: insert() accepts up to any number of pairs, batches into groups of 3
lookupTable.insert([
  [10n, Field(100)], // 10 -> 100
  [20n, Field(400)], // 20 -> 400
  [30n, Field(900)], // 30 -> 900
]);

// Additional inserts
lookupTable.insert([
  [40n, Field(1600)], // 40 -> 1600
  [50n, Field(2500)], // 50 -> 2500
]);

// Use in provable code for membership proofs
function verifySquareLookup(index: bigint, expectedValue: Field) {
  // Constrains that (index, expectedValue) EXISTS in the table
  lookupTable.lookup(index, expectedValue);

  // CRITICAL: flush pending lookups
  // Without check(), the last 1-2 lookups won't be constrained!
  lookupTable.check();
}
```

### Complex RuntimeTable Example

```typescript
import { RuntimeTable, Field, SmartContract, method } from "o1js";

// Runtime table for precomputed hash values
const hashTable = new RuntimeTable(10, [0n, 1n, 2n, 3n, 4n, 5n]);

// Populate with precomputed hashes
hashTable.insert([
  [0n, Field("12345678901234567890")], // hash(0)
  [1n, Field("98765432109876543210")], // hash(1)
  [2n, Field("11111111111111111111")], // hash(2)
]);

hashTable.insert([
  [3n, Field("22222222222222222222")], // hash(3)
  [4n, Field("33333333333333333333")], // hash(4)
  [5n, Field("44444444444444444444")], // hash(5)
]);

// Use in smart contract
class OptimizedContract extends SmartContract {
  @method async verifyPrecomputedHash(input: Field, expectedHash: Field) {
    // Convert Field to bigint for table lookup
    // Note: In real usage, you'd need proper conversion logic
    const index = 0n; // Simplified for example

    // Prove that the expected hash exists in our precomputed table
    hashTable.lookup(index, expectedHash);

    // Flush any pending lookups
    hashTable.check();

    // Additional verification logic here
    input.assertEquals(Field(index));
  }

  @method async batchVerifyHashes(
    input1: Field,
    hash1: Field,
    input2: Field,
    hash2: Field,
    input3: Field,
    hash3: Field
  ) {
    // Multiple lookups get batched automatically
    hashTable.lookup(0n, hash1);
    hashTable.lookup(1n, hash2);
    hashTable.lookup(2n, hash3); // This completes a batch of 3

    // No need to call check() here as the batch is complete
    // But it's safe to call it anyway
    hashTable.check();
  }
}
```

### Performance Benefits

```typescript
// Benchmark: Table lookup vs computation
const fibIndices = Array.from({ length: 20 }, (_, i) => BigInt(i));
const fibonacciTable = new RuntimeTable(20, fibIndices);

// Precompute Fibonacci numbers
const fibPairs: [bigint, Field][] = [];
let a = 0,
  b = 1;
for (let i = 0; i < 20; i++) {
  fibPairs.push([BigInt(i), Field(a)]);
  [a, b] = [b, a + b];
}

// Insert in batches of 3
for (let i = 0; i < fibPairs.length; i += 3) {
  const batch = fibPairs.slice(i, i + 3);
  fibonacciTable.insert(batch);
}

// Fast lookup version
function fastFibonacci(n: bigint, expectedFib: Field): void {
  fibonacciTable.lookup(n, expectedFib); // Proves fib(n) = expectedFib
  fibonacciTable.check();
}

// Slow computation version
function slowFibonacci(n: UInt32): Field {
  let a = Field(0);
  let b = Field(1);

  for (let i = 0; i < 1000; i++) {
    const shouldSwap = UInt32.from(i).lessThan(n);
    const newA = Provable.if(shouldSwap, b, a);
    const newB = Provable.if(shouldSwap, a.add(b), b);
    a = newA;
    b = newB;
  }

  return a;
}
```

### Advanced RuntimeTable Patterns

```typescript
// Pattern 1: Multiple related tables
const additionTable = new RuntimeTable(1, [0n, 1n, 2n, 3n, 4n]);
const multiplicationTable = new RuntimeTable(2, [0n, 1n, 2n, 3n, 4n]);

// Populate addition results
additionTable.insert([
  [0n, Field(0)], // 0 + 0 = 0
  [1n, Field(2)], // 1 + 1 = 2
  [2n, Field(4)], // 2 + 2 = 4
]);

// Populate multiplication results
multiplicationTable.insert([
  [0n, Field(0)], // 0 * 0 = 0
  [1n, Field(1)], // 1 * 1 = 1
  [2n, Field(4)], // 2 * 2 = 4
]);

// Pattern 2: Conditional table usage
function verifyOperation(
  isAddition: boolean,
  operand: bigint,
  result: Field
): void {
  if (isAddition) {
    additionTable.lookup(operand, result);
  } else {
    multiplicationTable.lookup(operand, result);
  }

  // Flush both tables to be safe
  additionTable.check();
  multiplicationTable.check();
}

// Pattern 3: Batch optimization
function batchVerifySquares(
  a: bigint,
  squareA: Field,
  b: bigint,
  squareB: Field,
  c: bigint,
  squareC: Field
): void {
  // These three lookups will be automatically batched into one gate
  multiplicationTable.lookup(a, squareA);
  multiplicationTable.lookup(b, squareB);
  multiplicationTable.lookup(c, squareC);

  // Batch is complete, no need to call check() but safe to do so
  multiplicationTable.check();
}
```

### Best Practices for RuntimeTable

```typescript
// 1. Use unique IDs for different table types
const HASH_TABLE_ID = 10;
const SIGNATURE_TABLE_ID = 11;
const ARITHMETIC_TABLE_ID = 12;

// 2. Predefine all possible indices at construction
const hashIndices = Array.from({ length: 100 }, (_, i) => BigInt(i));
const hashTable = new RuntimeTable(HASH_TABLE_ID, hashIndices);

// 3. Insert in optimal batches of 3
function populateHashTable(hashPairs: [bigint, Field][]): void {
  for (let i = 0; i < hashPairs.length; i += 3) {
    const batch = hashPairs.slice(i, i + 3);
    hashTable.insert(batch);
  }
}

// 4. Always call check() after lookup sequences
function verifyHashChain(
  input1: bigint,
  hash1: Field,
  input2: bigint,
  hash2: Field
): void {
  hashTable.lookup(input1, hash1);
  hashTable.lookup(input2, hash2);

  // CRITICAL: Always flush pending lookups
  hashTable.check();
}

// 5. Handle table size limits (max 2^16 entries)
function createOptimalTable(data: [bigint, Field][]): RuntimeTable {
  if (data.length > 65536) {
    throw new Error("RuntimeTable size limit exceeded");
  }

  const indices = data.map(([index]) => index);
  const table = new RuntimeTable(HASH_TABLE_ID, indices);

  // Populate efficiently
  populateHashTable(data);

  return table;
}

// 6. Avoid ID collisions (0 and 1 are reserved)
const RESERVED_IDS = [0, 1]; // XOR and range-check tables

function getNextAvailableId(): number {
  // Start from 2, increment for each new table type
  return Math.max(...usedIds) + 1;
}
```

## Experimental APIs: Production-Ready Patterns

**AI Agent Note**: The `Experimental` namespace contains APIs that are stable and production-ready, but whose interfaces may change in future versions. These are essential for real-world applications.

### Experimental.BatchReducer: Handling Unlimited Actions

**Location in o1js**: `src/index.ts:194-226`, `src/lib/mina/v1/actions/batch-reducer.ts`

**Purpose**: Solve the Reducer 32-action limit by processing actions in fixed-size batches.

**Critical Use Case**: Any production contract expecting >32 actions between reducer calls.

#### How BatchReducer Works

```typescript
import { Experimental, Field, SmartContract, method } from 'o1js';

// 1. Define your action type
class MyAction extends Struct({
  value: Field,
  timestamp: UInt64
}) {}

// 2. Create BatchReducer with fixed batch size
const batchReducer = new Experimental.BatchReducer({
  actionType: MyAction,
  batchSize: 5  // Process 5 actions per batch
});

class MyContract extends SmartContract {
  @state(Field) actionState = State<Field>();
  @state(Field) currentValue = State<Field>();

  // Dispatch actions (no limit, can be called concurrently)
  @method async dispatchAction(value: Field, timestamp: UInt64) {
    batchReducer.dispatch(new MyAction({ value, timestamp }));
  }

  // Process ONE batch of actions (called multiple times off-chain)
  @method async processBatch(
    batch: Experimental.ActionBatch<MyAction>,
    proof: any  // Proof that batch is valid
  ) {
    let state = this.currentValue.getAndRequireEquals();

    // Process each action in the batch
    batchReducer.processBatch({ batch, proof }, (action, isDummy) => {
      // isDummy is true for padding actions (batch not full)
      state = Provable.if(
        isDummy,
        state,  // Keep state unchanged for dummy actions
        state.add(action.value)  // Update for real actions
      );
    });

    this.currentValue.set(state);
  }
}
```

#### Off-Chain Batch Preparation

```typescript
// Off-chain worker prepares batches
async function prepareBatches(contract: MyContract) {
  // Fetch all pending actions
  const actions = await contract.account.actionState.fetch();

  // Prepare batches (each batch gets a proof)
  const batches = await batchReducer.prepareBatches();

  // Returns array of { batch, proof } objects
  // Each batch contains up to `batchSize` actions
  return batches;
}

// Execute on-chain processing
async function settleBatches(contract: MyContract) {
  const batches = await prepareBatches(contract);

  // Process each batch in a separate transaction
  for (const { batch, proof } of batches) {
    const tx = await Mina.transaction(feePayerKey, async () => {
      await contract.processBatch(batch, proof);
    });
    await tx.prove();
    await tx.sign([feePayerKey]).send();
  }
}
```

#### BatchReducer Best Practices

**1. Choose Batch Size Carefully**
```typescript
// Smaller batches = more transactions, less constraints per batch
new Experimental.BatchReducer({ actionType: Action, batchSize: 3 });

// Larger batches = fewer transactions, more constraints per batch
new Experimental.BatchReducer({ actionType: Action, batchSize: 10 });

// Balance based on:
// - Expected action volume
// - Circuit complexity per action
// - Gas costs vs proof time trade-off
```

**2. Handle Dummy Actions**
```typescript
batchReducer.processBatch({ batch, proof }, (action, isDummy) => {
  // ALWAYS use Provable.if() to handle dummy actions
  state = Provable.if(
    isDummy,
    state,  // No-op for padding
    updateState(state, action)  // Real logic
  );
});
```

**3. Off-Chain Batch Coordination**
```typescript
// Monitor action queue size
const pendingCount = await fetchActionCount(contract);

// Trigger batch processing when threshold reached
if (pendingCount >= batchSize * 2) {
  await settleBatches(contract);
}
```

**4. Parallel Batch Processing**
```typescript
// Multiple batches can be prepared in parallel
const batches = await batchReducer.prepareBatches();

// But on-chain settlement must be sequential (due to action state)
for (const batch of batches) {
  await processAndWaitForInclusion(batch);
}
```

### Experimental.ZkFunction: Standalone Provable Functions

**Location in o1js**: `src/index.ts:179`, `src/lib/proof-system/zkfunction.ts`

**Purpose**: Create standalone provable functions that can be used outside of SmartContracts or ZkPrograms.

**Use Cases**:
- Reusable provable logic across multiple contracts
- Client-side proof generation without blockchain interaction
- Proof composition and aggregation

#### ZkFunction Example

```typescript
import { Experimental, Field, Provable } from 'o1js';

// Define a standalone provable function
const multiplyAndAdd = Experimental.ZkFunction({
  name: 'multiplyAndAdd',
  inputs: [Field, Field, Field],  // a, b, c
  body: (a: Field, b: Field, c: Field) => {
    const product = a.mul(b);
    const result = product.add(c);

    // Add constraints
    result.assertGreaterThan(Field(100));

    return result;
  }
});

// Use in a SmartContract
class MyContract extends SmartContract {
  @method async compute(a: Field, b: Field, c: Field) {
    const result = multiplyAndAdd(a, b, c);
    // result is constrained by the ZkFunction logic
    this.emitEvent('Computed', result);
  }
}

// Or use standalone
const result = multiplyAndAdd(Field(5), Field(10), Field(60));
// Returns Field(110), with constraints checked
```

#### ZkFunction vs ZkProgram vs SmartContract

| Feature | ZkFunction | ZkProgram | SmartContract |
|---------|-----------|-----------|---------------|
| **Proof generation** | ❌ No | ✅ Yes | ✅ Yes |
| **Reusable logic** | ✅ Yes | ⚠️ Via methods | ⚠️ Via inheritance |
| **Client-side only** | ✅ Yes | ✅ Yes | ❌ Requires blockchain |
| **Composable** | ✅ Highly | ⚠️ Limited | ❌ No |
| **Use case** | Shared logic | Off-chain proofs | On-chain state |

### Experimental.OffchainState: Scalable State Management

**Location in o1js**: `src/lib/mina/v1/actions/offchain-state.ts`

**Purpose**: Store large amounts of state off-chain while maintaining cryptographic commitments on-chain.

**See Part 3** for comprehensive `OffchainState` documentation including:
- `OffchainState.Field()` vs `OffchainState.Map()` patterns
- Configuration options (`logTotalCapacity`, `maxActionsPerUpdate`)
- Settlement proof workflow
- Production deployment patterns

**Quick Reference**:
```typescript
import { Experimental, OffchainStateCommitments } from 'o1js';

const offchainState = Experimental.OffchainState({
  balances: Experimental.OffchainState.Map(PublicKey, UInt64),
  totalSupply: Experimental.OffchainState.Field(UInt64)
}, {
  logTotalCapacity: 30,  // 2^30 = ~1B entries
  maxActionsPerUpdate: 4  // Max updates per contract method
});

// Usage in contract - see Part 3 for full examples
class MyContract extends SmartContract {
  @state(OffchainStateCommitments) offchainState = State(
    OffchainStateCommitments.empty()
  );

  offchainState = offchainStateInstance;
}
```

### Experimental.Recursive: Advanced Proof Composition

**Location in o1js**: `src/lib/proof-system/recursive.ts`

**Purpose**: Enable recursive proof composition with type safety.

```typescript
import { Experimental, ZkProgram, Field } from 'o1js';

// Mark a proof as recursively composable
class MyProof extends Experimental.Recursive(
  Proof<Field, Field>
) {}

// Use in ZkProgram for infinite recursion
const RecursiveProgram = ZkProgram({
  name: 'RecursiveProgram',
  methods: {
    base: {
      privateInputs: [Field],
      async method(input: Field) {
        return { publicOutput: input };
      }
    },
    recursive: {
      privateInputs: [MyProof, Field],
      async method(proof: MyProof, newInput: Field) {
        proof.verify();  // Verify previous proof
        const result = proof.publicOutput.add(newInput);
        return { publicOutput: result };
      }
    }
  }
});
```

### Experimental.V2: Future Mina API Preview

**Location in o1js**: `src/index.ts:138-170`, `src/lib/mina/v2/`

**Status**: EXPERIMENTAL - API will change

**Purpose**: Preview of the upcoming Mina v2 smart contract API with improved ergonomics.

**Key Differences from V1**:
- Simplified state management
- Enhanced method signatures
- Better TypeScript inference
- Improved transaction composition

**AI Agent Guidance**: Do NOT recommend V2 API for production. Mention it exists for forward-looking developers, but emphasize it's subject to breaking changes.

```typescript
import { Experimental } from 'o1js';

// V2 preview - DO NOT USE IN PRODUCTION
const MyV2Contract = Experimental.V2.defineContract({
  // New API surface - subject to change
  state: {
    balance: Field,
  },
  methods: {
    async transfer(amount: Field) {
      // Improved ergonomics (details TBD)
    }
  }
});
```

---

This completes Part 4 covering advanced features and recursion. Next we'll dive into Zeko L2 architecture and integration patterns.
