# Part 3: Smart Contract Development Patterns - zkApps, State Management, and Implementation Strategies

> **AI Agent Guidance**: This document covers patterns for building production zkApps. Reference established implementations (o1js TokenContract, Mina fungible-token-contract, Silvana's token) rather than suggesting custom implementations. Always reference ZkNoid's reducer articles for actions/reducers.

## SmartContract Class Architecture

**Key Limitation for AI Agents**: Mina contracts have only 8 Field elements for on-chain state. Guide users toward off-chain storage patterns for complex state.

### Basic Contract Structure

```typescript
import {
  SmartContract,
  State,
  state,
  method,
  Field,
  Bool,
  UInt64,
  PublicKey,
  Signature,
  Permissions,
  DeployArgs,
  AccountUpdate,
} from 'o1js';

class BasicContract extends SmartContract {
  // On-chain state declarations (max 8 Fields)
  @state(Field) counter = State<Field>();
  @state(Bool) isActive = State<Bool>();
  @state(PublicKey) owner = State<PublicKey>();

  // Contract initialization
  init() {
    super.init();
    this.counter.set(Field(0));
    this.isActive.set(Bool(true));
    this.owner.set(this.sender);
  }

  // Deploy method with custom permissions
  async deploy(args: DeployArgs) {
    await super.deploy(args);
    this.account.permissions.set({
      ...Permissions.default(),
      editState: Permissions.proof(), // Only proof can modify state
      send: Permissions.proof(),
      receive: Permissions.none(),
    });
  }

  // Provable method (generates constraints)
  @method async increment() {
    // Get current state with precondition
    const currentCounter = this.counter.getAndRequireEquals();
    const isActive = this.isActive.getAndRequireEquals();

    // Validate state
    isActive.assertTrue();

    // Update state
    this.counter.set(currentCounter.add(1));
  }

  // Method with authentication
  @method async setActive(isActive: Bool, signature: Signature) {
    // Get current owner
    const owner = this.owner.getAndRequireEquals();

    // Verify signature
    signature.verify(owner, isActive.toFields()).assertTrue();

    // Update state
    this.isActive.set(isActive);
  }
}
```

### State Management Patterns

#### **State Preconditions and Consistency**

```typescript
class StateConsistencyExample extends SmartContract {
  @state(Field) balance = State<Field>();
  @state(UInt64) lastUpdate = State<UInt64>();

  @method async transfer(amount: Field, recipient: PublicKey) {
    // Method 1: getAndRequireEquals() - Adds precondition automatically
    const currentBalance = this.balance.getAndRequireEquals();

    // Method 2: Manual precondition setting
    const lastUpdate = this.lastUpdate.get();
    this.lastUpdate.requireEquals(lastUpdate);

    // Validate sufficient balance
    currentBalance.assertGreaterThanOrEqual(amount);

    // Update state
    this.balance.set(currentBalance.sub(amount));
    this.lastUpdate.set(this.network.blockchainLength.getAndRequireEquals());

    // Send tokens (this creates an AccountUpdate)
    this.send({ to: recipient, amount: UInt64.from(amount) });
  }

  // Read-only method (no state changes)
  @method async getBalance(): Field {
    return this.balance.getAndRequireEquals();
  }
}
```

#### **Off-chain State with Merkle Trees**

```typescript
// Store large datasets off-chain with on-chain commitments
class OffChainStateContract extends SmartContract {
  @state(Field) usersRoot = State<Field>(); // Merkle root of all users
  @state(Field) balancesRoot = State<Field>(); // Merkle root of balances

  // User registration with merkle proof
  @method async registerUser(
    newUser: PublicKey,
    oldUsersRoot: Field,
    witness: MerkleWitness20
  ) {
    // Verify current root matches
    this.usersRoot.requireEquals(oldUsersRoot);

    // Calculate new root with user added
    const newUsersRoot = witness.calculateRoot(
      Poseidon.hash(newUser.toFields())
    );

    // Update on-chain root
    this.usersRoot.set(newUsersRoot);
  }

  // Update balance with proof of current state
  @method async updateBalance(
    user: PublicKey,
    oldBalance: Field,
    newBalance: Field,
    balanceWitness: MerkleWitness20,
    userWitness: MerkleWitness20
  ) {
    // Verify user exists
    const userHash = Poseidon.hash(user.toFields());
    const currentUsersRoot = this.usersRoot.getAndRequireEquals();
    userWitness.calculateRoot(userHash).assertEquals(currentUsersRoot);

    // Verify current balance
    const currentBalancesRoot = this.balancesRoot.getAndRequireEquals();
    balanceWitness.calculateRoot(oldBalance).assertEquals(currentBalancesRoot);

    // Calculate new balances root
    const newBalancesRoot = balanceWitness.calculateRoot(newBalance);
    this.balancesRoot.set(newBalancesRoot);
  }
}
```

## Permission System and Security

### Comprehensive Permission Configuration

```typescript
class SecureContract extends SmartContract {
  @state(PublicKey) admin = State<PublicKey>();
  @state(Field) criticalData = State<Field>();

  async deploy(args: DeployArgs) {
    await super.deploy(args);

    // Configure detailed permissions
    this.account.permissions.set({
      // State modification permissions
      editState: Permissions.proof(), // Only via zkApp methods
      editActionState: Permissions.proof(),

      // Token permissions
      send: Permissions.proof(), // Can send tokens via methods
      receive: Permissions.none(), // Cannot receive tokens

      // Account modification permissions
      setDelegate: Permissions.signature(), // Owner can delegate
      setPermissions: Permissions.impossible(), // Cannot change permissions
      setVerificationKey: Permissions.signature(), // Owner can upgrade

      // Voting and timing permissions
      setVotingFor: Permissions.signature(),
      setTiming: Permissions.signature(),

      // Most restrictive permission
      access: Permissions.proofOrSignature(), // Proof OR signature required
    });
  }

  // Admin-only method
  @method async adminUpdateData(newData: Field, signature: Signature) {
    const admin = this.admin.getAndRequireEquals();

    // Verify admin signature
    signature.verify(admin, [newData]).assertTrue();

    this.criticalData.set(newData);
  }

  // Public method with proof requirement
  @method async publicUpdate(newData: Field) {
    // Any user can call, but must provide valid proof
    const currentData = this.criticalData.getAndRequireEquals();

    // Business logic validation
    newData.assertGreaterThan(currentData);

    this.criticalData.set(newData);
  }
}
```

### Access Control Patterns

```typescript
// Role-based access control
class RoleBasedContract extends SmartContract {
  @state(Field) adminRole = State<Field>();
  @state(Field) userRole = State<Field>();

  // Define role constants
  static readonly ADMIN_ROLE = Field(1);
  static readonly USER_ROLE = Field(2);
  static readonly MODERATOR_ROLE = Field(3);

  @method async grantRole(user: PublicKey, role: Field, adminSig: Signature) {
    const admin = this.getAdminFromRole(this.adminRole.getAndRequireEquals());

    // Verify admin signature
    adminSig.verify(admin, [...user.toFields(), role]).assertTrue();

    // Role validation
    this.validateRole(role);

    // Store role in off-chain merkle tree (implementation depends on design)
    this.emitEvent('RoleGranted', { user, role });
  }

  @method async requireRole(user: PublicKey, requiredRole: Field, proof: any) {
    // Verify user has required role via merkle proof
    this.verifyRoleProof(user, requiredRole, proof);
  }

  private validateRole(role: Field): void {
    const validRole = role.equals(RoleBasedContract.ADMIN_ROLE)
      .or(role.equals(RoleBasedContract.USER_ROLE))
      .or(role.equals(RoleBasedContract.MODERATOR_ROLE));

    validRole.assertTrue();
  }

  private getAdminFromRole(roleState: Field): PublicKey {
    // Implementation depends on how admin is stored
    return PublicKey.empty(); // Placeholder
  }

  private verifyRoleProof(user: PublicKey, role: Field, proof: any): void {
    // Implementation depends on merkle tree structure
  }
}
```

## Events and Actions System

### Events: Public Logging

```typescript
class EventLoggingContract extends SmartContract {
  events = {
    'UserRegistered': PublicKey,
    'BalanceUpdated': Provable.Struct({
      user: PublicKey,
      oldBalance: UInt64,
      newBalance: UInt64,
      timestamp: UInt64,
    }),
    'AdminAction': Provable.Struct({
      admin: PublicKey,
      action: Field,
      target: PublicKey,
      data: Field,
    }),
  };

  @method async registerUser(user: PublicKey) {
    // Registration logic here

    // Emit event for off-chain indexing
    this.emitEvent('UserRegistered', user);
  }

  @method async updateBalance(
    user: PublicKey,
    amount: UInt64
  ) {
    const oldBalance = this.getBalance(user); // Implementation specific
    const newBalance = oldBalance.add(amount);

    // Update balance logic here

    // Emit detailed event
    this.emitEvent('BalanceUpdated', {
      user,
      oldBalance,
      newBalance,
      timestamp: this.network.timestamp.getAndRequireEquals(),
    });
  }

  private getBalance(user: PublicKey): UInt64 {
    // Implementation depends on storage strategy
    return UInt64.zero; // Placeholder
  }
}
```

### Actions: Batch Processing and Reducers

**Actions and reducers are critical patterns in Mina zkApps for handling concurrent state updates.** They solve the fundamental problem where multiple transactions trying to update the same state simultaneously can cause failures due to Mina's execute-order-validate model.

#### **The Concurrent State Update Problem**

In traditional blockchains, transactions are ordered first, then executed. But Mina uses execute-order-validate: transactions are executed client-side, generating proofs that are then ordered and validated. This creates a problem where multiple users updating the same state simultaneously will cause all but the first transaction to fail when their preconditions become invalid.

**For comprehensive understanding of actions/reducers, refer to ZkNoid's excellent 5-part series:**

1. **[Why We Need Them](./zknoid-action-reducer/1.md)** - Understanding the concurrent state update problem
2. **[Let's Take a Closer Look](./zknoid-action-reducer/2.md)** - Deep dive into Merkle lists and action state mechanics
3. **[Writing Our Own Reducers](./zknoid-action-reducer/3.md)** - Three reducer approaches: default, recursive proof, and snapshot
4. **[Off-chain Storage](./zknoid-action-reducer/4.md)** - Using o1js off-chain storage structures
5. **[Batch Reducers](./zknoid-action-reducer/5.md)** - Advanced pattern for unlimited actions

#### **Basic Action/Reducer Pattern**

```typescript
class TokenWithActions extends SmartContract {
  @state(Field) totalSupply = State<Field>();
  @state(Field) balancesRoot = State<Field>();
  @state(Field) lastProcessedActionState = State<Field>();

  // Define action types
  static TransferAction = Provable.Struct({
    from: PublicKey,
    to: PublicKey,
    amount: UInt64,
  });

  reducer = Reducer({ actionType: TokenWithActions.TransferAction });

  init() {
    super.init();
    this.lastProcessedActionState.set(Reducer.initialActionState);
  }

  // Dispatch transfer request (no state changes yet)
  @method async requestTransfer(to: PublicKey, amount: UInt64) {
    const from = this.sender.getAndRequireSignature();

    // Basic validation
    amount.assertGreaterThan(UInt64.zero);

    // Dispatch action - multiple users can do this concurrently
    this.reducer.dispatch(
      new TokenWithActions.TransferAction({ from, to, amount })
    );
  }

  // Process all pending transfer actions
  @method async processTransfers() {
    const lastProcessedState = this.lastProcessedActionState.getAndRequireEquals();
    const currentBalancesRoot = this.balancesRoot.getAndRequireEquals();

    // Get all pending actions
    const pendingActions = this.reducer.getActions({
      fromActionState: lastProcessedState,
    });

    // Process actions and update balances
    const { state: newBalancesRoot, actionState: newActionState } = this.reducer.reduce(
      pendingActions,
      Field, // State type
      (balancesRoot: Field, action: TokenWithActions.TransferAction) => {
        // Process individual transfer
        return this.processTransfer(balancesRoot, action);
      },
      currentBalancesRoot, // Initial state
      { maxUpdatesWithActions: 32 } // Limitation: max 32 actions
    );

    // Update state
    this.balancesRoot.set(newBalancesRoot);
    this.lastProcessedActionState.set(newActionState);
  }

  private processTransfer(
    balancesRoot: Field,
    action: TokenWithActions.TransferAction
  ): Field {
    // Implementation would include:
    // 1. Verify sender has sufficient balance
    // 2. Update Merkle tree with new balances
    // 3. Return new root
    return balancesRoot; // Simplified
  }
}
```

#### **Advanced Reducer Patterns**

**1. Batch Reducers (Recommended for Production)**

For unlimited actions without the 32-action limit:

```typescript
import { BatchReducer } from 'o1js';

export const batchReducer = new BatchReducer({
  actionType: Field,
  batchSize: 2,
});

class ProductionContract extends SmartContract {
  @state(Field) actionState = State(BatchReducer.initialActionState);
  @state(Field) actionStack = State(BatchReducer.initialActionStack);
  @state(Field) counter = State<Field>();

  init() {
    super.init();
    batchReducer.setContractInstance(this);
  }

  @method async add(value: Field) {
    value.assertGreaterThan(Field(0));
    batchReducer.dispatch(value);
  }

  @method async batchReduce(batch: any, proof: any) {
    const currentTotal = this.counter.getAndRequireEquals();
    let newTotal = currentTotal;

    batchReducer.processBatch({ batch, proof }, (number, isDummy) => {
      newTotal = Provable.if(isDummy, newTotal, newTotal.add(number));
    });

    this.counter.set(newTotal);
  }
}
```

**2. Off-chain Storage Pattern**

For complex state management:

```typescript
import { OffchainState } from 'o1js';

const offchainState = OffchainState(
  {
    accounts: OffchainState.Map(PublicKey, UInt64),
    totalSupply: OffchainState.Field(UInt64),
  },
  { logTotalCapacity: 10, maxActionsPerProof: 5 }
);

class OffchainContract extends SmartContract {
  @state(OffchainState.Commitments) offchainStateCommitments =
    offchainState.emptyCommitments();

  offchainState = offchainState.init(this);

  @method async transfer(to: PublicKey, amount: UInt64) {
    const from = this.sender.getAndRequireSignature();

    // Get current balances
    const fromBalance = await this.offchainState.fields.accounts.get(from);
    const toBalance = await this.offchainState.fields.accounts.get(to);

    // Update with conflict detection
    this.offchainState.fields.accounts.update(from, {
      from: fromBalance,
      to: fromBalance.orElse(0n).sub(amount),
    });

    this.offchainState.fields.accounts.update(to, {
      from: toBalance,
      to: toBalance.orElse(0n).add(amount),
    });
  }

  @method async settle(proof: OffchainState.Proof) {
    await this.offchainState.settle(proof);
  }
}
```

#### **Key Takeaways**

- **Use actions/reducers when multiple users need to update shared state concurrently**
- **Default reducer has 32-action limit - use batch reducers for production**
- **Off-chain storage provides native-like state management with conflict detection**
- **Actions are queued immediately; reducers process them later**
- **Study ZkNoid's articles for comprehensive implementation patterns**
```

## Token Contracts and Custom Tokens

**For token development on Mina, use established implementations rather than building from scratch.** The ecosystem provides several production-ready token contracts that handle the complexities of Mina's token system correctly.

### **Recommended Token Implementations**

#### **1. o1js TokenContract (Built-in Base Class)**

The o1js library provides a base `TokenContract` class that handles core token functionality:

```typescript
import { TokenContract, UInt64, PublicKey, method } from 'o1js';

// Extend the built-in TokenContract
class MyToken extends TokenContract {
  @method async approveSend(
    forest: AccountUpdateForest
  ): Promise<AccountUpdateForest> {
    // Override to implement custom approval logic
    this.checkZeroBalanceChange(forest);
    return forest;
  }

  @method async mint(recipient: PublicKey, amount: UInt64) {
    // Only token owner can mint
    this.internal.mint({ address: recipient, amount });
  }

  @method async burn(owner: PublicKey, amount: UInt64) {
    // Burn tokens from owner
    this.internal.burn({ address: owner, amount });
  }
}
```

**Key Features:**
- Built-in token protocol compliance
- Automatic balance tracking
- Transfer validation
- Integration with Mina's account system

**Location**: `./o1js/src/lib/mina/v1/token/token-contract.ts`

#### **2. Mina Fungible Token Contract (Official Standard)**

The official Mina Protocol fungible token implementation provides a complete, audited token contract:

```typescript
// Using the official Mina fungible token
import { FungibleToken, FungibleTokenAdmin } from 'mina-fungible-token';

// Deploy the admin contract first
const admin = new FungibleTokenAdmin(adminAddress);

// Deploy the token contract
const token = new FungibleToken(tokenAddress);

// Initialize with admin
await token.initialize(
  admin.address,
  UInt8.from(9), // decimals
  Bool(false)    // disable mint initially
);
```

**Key Features:**
- ERC-20 compatible interface
- Comprehensive access controls
- Pausable functionality
- Burnable tokens
- Admin role management
- Production tested and audited

**Repository**: `./mina-fungible-token`
**Documentation**: Follow official Mina documentation for deployment and usage

#### **3. Silvana Token Implementation (Best Practice Example)**

Silvana Labs provides an exemplary token implementation demonstrating best practices:

```typescript
// Based on Silvana's implementation
import { FungibleToken } from '@silvana/token';

class ProductionToken extends FungibleToken {
  // Inherits robust implementation with:
  // - Proper error handling
  // - Gas optimization
  // - Security best practices
  // - Comprehensive testing
}
```

**Key Features:**
- Battle-tested in production
- Optimized for gas efficiency
- Comprehensive error handling
- Full test coverage
- Security audit passed

**Location**: `./silvana-lib/packages/token`

### **Development Guidance**

#### **For New Projects:**

1. **Start with Mina Fungible Token Contract** for standard ERC-20-like functionality
2. **Use o1js TokenContract base class** for custom token logic that needs deep protocol integration
3. **Study Silvana's implementation** for production best practices and patterns

#### **Key Considerations:**

- **Don't reinvent token mechanics** - use proven implementations
- **Token transfers in Mina require special handling** due to the account system
- **Always test token interactions thoroughly** on devnet before mainnet
- **Consider using actions/reducers** for complex token operations requiring state coordination

#### **Integration Example**

```typescript
import { FungibleToken } from 'mina-fungible-token';
import { MyDApp } from './my-dapp';

class TokenIntegratedDApp extends MyDApp {
  tokenContract = new FungibleToken(this.tokenAddress);

  @method async payWithTokens(amount: UInt64) {
    // Transfer tokens from user to contract
    this.tokenContract.transfer(
      this.sender,
      this.address,
      amount
    );

    // Perform dApp logic
    this.performService();
  }
}
```

### **Resources and Documentation**

- **Official Mina Docs**: [Custom Tokens Tutorial](https://docs.minaprotocol.com/zkapps/tutorials/custom-tokens)
- **o1js TokenContract**: Study the base implementation for understanding core mechanics
- **Mina Fungible Token**: Reference implementation for standard functionality
- **Silvana Token Package**: Production-grade patterns and optimizations

**Critical AI Agent Warning**:
- **NEVER suggest implementing token contracts from scratch**
- **ALWAYS recommend using established implementations** (o1js TokenContract, Mina fungible-token-contract, Silvana's implementation)
- **Token mechanics in Mina are complex** - incorrect implementations can lead to loss of funds
- **Guide users to proven, audited contracts** rather than custom implementations

## Network Integration and Transaction Patterns

### Network State Access

```typescript
class NetworkStateContract extends SmartContract {
  @state(UInt64) lastBlockHeight = State<UInt64>();
  @state(UInt64) creationTimestamp = State<UInt64>();

  @method async updateWithNetworkData() {
    // Access network state
    const currentHeight = this.network.blockchainLength.getAndRequireEquals();
    const currentTimestamp = this.network.timestamp.getAndRequireEquals();
    const totalCurrency = this.network.totalCurrency.getAndRequireEquals();

    // Network assertions
    this.network.timestamp.requireBetween(
      UInt64.from(1640995200), // January 1, 2022
      UInt64.from(2000000000)  // May 18, 2033
    );

    // Update state with network data
    this.lastBlockHeight.set(currentHeight);

    // Emit event with network data
    this.emitEvent('NetworkUpdate', {
      height: currentHeight,
      timestamp: currentTimestamp,
      totalCurrency,
    });
  }

  @method async timeBasedLogic() {
    const creationTime = this.creationTimestamp.getAndRequireEquals();
    const currentTime = this.network.timestamp.getAndRequireEquals();
    const timeDiff = currentTime.sub(creationTime);

    // Time-based constraints
    const oneDay = UInt64.from(24 * 60 * 60 * 1000); // 24 hours in milliseconds
    timeDiff.assertGreaterThanOrEqual(oneDay);

    // Time-based state updates
    const daysPassed = timeDiff.div(oneDay);
    this.processTimePeriod(daysPassed);
  }

  private processTimePeriod(days: UInt64): void {
    // Implementation specific to time-based logic
  }
}
```

### Account Information Access

```typescript
class AccountManagementContract extends SmartContract {
  @method async manageAccount(targetAccount: PublicKey) {
    // Access account information
    const balance = AccountUpdate.create(targetAccount).account.balance.getAndRequireEquals();
    const nonce = AccountUpdate.create(targetAccount).account.nonce.getAndRequireEquals();
    const delegate = AccountUpdate.create(targetAccount).account.delegate.getAndRequireEquals();

    // Account assertions
    balance.assertGreaterThan(UInt64.from(1000000)); // Minimum balance requirement

    // Create account update
    const accountUpdate = AccountUpdate.create(targetAccount);
    accountUpdate.account.balance.requireEquals(balance);
    accountUpdate.requireSignature(); // Require account signature

    // Perform account operations
    this.performAccountOperation(accountUpdate);
  }

  @method async sendTokens(recipient: PublicKey, amount: UInt64) {
    // Send MINA tokens
    this.send({ to: recipient, amount });

    // Send custom tokens
    this.token.send({
      from: this.address,
      to: recipient,
      amount,
    });
  }

  private performAccountOperation(accountUpdate: AccountUpdate): void {
    // Implementation specific operations
  }
}
```

## Complex State Management Patterns

### Multi-Contract Interaction

```typescript
// Contract A
class ContractA extends SmartContract {
  @state(Field) sharedData = State<Field>();

  @method async updateSharedData(newData: Field, contractB: ContractB) {
    // Update local state
    this.sharedData.set(newData);

    // Call method on Contract B
    await contractB.receiveUpdate(newData);
  }
}

// Contract B
class ContractB extends SmartContract {
  @state(Field) receivedData = State<Field>();

  @method async receiveUpdate(data: Field) {
    // Verify caller (could be done via signature or other means)
    this.receivedData.set(data);
  }

  @method async crossContractRead(contractA: ContractA): Field {
    // Read from another contract (non-modifying)
    return contractA.sharedData.getAndRequireEquals();
  }
}
```

### Proxy Pattern for Upgradability

```typescript
// Proxy contract that delegates to implementation
class ProxyContract extends SmartContract {
  @state(PublicKey) implementation = State<PublicKey>();
  @state(PublicKey) admin = State<PublicKey>();

  @method async upgrade(newImplementation: PublicKey, adminSig: Signature) {
    const admin = this.admin.getAndRequireEquals();

    // Verify admin signature
    adminSig.verify(admin, newImplementation.toFields()).assertTrue();

    // Update implementation
    this.implementation.set(newImplementation);
  }

  @method async delegateCall(methodData: Field[], signature: Signature) {
    const impl = this.implementation.getAndRequireEquals();

    // Verify signature and delegate to implementation
    // Implementation would need to be more complex in practice
    this.performDelegatedCall(impl, methodData, signature);
  }

  private performDelegatedCall(
    impl: PublicKey,
    methodData: Field[],
    signature: Signature
  ): void {
    // Complex implementation for delegation
    // This is a simplified example
  }
}

// Implementation contract
class ImplementationV1 extends SmartContract {
  @state(Field) data = State<Field>();

  @method async businessLogic(input: Field) {
    // Version 1 business logic
    this.data.set(input.mul(2));
  }
}

class ImplementationV2 extends SmartContract {
  @state(Field) data = State<Field>();

  @method async businessLogic(input: Field) {
    // Version 2 business logic (improved)
    this.data.set(input.mul(3).add(1));
  }
}
```

### Factory Pattern for Contract Creation

```typescript
class ContractFactory extends SmartContract {
  @state(Field) contractCount = State<Field>();
  @state(Field) contractsRoot = State<Field>(); // Merkle root of created contracts

  events = {
    'ContractCreated': Provable.Struct({
      contractAddress: PublicKey,
      creator: PublicKey,
      contractType: Field,
      index: Field,
    }),
  };

  @method async createContract(
    contractType: Field,
    initData: Field[],
    witness: MerkleWitness20
  ) {
    const currentCount = this.contractCount.getAndRequireEquals();
    const creator = this.sender;

    // Generate deterministic address
    const contractAddress = this.deriveContractAddress(creator, currentCount);

    // Update contracts merkle tree
    const contractHash = Poseidon.hash([
      ...contractAddress.toFields(),
      creator.toFields()[0],
      contractType,
    ]);

    const newRoot = witness.calculateRoot(contractHash);
    this.contractsRoot.set(newRoot);

    // Update count
    this.contractCount.set(currentCount.add(1));

    // Emit creation event
    this.emitEvent('ContractCreated', {
      contractAddress,
      creator,
      contractType,
      index: currentCount,
    });

    // Deploy actual contract (implementation specific)
    this.deployContract(contractAddress, contractType, initData);
  }

  private deriveContractAddress(creator: PublicKey, index: Field): PublicKey {
    // Deterministic address generation
    const hash = Poseidon.hash([...creator.toFields(), index]);
    return PublicKey.fromFields([hash, Field(0)]); // Simplified
  }

  private deployContract(
    address: PublicKey,
    contractType: Field,
    initData: Field[]
  ): void {
    // Contract deployment logic
    // This would involve creating the actual contract instance
  }
}
```

This completes Part 3 covering smart contract development patterns. Next we'll explore advanced features including recursion, ZkPrograms, and complex proof composition.