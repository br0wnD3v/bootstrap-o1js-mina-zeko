# Part 3: Smart Contract Development Patterns - zkApps, State Management, and Implementation Strategies

## SmartContract Class Architecture

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

```typescript
class ActionProcessingContract extends SmartContract {
  @state(Field) processedActionState = State<Field>();
  @state(UInt64) totalProcessedActions = State<UInt64>();

  // Define action type
  static actionType = Provable.Struct({
    user: PublicKey,
    amount: UInt64,
    operation: Field, // 1 = deposit, 2 = withdraw
  });

  // Initialize reducer
  reducer = Reducer({ actionType: ActionProcessingContract.actionType });

  @method async dispatchAction(
    user: PublicKey,
    amount: UInt64,
    operation: Field
  ) {
    // Validate operation type
    const validOperation = operation.equals(Field(1)).or(operation.equals(Field(2)));
    validOperation.assertTrue();

    // Dispatch action to be processed later
    this.reducer.dispatch({
      user,
      amount,
      operation,
    });
  }

  @method async processActions() {
    // Get current state
    const currentActionState = this.processedActionState.getAndRequireEquals();
    const currentCount = this.totalProcessedActions.getAndRequireEquals();

    // Get all pending actions
    const pendingActions = this.reducer.getActions({
      fromActionState: currentActionState,
    });

    // Process actions in batch
    const { state: newCount, actionState: newActionState } = this.reducer.reduce(
      pendingActions,
      UInt64, // State type
      (state: UInt64, action: any) => {
        // Process individual action
        this.processIndividualAction(action);
        return state.add(1);
      },
      currentCount // Initial state
    );

    // Update state
    this.totalProcessedActions.set(newCount);
    this.processedActionState.set(newActionState);
  }

  private processIndividualAction(action: any): void {
    // Implement specific action processing logic
    const isDeposit = action.operation.equals(Field(1));

    // Different logic for deposits vs withdrawals
    Provable.if(
      isDeposit,
      () => this.processDeposit(action.user, action.amount),
      () => this.processWithdrawal(action.user, action.amount)
    );
  }

  private processDeposit(user: PublicKey, amount: UInt64): void {
    // Deposit processing logic
  }

  private processWithdrawal(user: PublicKey, amount: UInt64): void {
    // Withdrawal processing logic
  }
}
```

## Token Contracts and Custom Tokens

### Basic Token Implementation

```typescript
class CustomToken extends SmartContract {
  @state(UInt64) totalSupply = State<UInt64>();
  @state(PublicKey) tokenOwner = State<PublicKey>();

  // Token metadata (could be stored off-chain)
  static readonly TOKEN_NAME = "MyToken";
  static readonly TOKEN_SYMBOL = "MTK";
  static readonly DECIMALS = 9;

  init() {
    super.init();
    this.totalSupply.set(UInt64.zero);
    this.tokenOwner.set(this.sender);
  }

  @method async mint(recipient: PublicKey, amount: UInt64) {
    // Only token owner can mint
    const owner = this.tokenOwner.getAndRequireEquals();
    this.sender.assertEquals(owner);

    // Update total supply
    const currentSupply = this.totalSupply.getAndRequireEquals();
    this.totalSupply.set(currentSupply.add(amount));

    // Mint tokens to recipient
    this.token.mint({
      address: recipient,
      amount,
    });
  }

  @method async burn(amount: UInt64) {
    // Burn from sender's account
    this.token.burn({
      address: this.sender,
      amount,
    });

    // Update total supply
    const currentSupply = this.totalSupply.getAndRequireEquals();
    this.totalSupply.set(currentSupply.sub(amount));
  }

  @method async transfer(from: PublicKey, to: PublicKey, amount: UInt64) {
    // This method allows the contract to facilitate transfers
    this.token.send({
      from,
      to,
      amount,
    });
  }

  // Approve pattern for delegated transfers
  @method async approveTransfer(
    owner: PublicKey,
    spender: PublicKey,
    amount: UInt64,
    signature: Signature
  ) {
    // Verify owner's signature for approval
    signature.verify(owner, [...spender.toFields(), amount.value]).assertTrue();

    // Store approval off-chain or use actions for batch processing
    this.emitEvent('Approval', { owner, spender, amount });
  }
}
```

### Advanced Token Features

```typescript
class AdvancedToken extends SmartContract {
  @state(UInt64) totalSupply = State<UInt64>();
  @state(Bool) paused = State<Bool>();
  @state(Field) blacklistRoot = State<Field>(); // Merkle root of blacklisted addresses

  events = {
    'Transfer': Provable.Struct({
      from: PublicKey,
      to: PublicKey,
      amount: UInt64,
    }),
    'Approval': Provable.Struct({
      owner: PublicKey,
      spender: PublicKey,
      amount: UInt64,
    }),
    'Pause': Bool,
    'Blacklist': PublicKey,
  };

  @method async transferWithChecks(
    from: PublicKey,
    to: PublicKey,
    amount: UInt64,
    fromBlacklistWitness: MerkleWitness20,
    toBlacklistWitness: MerkleWitness20
  ) {
    // Check if contract is paused
    const isPaused = this.paused.getAndRequireEquals();
    isPaused.assertFalse();

    // Verify neither address is blacklisted
    this.verifyNotBlacklisted(from, fromBlacklistWitness);
    this.verifyNotBlacklisted(to, toBlacklistWitness);

    // Perform transfer
    this.token.send({ from, to, amount });

    // Emit event
    this.emitEvent('Transfer', { from, to, amount });
  }

  @method async pause(adminSignature: Signature) {
    const admin = this.getAdmin(); // Implementation specific

    // Verify admin signature
    adminSignature.verify(admin, [Bool(true).toField()]).assertTrue();

    this.paused.set(Bool(true));
    this.emitEvent('Pause', Bool(true));
  }

  @method async addToBlacklist(
    address: PublicKey,
    witness: MerkleWitness20,
    adminSignature: Signature
  ) {
    const admin = this.getAdmin();

    // Verify admin signature
    adminSignature.verify(admin, address.toFields()).assertTrue();

    // Update blacklist merkle tree
    const currentRoot = this.blacklistRoot.getAndRequireEquals();
    const addressHash = Poseidon.hash(address.toFields());
    const newRoot = witness.calculateRoot(addressHash);

    this.blacklistRoot.set(newRoot);
    this.emitEvent('Blacklist', address);
  }

  private verifyNotBlacklisted(
    address: PublicKey,
    witness: MerkleWitness20
  ): void {
    const blacklistRoot = this.blacklistRoot.getAndRequireEquals();
    const addressHash = Poseidon.hash(address.toFields());

    // This should fail if address is in blacklist
    const calculatedRoot = witness.calculateRoot(addressHash);
    calculatedRoot.assertNotEquals(blacklistRoot);
  }

  private getAdmin(): PublicKey {
    // Implementation depends on admin storage strategy
    return PublicKey.empty(); // Placeholder
  }
}
```

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