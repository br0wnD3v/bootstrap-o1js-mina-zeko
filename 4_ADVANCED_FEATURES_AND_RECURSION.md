# Part 4: Advanced Features and Recursion - ZkPrograms, Infinite Recursion, and Complex Proof Composition

## ZkPrograms: Advanced Proof Systems

### Basic ZkProgram Structure

```typescript
import { ZkProgram, Field, Struct, SelfProof, verify } from 'o1js';

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
  name: 'math-program',
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
const recursiveInput = new ProgramInput({ value: Field(3), multiplier: Field(4) });
const recursiveProof = await MathProgram.recursiveStep(recursiveInput, baseProof);

console.log('Final result:', recursiveProof.publicOutput.result.toString());
console.log('Steps taken:', recursiveProof.publicOutput.stepCount.toString());
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
  name: 'counter-state-machine',
  publicInput: Field, // New value to add
  publicOutput: CounterState,

  methods: {
    // Initialize counter
    init: {
      privateInputs: [Field], // maxValue
      async method(
        increment: Field,
        maxValue: Field
      ): Promise<CounterState> {
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
        const finalValue = Provable.if(exceedsMax, prevState.maxValue, newValue);
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
  name: 'tree-computation',
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
    values.map(value => TreeProgram.leaf(value))
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
  name: 'transaction-rollup',
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
        return this.processTransaction(intermediateState, transaction, stateWitness);
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
  name: 'cross-chain-messaging',
  publicInput: ChainMessage,
  publicOutput: ChainState,

  methods: {
    // Initialize chain state
    initChain: {
      privateInputs: [Field], // Chain ID
      async method(
        message: ChainMessage,
        chainId: Field
      ): Promise<ChainState> {
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
  name: 'universal-verifier',
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

    adminSignature.verify(admin, [newVersion, newImplementationHash]).assertTrue();

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
  name: 'fibonacci-optimized',
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
  name: 'game-state-machine',
  publicInput: GameAction,
  publicOutput: GameStateData,

  methods: {
    // Initialize new game
    initGame: {
      privateInputs: [Field], // Game ID
      async method(
        action: GameAction,
        gameId: Field
      ): Promise<GameStateData> {
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
  const canStart = currentStateField.equals(Field(GameState.WAITING))
    .and(actionType.equals(Field(1)));

  // Make move transition
  const canMove = currentStateField.equals(Field(GameState.PLAYING))
    .and(actionType.equals(Field(2)))
    .and(action.player.equals(currentState.currentPlayer));

  // Pause transition
  const canPause = currentStateField.equals(Field(GameState.PLAYING))
    .and(actionType.equals(Field(3)));

  // Resume transition
  const canResume = currentStateField.equals(Field(GameState.PAUSED))
    .and(actionType.equals(Field(4)));

  // End game transition
  const canEnd = actionType.equals(Field(5));

  // Validate at least one transition is valid
  const isValidTransition = canStart.or(canMove).or(canPause).or(canResume).or(canEnd);
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

This completes Part 4 covering advanced features and recursion. Next we'll dive into Zeko L2 architecture and integration patterns.