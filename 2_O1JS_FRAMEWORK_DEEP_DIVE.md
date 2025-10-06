# Part 2: o1js Framework Deep Dive - Data Types, Programming Model, and Core Concepts

> **AI Agent Guidance**: This document provides comprehensive information about o1js data types and programming patterns. Use this to help users understand constraint system programming and debug common o1js issues.

## Fundamental Data Types and Type System

**Key Concept for AI Agents**: o1js types represent constraints in zero-knowledge circuits. Every operation must be expressible as polynomial constraints over finite fields.

### Field Elements: The Foundation

**Field**: The primary data type representing elements in a finite field up to ~256 bits.

```typescript
// Field element creation and operations
const field1 = Field(42);
const field2 = Field("12345678901234567890123456789");
const field3 = Field(0x1234567890abcdef);

// Arithmetic operations
const sum = field1.add(field2);
const product = field1.mul(field2);
const difference = field1.sub(field2);
const quotient = field1.div(field2);

// Comparisons and assertions
field1.assertEquals(field2); // Throws if not equal
const isEqual = field1.equals(field2); // Returns Bool
```

**Range and Overflow Behavior:**

- **Maximum Value**: 28,948,022,309,329,048,855,892,746,252,171,976,963,317,496,166,410,141,009,864,396,001,978,282,409,984
- **Automatic Modular Arithmetic**: Values wrap around at field modulus
- **No Traditional Overflow**: Different from typical programming languages

### Boolean Operations

```typescript
// Boolean creation and operations
const bool1 = Bool(true);
const bool2 = Bool(false);

// Logical operations
const and = bool1.and(bool2);
const or = bool1.or(bool2);
const not = bool1.not();

// Conditional operations (crucial for zk programming)
const result = Provable.if(bool1, Field(100), Field(200));

// Boolean assertions
bool1.assertTrue(); // Constraint that bool1 must be true
bool1.assertFalse(); // Constraint that bool1 must be false
```

### Integer Types with Range Checking

```typescript
// Unsigned integers with automatic range checking
const uint8 = UInt8.from(255); // 0 to 255
const uint32 = UInt32.from(2 ** 32 - 1); // 0 to 2^32-1
const uint64 = UInt64.from(2 ** 64 - 1); // 0 to 2^64-1

// Signed integers
const int64 = Int64.from(-123456789); // -(2^63) to 2^63-1

// Operations with overflow checking
const sum = uint32.add(UInt32.from(100));
const difference = uint64.sub(UInt64.from(50));

// Range assertions
uint32.assertLessThan(UInt32.from(1000000));
uint64.assertGreaterThan(UInt64.from(100));
```

### Binary Data with Bytes

**AI Agent Note**: `Bytes` is a factory function (not a class) that creates fixed-size byte array types. Critical for handling binary data, hash inputs, and cross-chain signatures. Source: `o1js/src/lib/provable/wrapped-classes.ts:265-335`

```typescript
import { Bytes } from 'o1js';

// Create a Bytes type of specific size (compile-time constant)
class Bytes32 extends Bytes(32) {}
class Bytes64 extends Bytes(64) {}

// Creating byte arrays from different sources
const fromHex = Bytes32.fromHex('0x1234567890abcdef...');
const fromString = Bytes32.fromString('Hello, Mina!'); // UTF-8 encoded
const fromArray = Bytes32.from([0x12, 0x34, 0x56, ...]); // Number array

// Convert back to hex string
const hexString: string = fromHex.toHex(); // "0x1234567890abcdef..."

// Access individual bytes (returns UInt8)
const firstByte = fromHex.bytes[0];

// Provable equality
fromHex.assertEquals(Bytes32.fromHex('0x1234567890abcdef...'));
```

**Common Use Cases:**

1. **Hash Function Inputs:**
```typescript
import { Hash } from 'o1js';

class Bytes64 extends Bytes(64) {}

@method async verifyHashPreimage(preimage: Bytes64, expectedHash: Field) {
  // Hash the byte array
  const hash = Hash.SHA3_256.hash(preimage);
  hash.assertEquals(expectedHash);
}
```

2. **Cross-Chain Signature Verification (with Foreign Fields):**
```typescript
import { Bytes, createForeignCurve, Crypto } from 'o1js';

// Bitcoin/Ethereum signatures are 64 bytes (r=32, s=32)
class Secp256k1Signature extends Bytes(64) {}
class Secp256k1 extends createForeignCurve(Crypto.CurveParams.Secp256k1) {}

@method async verifyBitcoinSignature(
  sig: Secp256k1Signature,
  msgHash: Bytes32,
  pubKey: Secp256k1
) {
  // Verify ECDSA signature on foreign curve
  const isValid = pubKey.verifySignature(msgHash, sig);
  isValid.assertTrue('Invalid Bitcoin signature');
}
```

3. **Binary Protocol Data:**
```typescript
class Bytes128 extends Bytes(128) {}

@method async processBinaryPayload(payload: Bytes128) {
  // Extract header (first 16 bytes)
  const header = payload.bytes.slice(0, 16);

  // Extract body (remaining bytes)
  const body = payload.bytes.slice(16);

  // Parse specific fields
  const version = header[0]; // UInt8
  const flags = header[1];   // UInt8
}
```

**Performance Characteristics:**

- **Fixed Size Required**: Size must be compile-time constant (cannot use runtime variables)
- **Constraint Cost**: Each byte adds ~8 constraints (for UInt8 range check)
- **Optimal Sizes**: Use smallest size needed (e.g., Bytes32 for hashes, not Bytes128)

**Integration with Foreign Curves:**

Bytes are essential for cross-chain interoperability:

```typescript
// Verify Ethereum signature in Mina zkApp
import { Bytes, ForeignCurve, Ecdsa } from 'o1js';

class Secp256k1 extends createForeignCurve(Crypto.CurveParams.Secp256k1) {}
class Signature extends Bytes(64) {}

const ethAddress = Secp256k1.from({ x: Field(...), y: Field(...) });
const signature = Signature.fromHex('0x...');
const messageHash = Bytes32.fromHex('0x...');

// Verify in circuit
Ecdsa.verify(Secp256k1, signature, messageHash, ethAddress);
```

**When to Use:**
- ✅ Cross-chain signature verification (Bitcoin, Ethereum)
- ✅ Hash function preimages and inputs
- ✅ Binary protocol data parsing
- ✅ Cryptographic key material
- ❌ Don't use for variable-length data (use CircuitString or Field arrays)
- ❌ Don't use for large data (consider Merkle tree roots instead)

### String Handling with CircuitString

**AI Agent Note**: `CircuitString` is a fixed-size provable string type (max 128 characters). Uses null-terminated character arrays internally. Supports UTF-8 and ASCII encodings. Source: `o1js/src/lib/provable/string.ts:76-250`

```typescript
import { CircuitString, Character, Field } from 'o1js';

// Creating strings (max 128 characters)
const str = CircuitString.fromString("Hello, Mina!");

// String operations
const char = str.charAt(0); // Returns Character (Field wrapper)
const length = str.length(); // Returns Field (dynamic length)

// Character extraction
const firstChar = str.values[0]; // Character type
const charField = firstChar.toField(); // Get Field representation

// String comparison
str.equals(CircuitString.fromString("Hello, Mina!")); // Returns Bool
str.assertEquals(CircuitString.fromString("Hello, Mina!")); // Throws if not equal

// String concatenation (provably fits within maxLength)
const str1 = CircuitString.fromString("Hello, ");
const str2 = CircuitString.fromString("Mina!");
const combined = str1.append(str2); // "Hello, Mina!"

// Substring extraction
const sub = str.substring(0, 5); // "Hello"

// Hashing (Poseidon hash of all characters)
const hash = str.hash(); // Field
```

**Encoding Support:**

```typescript
// ASCII (default, backward compatible)
CircuitString.encoding = 'ascii';
const ascii = CircuitString.fromString("Hello");

// UTF-8 (supports full Unicode)
CircuitString.encoding = 'utf-8';
const utf8 = CircuitString.fromString("Hëllö 世界");

// Convert to JavaScript string
const jsString = str.toString('utf-8'); // or 'ascii'
```

**Common Patterns:**

1. **Username/Label Validation:**
```typescript
@method async registerUser(username: CircuitString) {
  // Length constraints
  const len = username.length();
  len.assertGreaterThan(Field(3));  // Min 3 chars
  len.assertLessThan(Field(20));    // Max 20 chars

  // Store hash on-chain (more efficient than full string)
  const usernameHash = username.hash();
  this.usernameRegistry.set(usernameHash);
}
```

**Important Limitations:**

- **Fixed Buffer**: Always allocates 128 characters internally (even if string is shorter)
- **Constraint Cost**: ~16 constraints per character (range check for 16-bit char codes)
- **Total Cost**: ~2048 constraints for full 128-character string
- **No Dynamic Allocation**: Cannot create CircuitString with custom max length at runtime

**Performance Notes:**

- `length()` computes dynamic length by counting non-null characters (O(n) operation)
- `append()` uses `Provable.switch` internally (expensive for long strings)
- Consider storing `Poseidon.hash(str)` on-chain instead of full string

**When to Use:**
- ✅ Short labels, usernames, identifiers
- ✅ Fixed-format messages (e.g., "VOTED_ON:proposalID")
- ✅ Hash preimages for human-readable data
- ❌ Don't use for long text (>50 chars) - constraint cost grows linearly
- ❌ Don't use for variable-format data - consider Field arrays with custom encoding

### Optional Values with Option<T>

**AI Agent Note**: `Option<T>` is a factory function that creates optional/nullable versions of provable types, similar to Rust's `Option<T>` or TypeScript's `T | undefined`. Essential for handling missing data in circuits. Source: `o1js/src/lib/provable/option.ts:34-121`

```typescript
import { Option, Field, UInt64, Bool } from 'o1js';

// Create an optional type
class OptionUInt64 extends Option(UInt64) {}
class OptionField extends Option(Field) {}

// Creating optional values
const some = OptionUInt64.from(5n);        // Has a value
const none = OptionUInt64.none();          // No value
const fromUndefined = OptionUInt64.from(undefined); // Same as none()

// Accessing values safely
const value: UInt64 = some.assertSome('must have a value'); // Throws if none
const withDefault: UInt64 = none.orElse(UInt64.from(100)); // Returns 100

// Checking if value exists
some.isSome.assertTrue();  // Passes
none.isSome.assertFalse(); // Passes

// Asserting absence
none.assertNone('must be empty'); // Passes
```

**Common Patterns:**

1. **Optional Smart Contract Parameters:**
```typescript
class OptionField extends Option(Field) {}

@method async processPayment(
  amount: UInt64,
  referrer: OptionField  // Optional referrer code
) {
  // Handle with default
  const referralBonus = referrer.orElse(Field(0));

  // Or require it
  const code = referrer.assertSome('Referrer required for this tier');
}
```

2. **Nullable State Updates:**
```typescript
class OptionUInt64 extends Option(UInt64) {}

@method async updatePrice(newPrice: OptionUInt64) {
  // Only update if provided
  Provable.if(
    newPrice.isSome,
    () => {
      this.price.set(newPrice.assertSome());
    },
    () => {
      // Keep existing price
    }
  );
}
```

3. **Search Results / Lookups:**
```typescript
class OptionPublicKey extends Option(PublicKey) {}

@method async findOwner(tokenId: Field): OptionPublicKey {
  // Search in Merkle tree
  const owner = this.searchMerkleTree(tokenId);

  if (ownerExists.toBoolean()) {
    return OptionPublicKey.from(owner);
  } else {
    return OptionPublicKey.none();
  }
}
```

**How Option<T> Works Under the Hood:**

```typescript
// Option is a struct with two fields
type Option<T> = {
  isSome: Bool,      // true if value exists
  value: T           // Always present, but ignored if isSome=false
}

// When isSome=false, value is a dummy "synthesized" value
// This ensures constant constraint count regardless of Some/None
```

**Key Methods:**

- `Option.from(value?)`: Create Some(value) or None if undefined
- `Option.none()`: Create None explicitly
- `option.assertSome(message?)`: Extract value, throw if None
- `option.assertNone(message?)`: Assert empty, throw if Some
- `option.orElse(defaultValue)`: Get value or default (uses `Provable.if`)
- `option.isSome`: Bool indicating presence

**Performance Characteristics:**

- **Constant Constraints**: Option<T> always allocates space for `T`, even if None
- **Cost**: `sizeof(T) + 1 Bool` (1 constraint for Bool)
- **No Runtime Branching**: Uses `Provable.if` for conditional logic

**When to Use:**
- ✅ Optional method parameters
- ✅ Nullable return values from lookups
- ✅ Conditional state updates
- ✅ Search results (found vs not found)
- ❌ Don't use for "large" types (consider using a separate existence Bool + conditional loading)
- ❌ Don't use if you always have a sensible default (just use the default)

### Cryptographic Types

```typescript
// Key pairs and signatures
const privateKey = PrivateKey.random();
const publicKey = privateKey.toPublicKey();

// Signature creation and verification
const message = [Field(1), Field(2), Field(3)];
const signature = Signature.create(privateKey, message);
const isValid = signature.verify(publicKey, message);

// Group elements (elliptic curve points)
const group = Group.from(x, y); // x, y are Field elements
const scalar = Scalar.random();
const scaledGroup = group.scale(scalar);

// Hash functions
const hash = Poseidon.hash([Field(1), Field(2), Field(3)]);
const pedersenHash = Pedersen.hash(Group.generator.scale(Scalar.from(123)));

// ForeignField operations (Added in 2.9.0)
// For interoperability with other cryptographic systems (secp256k1, BN254, etc.)
const foreignValue = Field(12345);

// Unsafe constructor for advanced use cases (2.9.0+)
const foreignField = ForeignField.Unsafe.fromField(foreignValue);

// Safe operations with foreign fields
const safeForeignField = ForeignField.from(12345);
const foreignSum = safeForeignField.add(ForeignField.from(67890));

// Nullifier - for anonymous commitments and privacy
import { Nullifier } from 'o1js';

// Nullifier is typically created from mina-signer output
// Used for anonymous voting, preventing double-spending, etc.
const nullifierJson = minaSignerResult.nullifier;
const nullifier = Nullifier.fromJSON(nullifierJson);
```

### Nullifier: Anonymous Account Commitments

**Location in o1js**: `src/lib/provable/crypto/nullifier.ts` (line 114)

**RFC**: https://github.com/o1-labs/o1js/issues/756
**Paper**: https://eprint.iacr.org/2022/1255.pdf

**Purpose**: Nullifiers provide anonymous account commitments for privacy-preserving applications like anonymous voting, preventing double-spending, and maintaining consistent identity across anonymous actions.

#### **What is a Nullifier?**

A nullifier is a cryptographic primitive that:
1. **Proves knowledge** of a private key without revealing it
2. **Binds to a specific message** (e.g., vote ID, spending record)
3. **Provides a unique key** that can be stored in a Merkle tree to prevent reuse
4. **Maintains anonymity** - observers cannot link nullifiers to public keys

#### **Nullifier Structure**

```typescript
import { Nullifier, Group, Scalar, Field } from 'o1js';

// Nullifier contains:
class NullifierStructure extends Struct({
  publicKey: Group,          // Public key commitment
  public: {
    nullifier: Group,        // The actual nullifier (public)
    s: Scalar,               // Challenge response
  },
  private: {
    c: Field,                // Challenge (private during proof)
    g_r: Group,              // Randomness commitment 1
    h_m_pk_r: Group          // Randomness commitment 2
  }
}) {}
```

#### **Creating Nullifiers**

**Note**: Nullifiers are typically created by `mina-signer` (in wallet/client code), not directly in o1js contracts.

```typescript
// Client-side (mina-signer):
// Wallet creates nullifier for a message
let privateKey = PrivateKey.random();
let messageFields = [voteId, proposalHash];

// mina-signer creates the nullifier (off-chain)
let nullifierJson = createNullifier(messageFields, privateKey);

// On-chain: Contract receives and verifies
class VotingContract extends SmartContract {
  @method async castVote(
    nullifier: Nullifier,
    voteChoice: Field
  ) {
    // Verify nullifier is valid for this vote
    let voteId = this.voteId.getAndRequireEquals();
    nullifier.verify([voteId]);  // Throws if invalid

    // Check nullifier hasn't been used
    let nullifierKey = nullifier.key();  // Unique key per (message, pubkey)
    let isUsed = this.nullifierSet.get(nullifierKey);
    isUsed.assertEquals(Bool(false), 'Already voted');

    // Mark nullifier as used
    this.nullifierSet.set(nullifierKey, Bool(true));

    // Record vote (without revealing voter identity)
    this.recordVote(voteChoice);
  }
}
```

#### **Nullifier API Methods**

**1. Verify**: Prove nullifier belongs to specific message
```typescript
@method async verifyNullifier(nullifier: Nullifier, message: Field[]) {
  // Verifies:
  // 1. Nullifier was created by someone with the private key
  // 2. Nullifier is for this specific message
  // 3. Challenge-response is correct
  nullifier.verify(message);  // Throws if invalid
}
```

**2. Key**: Get unique identifier for Merkle tree storage
```typescript
// Get nullifier key for indexing
let key: Field = nullifier.key();

// Use as index in Merkle Map to track used nullifiers
class NullifierSet {
  private map = new MerkleMap();

  isUsed(nullifier: Nullifier): Bool {
    let key = nullifier.key();
    return this.map.get(key).equals(Field(1));
  }

  markUsed(nullifier: Nullifier) {
    let key = nullifier.key();
    this.map.set(key, Field(1));
  }
}
```

**3. GetPublicKey**: Extract public key commitment
```typescript
// Get public key from nullifier (Group element)
let publicKey: Group = nullifier.getPublicKey();

// Can be used for additional checks
let pkHash = Poseidon.hash(Group.toFields(publicKey));
```

**4. FromJSON / ToJSON**: Serialization
```typescript
// From JSON (from mina-signer)
let nullifier = Nullifier.fromJSON(jsonData);

// To JSON (for storage/transmission)
let json = Nullifier.toJSON(nullifier);
```

#### **Common Patterns**

**1. Anonymous Voting**
```typescript
class AnonymousVoting extends SmartContract {
  @state(Field) nullifierSetRoot = State<Field>();
  @state(Field) voteTally = State<Field>();

  @method async vote(
    nullifier: Nullifier,
    vote: Bool,  // true = yes, false = no
    nullifierWitness: MerkleMapWitness
  ) {
    // Verify nullifier for this election
    let electionId = Field(42);
    nullifier.verify([electionId]);

    // Check not already used
    let key = nullifier.key();
    let currentRoot = this.nullifierSetRoot.getAndRequireEquals();
    let [witnessRoot, witnessKey] = nullifierWitness.computeRootAndKey(Field(0));
    witnessRoot.assertEquals(currentRoot);
    witnessKey.assertEquals(key);

    // Mark as used
    let [newRoot] = nullifierWitness.computeRootAndKey(Field(1));
    this.nullifierSetRoot.set(newRoot);

    // Tally vote (anonymously)
    let tally = this.voteTally.getAndRequireEquals();
    this.voteTally.set(Provable.if(vote, tally.add(1), tally));
  }
}
```

**2. Double-Spend Prevention**
```typescript
class PrivateToken extends SmartContract {
  @method async spend(
    amount: UInt64,
    nullifier: Nullifier,  // Prevents spending same UTXO twice
    spendWitness: MerkleMapWitness
  ) {
    // Message includes UTXO identifier
    let utxoId = this.deriveUtxoId(amount);
    nullifier.verify([utxoId]);

    // Check UTXO hasn't been spent
    let key = nullifier.key();
    // ... check in nullifier set ...

    // Process spend
    this.processSpend(amount);
  }
}
```

**3. Anonymous Identity Across Actions**
```typescript
// User can prove they're the same person across multiple actions
// without revealing their identity

class ReputationSystem extends SmartContract {
  @method async earnPoints(nullifier: Nullifier, points: UInt64) {
    let systemId = this.systemId.getAndRequireEquals();
    nullifier.verify([systemId]);

    // Same nullifier = same user (without revealing who)
    let userId = nullifier.key();
    this.addPoints(userId, points);
  }

  @method async spendPoints(nullifier: Nullifier, points: UInt64) {
    let systemId = this.systemId.getAndRequireEquals();
    nullifier.verify([systemId]);

    // Same user can spend their accumulated points
    let userId = nullifier.key();
    this.deductPoints(userId, points);
  }
}
```

#### **Security Considerations**

**⚠️ CRITICAL**:
1. **Message Binding**: Always verify nullifier with the exact same message fields used during creation
2. **Uniqueness Check**: Always check nullifier hasn't been used before (via Merkle tree/map)
3. **Key Derivation**: Use `nullifier.key()` for indexing - it's the hash of the nullifier group element
4. **Client-Side Creation**: Nullifiers must be created in wallet/client code with mina-signer, not in contracts

```typescript
// ❌ WRONG: Different message than creation
nullifier.verify([wrongVoteId]);  // Will fail

// ❌ WRONG: Not checking reuse
@method async vote(nullifier: Nullifier) {
  nullifier.verify([voteId]);
  // Missing: check if nullifier.key() already used
}

// ✅ CORRECT: Proper pattern
@method async vote(nullifier: Nullifier, witness: MerkleMapWitness) {
  nullifier.verify([voteId]);  // Correct message
  let key = nullifier.key();
  this.requireNotUsed(key, witness);  // Check not reused
  this.markUsed(key, witness);
}
```

#### **When to Use Nullifiers**

**✅ Use For:**
- Anonymous voting systems
- Private token transfers (UTXO model)
- Double-spend prevention
- Reputation systems with privacy
- Anonymous but consistent identity

**❌ Avoid For:**
- Simple authentication (use Signature instead)
- Public actions (no privacy needed)
- Non-anonymous voting
- Cases where you need to reveal the voter

### Foreign Fields and Curves: Cross-Chain Interoperability

**Location in o1js**: `src/index.ts:8-16`, `src/lib/provable/foreign-field.ts`, `src/lib/provable/crypto/foreign-curve.ts`

**Purpose**: Enable cryptographic operations with non-native field arithmetic (like secp256k1, secp256r1, BN254) for cross-chain interoperability.

#### **Foreign Field Types**

```typescript
import {
  createForeignField,
  ForeignField,
  AlmostForeignField,
  CanonicalForeignField
} from 'o1js';

// 1. Standard ForeignField - general purpose
const field = ForeignField.from(12345n);

// 2. AlmostForeignField - values almost in range [0, modulus)
// Used for intermediate calculations
const almostField = AlmostForeignField.from(field);

// 3. CanonicalForeignField - guaranteed in range [0, modulus)
// Used for final results that must be canonical
const canonicalField = CanonicalForeignField.from(field);

// Create custom foreign field for a specific modulus
class Secp256k1Field extends createForeignField(
  // secp256k1 field modulus
  0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffefffffc2fn
) {}

// Use in provable code
const a = Secp256k1Field.from(100n);
const b = Secp256k1Field.from(200n);
const sum = a.add(b);  // Foreign field addition
const product = a.mul(b);  // Foreign field multiplication
```

#### **Foreign Curves for ECDSA**

```typescript
import { createForeignCurve, createEcdsa, EcdsaSignature } from 'o1js';

// Define secp256k1 curve
class Secp256k1 extends createForeignCurve(
  Crypto.CurveParams.Secp256k1
) {}

// Create ECDSA signature verification
class Secp256k1ECDSA extends createEcdsa(Secp256k1) {}

// Verify Ethereum/Bitcoin signatures in a zkApp
class CrossChainVerifier extends SmartContract {
  @method async verifyEthereumSignature(
    message: Bytes,
    signature: EcdsaSignature,
    publicKey: Secp256k1.AlmostForeignField
  ) {
    // Verify ECDSA signature from Ethereum
    const isValid = signature.verify(Secp256k1ECDSA, message, publicKey);
    isValid.assertTrue('Invalid Ethereum signature');
  }
}
```

#### **Common Foreign Field Patterns**

**1. Bitcoin/Ethereum Signature Verification**
```typescript
import { Bytes } from 'o1js';

// Verify Bitcoin/Ethereum transaction signature
@method async verifyBitcoinTx(
  txHash: Bytes,
  signature: EcdsaSignature,
  pubKey: Secp256k1.AlmostForeignField
) {
  signature.verify(Secp256k1ECDSA, txHash, pubKey).assertTrue();
}
```

**2. BN254 Pairing for Groth16 Verification**
```typescript
// Define BN254 curve (used in many zkSNARKs)
class BN254Field extends createForeignField(
  // BN254 field modulus
  21888242871839275222246405745257275088548364400416034343698204186575808495617n
) {}

class BN254 extends createForeignCurve(
  Crypto.CurveParams.BN254
) {}

// Verify Groth16 proofs from other systems
```

**3. Range-Constrained Foreign Field Ops**
```typescript
// Almost field: for intermediate computations
let intermediate: AlmostForeignField = a.mul(b);  // May overflow

// Canonical field: for final results (auto range-checked)
let result: CanonicalForeignField = intermediate.assertCanonical();
// Proves result ∈ [0, modulus)
```

#### **Performance Considerations**

```typescript
// ⚠️ Foreign field ops are EXPENSIVE (100-1000x native Field ops)

// Native Mina field (FAST)
let a = Field(100);
let b = Field(200);
let sum = a.add(b);  // ~1 constraint

// Foreign field (SLOW)
let x = Secp256k1Field.from(100n);
let y = Secp256k1Field.from(200n);
let foreignSum = x.add(y);  // ~50-100 constraints

// Minimize foreign field operations
// ❌ BAD: Many foreign field ops
for (let i = 0; i < 10; i++) {
  result = result.add(Secp256k1Field.from(i));
}

// ✅ GOOD: Accumulate in native field, convert once
let nativeSum = Field(0);
for (let i = 0; i < 10; i++) {
  nativeSum = nativeSum.add(Field(i));
}
result = Secp256k1Field.from(nativeSum.toBigInt());
```

#### **When to Use Foreign Fields**

**✅ Use For:**
- Cross-chain signature verification (Ethereum, Bitcoin)
- Verifying proofs from other zkSNARK systems (Groth16, etc.)
- Interoperability with non-Mina blockchains
- Implementing standard cryptographic algorithms (ECDSA, EdDSA)

**❌ Avoid For:**
- General arithmetic (use native `Field` instead)
- Mina-native operations (always use Mina's field)
- Performance-critical inner loops
- Simple calculations that don't require foreign field semantics

### Encryption: Public Key Encryption in Circuits

**AI Agent Note**: `Encryption` namespace provides public-key encryption/decryption using Diffie-Hellman key exchange + Poseidon sponge. Useful for encrypted on-chain messages. Source: `o1js/src/lib/provable/crypto/encryption.ts:10-150`

```typescript
import { Encryption, PrivateKey, PublicKey, Field } from 'o1js';

// Encrypt a message (Field array)
const senderPrivateKey = PrivateKey.random();
const recipientPublicKey = PrivateKey.random().toPublicKey();
const message = [Field(100), Field(200), Field(300)];

const cipherText = Encryption.encrypt(message, recipientPublicKey);
// Returns: { publicKey: Group, cipherText: Field[] }

// Decrypt the message
const recipientPrivateKey = recipientPublicKey; // (in practice, recipient's private key)
const decrypted = Encryption.decrypt(cipherText, recipientPrivateKey);
// Returns: Field[] (original message)
```

**How It Works:**

1. **Key Exchange**: Uses Diffie-Hellman on Mina's curve (ECDH)
2. **Key Derivation**: Poseidon sponge absorbs shared secret
3. **Encryption**: Stream cipher using Poseidon output as keystream
4. **Authentication**: Includes authentication tag (prevents tampering)

**Binary Data Encryption:**

```typescript
import { Encryption, Bytes } from 'o1js';

class Bytes32 extends Bytes(32) {}

// Encrypt binary data
const messageBytes = Bytes32.fromHex('0x123456...');
const cipherTextBytes = Encryption.encryptBytes(
  messageBytes,
  recipientPublicKey
);

// Decrypt binary data
const decryptedBytes = Encryption.decryptBytes(
  cipherTextBytes,
  recipientPrivateKey
);
```

**Common Use Cases:**

```typescript
@method async sendEncryptedMessage(
  recipient: PublicKey,
  message: Field
) {
  // Encrypt message to recipient
  const encrypted = Encryption.encrypt([message], recipient);

  // Emit encrypted data (stored on-chain publicly, but only recipient can decrypt)
  this.emitEvent('EncryptedMessage', encrypted);
}
```

**When to Use:**
- ✅ Private messages between users (on-chain storage)
- ✅ Encrypted state commitments
- ✅ Secret sharing protocols
- ❌ Don't use for large data (expensive in constraints)
- ❌ Don't use if off-chain encryption suffices (use standard crypto off-chain)

**Performance Note**: Encryption/decryption is moderately expensive (~50-100 constraints per Field element). Consider encrypting only sensitive data.

### Encoding: Field Serialization Utilities

**AI Agent Note**: `Encoding` namespace provides low-level field encoding/decoding utilities. Rarely used directly (most types have built-in serialization). Source: `o1js/bindings/lib/encoding.ts`

```typescript
import { Encoding, Field } from 'o1js';

// Encode bigints to fields
const bigints = [100n, 200n, 300n];
const fields = Encoding.fieldsEncode(bigints);

// Decode fields to bigints
const decoded = Encoding.fieldsDecode(fields);
```

**When to Use:**
- ✅ Low-level protocol development
- ✅ Custom serialization formats
- ❌ Most users should use built-in `.toFields()` / `.fromFields()` methods

## Advanced Type Composition

### Structs: Custom Data Types

```typescript
// Define custom struct
class Vote extends Struct({
  hasVoted: Bool,
  inFavor: Bool,
  voteCount: UInt64,
  voterAddress: PublicKey,
}) {
  // Static factory methods
  static default(): Vote {
    return new Vote({
      hasVoted: Bool(false),
      inFavor: Bool(false),
      voteCount: UInt64.zero,
      voterAddress: PublicKey.empty(),
    });
  }

  // Custom methods
  castVote(inFavor: Bool): Vote {
    return new Vote({
      hasVoted: Bool(true),
      inFavor,
      voteCount: this.voteCount.add(1),
      voterAddress: this.voterAddress,
    });
  }

  // Validation methods
  validateVote(): void {
    this.hasVoted.assertTrue();
    this.voteCount.assertGreaterThan(UInt64.zero);
  }
}

// Usage in provable code
const vote = Vote.default();
const updatedVote = vote.castVote(Bool(true));
updatedVote.validateVote();
```

### Arrays: Fixed-Size Collections

```typescript
// Fixed-size array definition
const FieldArray3 = Provable.Array(Field, 3);
const UInt64Array10 = Provable.Array(UInt64, 10);

// Array creation and manipulation
const fieldArray = FieldArray3.from([Field(1), Field(2), Field(3)]);

// Witness pattern for dynamic array creation
const dynamicArray = Provable.witness(FieldArray3, () => {
  // Compute array values off-circuit
  return [
    Field(computeValue1()),
    Field(computeValue2()),
    Field(computeValue3()),
  ];
});

// Array iteration (must be fixed-size)
for (let i = 0; i < 3; i++) {
  const element = fieldArray[i];
  element.assertGreaterThan(Field(0));
}

// Array operations
const sum = fieldArray.reduce((acc, curr) => acc.add(curr), Field(0));
```

### Dynamic Arrays (Limited Support)

```typescript
// DynamicArray: Variable length up to maximum size
const dynamicArray = DynamicArray.from([Field(1), Field(2), Field(3)]);

// Operations
dynamicArray.push(Field(4));
const popped = dynamicArray.pop();
const length = dynamicArray.length.get();

// Iteration over dynamic array
for (let i = 0; i < dynamicArray.maxLength; i++) {
  const element = dynamicArray.get(UInt32.from(i));
  // Element is option type - might be undefined
  const isValid = element.isSome;
  const value = element.value;
}
```

## The Witness Pattern: Advanced Programming Technique
**How constraints get generated under the hood:** o1js executes your TypeScript twice whenever you call `SmartContract.compile()` or `ZkProgram.compile()`. The first pass runs in "compile" mode, recording every provable operation into an AST that the bindings turn into Kimchi gates. The second pass runs in "prove" mode, reusing the frozen circuit and hooking real witness values into each gate. The compiler reports constraint and key metadata as part of its return value, so any branch that fails to execute during the compile pass never becomes part of the circuit.

### Method Metadata and Caching

`SmartContract.compile()` analyzes every `@method` and caches both the prover key and verification key to disk. Subsequent runs reuse cached keys when the circuit hash matches, saving minutes of recompilation. You can introspect a method's static footprint by calling `SmartContract.analyzeMethods()` or `ZkProgram.analyzeMethods()` which returns rows, number of gates, public input size, and whether the method relies on lookups or recursive proofs (`o1js/src/lib/proof-system/zkprogram.ts:1`). In CI, keeping the cache folder warm dramatically improves `npm run build` times and ensures verification keys are consistent across developers.

### Provable, FlexibleProvable, and Core Namespace

The `Provable` namespace defines canonical encodings between TypeScript objects and field elements. Higher-level wrappers like `Struct`, `CircuitString`, `Provable.Array`, and `DynamicArray` build on it, while `FlexibleProvablePure` lets you admit optional serialization strategies (for example the reducer API accepts any `FlexibleProvablePure` action type). For low-level work, the new `Core` namespace exposes raw field and curve operations, transaction layouts, and protocol constants without the syntactic sugar; it maps almost one-to-one to the underlying OCaml bindings (`o1js/src/index.ts:106`).

### Witness and As-Prover Discipline

`Provable.witness()` spawns a temporary region where you can compute complex values in JavaScript yet still enforce consistency by asserting relationships afterward. `Provable.asProver()` does *not* add constraints; it is intended for logging, debugging, or deriving auxiliary data (for example writing to off-chain storage). Any logic that must hold on-chain must happen outside the `asProver` block. This distinction is the root of common under-constrained proof bugs—see the security discussion in `docs2/docs/zkapps/writing-a-zkapp/introduction-to-zkapps/secure-zkapps.mdx:1`.

### Runtime Tables, Lookups, and Foreign Fields

Kimchi supports lookup tables and foreign-field arithmetic, which o1js surfaces through provable APIs. The **`RuntimeTable` class** (o1js 2.9.0+) provides an ergonomic API for runtime-defined lookup tables with automatic batching—use it for membership proofs of (index, value) pairs defined at circuit construction time (see Part 4 for full details). Note: the old `Gates.addRuntimeTableConfig()` and `Gadgets.inTable()` APIs are deprecated and will be removed.

Foreign field helpers (`ForeignField.*`, `Gadgets.ForeignField.*`) allow implementing BN254 or secp256k1 arithmetic by decomposing values into base-field limbs (`o1js/src/lib/provable/foreign-field.ts:1`). These tools trade constraint count for flexibility but must be wired carefully—lookups and foreign-field operations balloon proving time if overused.

```typescript
import { RuntimeTable, Field } from 'o1js';

// ✅ Modern API (2.9.0+)
const table = new RuntimeTable(5, [0n, 1n, 2n]);
table.insert([[0n, Field(10)], [1n, Field(20)]]);
table.lookup(0n, Field(10));
table.check(); // flush pending lookups

// ❌ Deprecated - DO NOT USE
// Gates.addRuntimeTableConfig(...);
// Gadgets.inTable(...);
```

### Proof-System Resource Metrics

Every provable method has three main cost drivers: number of rows (Kimchi gates), number of lookups, and number of public inputs. `analyzeMethods()` and the debugging hooks in `Circuit.logConstraintCount()` surface these metrics so teams can monitor regression budgets. In production, aim to keep single-method row counts under ~200k for reasonable proving latency on commodity hardware; beyond that, consider splitting logic into multiple methods or using recursion to aggregate proofs.


**Critical AI Agent Knowledge**: The witness pattern is essential for efficient circuit design. Use this to explain performance optimization and debugging constraint issues.

### Basic Witness Usage

The witness pattern allows complex computations outside the circuit while proving the result inside:

```typescript
function efficientSquareRoot(y: Field): Field {
  return Provable.witness(Field, () => {
    // Complex computation happens off-circuit (JavaScript execution)
    const yBigInt = y.toBigInt();
    const sqrtBigInt = computeSquareRoot(yBigInt); // Expensive operation
    return Field(sqrtBigInt);
  }).let((x) => {
    // Prove the result is correct in-circuit (constraint generation)
    x.mul(x).assertEquals(y); // Only this constraint is added to circuit
    return x;
  });
}

// Usage
const input = Field(16);
const sqrt = efficientSquareRoot(input); // Returns Field(4)
```

### Unconstrained Values: Passing Data Between Witness Blocks

**AI Agent Note**: `Unconstrained<T>` is a container for passing unconstrained values between out-of-circuit blocks (`Provable.witness()`, `Provable.asProver()`). Values can ONLY be accessed in prover context, not in provable code. Source: `o1js/src/lib/provable/types/unconstrained.ts:39-141`

```typescript
import { Unconstrained, Field, Provable } from 'o1js';

// Create an unconstrained value
let counter = Unconstrained.from(0n);

class MyContract extends SmartContract {
  @method async myMethod(accumulator: Unconstrained<bigint>) {
    // ❌ ILLEGAL: Cannot access in provable code
    // let value = accumulator.get(); // Throws error!

    // ✅ CORRECT: Access only in witness blocks
    Provable.witness(Field, () => {
      let current = accumulator.get();
      let newValue = current + 1n;
      accumulator.set(newValue);
      return Field(newValue);
    });

    // ✅ Also works in asProver blocks
    Provable.asProver(() => {
      console.log('Current count:', accumulator.get());
    });
  }
}
```

**Key Characteristics:**

- **Zero Constraint Cost**: `Unconstrained<T>` contributes 0 fields to the proof
- **Prover-Only Access**: Can only call `.get()` in `Provable.witness()` or `Provable.asProver()` blocks
- **Pass-Through Container**: Used to thread state between multiple witness computations
- **No Guarantees**: Values are not checked or constrained by the circuit

**Common Use Cases:**

1. **Stateful Witness Computations:**
```typescript
@method async processSequence(items: Field[]) {
  // Track witness state across iterations
  let runningHash = Unconstrained.from(Field(0));

  for (let i = 0; i < items.length; i++) {
    const item = items[i];

    Provable.witness(Field, () => {
      // Access previous hash from unconstrained storage
      let prevHash = runningHash.get();

      // Compute new hash off-circuit
      let newHash = Poseidon.hash([prevHash, item]);

      // Update for next iteration
      runningHash.set(newHash);

      return newHash;
    });
  }
}
```

2. **Accumulating Witness Data:**
```typescript
@method async verifyMultipleMerkleProofs(proofs: MerkleWitness[]) {
  // Accumulate proof verification count
  let verifiedCount = Unconstrained.from(0);

  proofs.forEach((proof) => {
    const root = Provable.witness(Field, () => {
      // Increment count in witness
      verifiedCount.set(verifiedCount.get() + 1);
      return proof.calculateRoot();
    });

    // Verify on-circuit
    root.assertEquals(this.merkleRoot.get());
  });

  // Log final count (prover-only)
  Provable.asProver(() => {
    console.log(`Verified ${verifiedCount.get()} proofs`);
  });
}
```

**API Methods:**

- `Unconstrained.from(value)`: Create from value
- `Unconstrained.witness(compute)`: Create from witness computation
- `unconstrained.get()`: Read value (prover context only)
- `unconstrained.set(value)`: Update value
- `unconstrained.setTo(other)`: Copy from another Unconstrained
- `unconstrained.updateAsProver(fn)`: Update via function in prover block

**Important Limitations:**

- **No Provable Guarantees**: Unconstrained values are not part of the proof
- **Prover-Only**: Cannot be used in constraint logic
- **Compilation Caveat**: May be empty during compilation, only populated at proving time

**When to Use:**
- ✅ Threading state between multiple `Provable.witness()` calls
- ✅ Accumulating metadata during witness generation
- ✅ Passing auxiliary data to prover blocks
- ✅ Debugging/logging in `Provable.asProver()`
- ❌ Don't use for constrained values (use regular provable types)
- ❌ Don't access in provable code (will throw runtime error)

### Advanced Witness Patterns

```typescript
// Witness for complex data structures
function witnessComplexComputation(inputs: Field[]): {
  result: Field;
  proof: Field[];
} {
  return Provable.witness(
    Provable.Struct({ result: Field, proof: Provable.Array(Field, 5) }),
    () => {
      // Off-circuit computation
      const complexResult = performComplexAlgorithm(
        inputs.map((f) => f.toBigInt())
      );
      return {
        result: Field(complexResult.value),
        proof: complexResult.proofData.map((p) => Field(p)),
      };
    }
  ).let((witnessed) => {
    // In-circuit verification
    verifyComplexResult(inputs, witnessed.result, witnessed.proof);
    return witnessed;
  });
}

// Conditional witness based on circuit state
function conditionalWitness(condition: Bool, input: Field): Field {
  return Provable.witness(Field, () => {
    // Note: This executes regardless of condition value
    // The condition only affects circuit constraints
    if (condition.toBoolean()) {
      return computeExpensiveOperation(input.toBigInt());
    } else {
      return 0;
    }
  }).let((result) => {
    // Conditional constraint application
    const constrainedResult = Provable.if(
      condition,
      result, // Use witnessed result if condition is true
      Field(0) // Use zero if condition is false
    );
    return constrainedResult;
  });
}
```

## Conditional Logic and Control Flow

**AI Agent Warning**: Traditional `if/else` statements don't work in circuits. Always guide users to use `Provable.if()` for conditional logic.

### Provable Conditionals

```typescript
// Basic conditional value selection
const condition = Bool(true);
const valueA = Field(100);
const valueB = Field(200);
const result = Provable.if(condition, valueA, valueB);

// Conditional with different types
const numberResult = Provable.if(condition, UInt64.from(100), UInt64.from(200));
const boolResult = Provable.if(condition, Bool(true), Bool(false));

// Complex conditional logic
function complexConditional(x: Field, y: Field): Field {
  const isPositive = x.greaterThan(Field(0));
  const isEven = x.mod(2).equals(Field(0));

  return Provable.if(
    isPositive.and(isEven),
    x.mul(y),
    Provable.if(isPositive, x.add(y), x.sub(y))
  );
}
```

### Switch-Case Pattern with Provable.switch()

**Location in o1js**: `src/lib/provable/provable.ts:156-165`

**Purpose**: Select one value from multiple options based on a mask (like a switch statement).

```typescript
import { Provable, Bool, Field } from 'o1js';

// Provable.switch() - mask-based multi-case selection
// mask: array of Bools with exactly ONE true element
// cases: array of values to choose from
const mask = [Bool(false), Bool(true), Bool(false)];
const cases = [Field(10), Field(20), Field(30)];
const result = Provable.switch(mask, Field, cases);
// Returns Field(20) (corresponding to the true mask element)

// Common pattern: convert selector to mask
function selectByIndex(selector: Field, cases: Field[]): Field {
  // Create mask where only selector index is true
  const mask = cases.map((_, i) => selector.equals(Field(i)));
  return Provable.switch(mask, Field, cases);
}

// Usage
const index = Field(2);
const options = [Field(100), Field(200), Field(300)];
const selected = selectByIndex(index, options);  // Returns Field(300)
```

#### **Provable.switch() vs Nested Provable.if()**

```typescript
// ❌ INEFFICIENT: Nested Provable.if() (old pattern)
function oldSwitchCase(selector: Field, cases: Field[]): Field {
  let result = cases[0];
  for (let i = 1; i < cases.length; i++) {
    const condition = selector.equals(Field(i));
    result = Provable.if(condition, cases[i], result);
  }
  return result;
}
// Constraints: O(n) comparisons + O(n) conditional selections

// ✅ EFFICIENT: Provable.switch() (modern pattern)
function newSwitchCase(selector: Field, cases: Field[]): Field {
  const mask = cases.map((_, i) => selector.equals(Field(i)));
  return Provable.switch(mask, Field, cases);
}
// Constraints: O(n) comparisons + O(1) switch operation (more efficient)
```

#### **Common Use Cases**

**1. State Machine Transitions**
```typescript
enum State { Idle, Processing, Complete, Error }

function getNextState(currentState: Field, event: Field): Field {
  const states = [
    Field(State.Idle),
    Field(State.Processing),
    Field(State.Complete),
    Field(State.Error)
  ];

  const mask = states.map(s => currentState.equals(s).and(event.equals(Field(1))));
  return Provable.switch(mask, Field, [
    Field(State.Processing),  // Idle + event → Processing
    Field(State.Complete),    // Processing + event → Complete
    Field(State.Idle),        // Complete + event → Idle
    Field(State.Error)        // Error + event → Error (stay)
  ]);
}
```

**2. Multi-Tier Pricing**
```typescript
function calculatePrice(tier: Field): UInt64 {
  // tier: 0 = basic, 1 = pro, 2 = enterprise
  const mask = [
    tier.equals(Field(0)),
    tier.equals(Field(1)),
    tier.equals(Field(2))
  ];

  return Provable.switch(mask, UInt64, [
    UInt64.from(10),   // Basic: $10
    UInt64.from(50),   // Pro: $50
    UInt64.from(200)   // Enterprise: $200
  ]);
}
```

**3. Dynamic Function Selection**
```typescript
function applyOperation(op: Field, a: Field, b: Field): Field {
  // op: 0 = add, 1 = sub, 2 = mul
  const results = [
    a.add(b),
    a.sub(b),
    a.mul(b)
  ];

  const mask = [
    op.equals(Field(0)),
    op.equals(Field(1)),
    op.equals(Field(2))
  ];

  return Provable.switch(mask, Field, results);
}
```

#### **Mask Requirements and Safety**

```typescript
// ⚠️ CRITICAL: Mask must have EXACTLY ONE true element

// ✅ VALID: One true element
const validMask = [Bool(false), Bool(true), Bool(false)];
const result = Provable.switch(validMask, Field, cases);

// ❌ INVALID: Multiple true elements (undefined behavior)
const invalidMask = [Bool(true), Bool(true), Bool(false)];
// Don't do this - circuit may not constrain properly

// ❌ INVALID: All false (undefined behavior)
const allFalseMask = [Bool(false), Bool(false), Bool(false)];
// Don't do this - no value will be selected

// ✅ SAFE: Generate mask programmatically
function safeMask(selector: Field, numCases: number): Bool[] {
  return Array.from({ length: numCases }, (_, i) =>
    selector.equals(Field(i))
  );
  // Guarantees exactly one true if selector ∈ [0, numCases)
}
```

#### **Performance Comparison**

| Pattern | Constraints | When to Use |
|---------|-------------|-------------|
| `Provable.if()` | ~2-3 per call | 2 cases (binary choice) |
| Nested `Provable.if()` | ~O(n) | Legacy code, <5 cases |
| `Provable.switch()` | ~O(n) + optimized | 3+ cases (recommended) |

**Recommendation**: Always use `Provable.switch()` for 3+ cases. It's more readable and optimized for multi-case selection.

### Loop Patterns and Constraints

```typescript
// Fixed iteration count (required for circuit consistency)
function fixedLoopSum(array: Field[]): Field {
  let sum = Field(0);

  // Must be compile-time constant
  for (let i = 0; i < array.length; i++) {
    sum = sum.add(array[i]);
  }

  return sum;
}

// Conditional execution within fixed loops
function conditionalSum(array: Field[], condition: Bool): Field {
  let sum = Field(0);

  for (let i = 0; i < array.length; i++) {
    // Each iteration adds either array[i] or 0
    const addend = Provable.if(condition, array[i], Field(0));
    sum = sum.add(addend);
  }

  return sum;
}

// Early termination pattern (all iterations still execute)
function findFirstMatch(
  array: Field[],
  target: Field
): { found: Bool; index: UInt32 } {
  let found = Bool(false);
  let foundIndex = UInt32.zero;

  for (let i = 0; i < array.length; i++) {
    const isMatch = array[i].equals(target).and(found.not());
    found = found.or(isMatch);
    foundIndex = Provable.if(isMatch, UInt32.from(i), foundIndex);
  }

  return { found, index: foundIndex };
}
```

## Data Structures and Collections

### Merkle Trees: Efficient Provable Storage

```typescript
// Basic Merkle Tree
const height = 8; // Tree can hold 2^8 = 256 leaves
const tree = new MerkleTree(height);

// Add leaves to tree
tree.setLeaf(0n, Field(1));
tree.setLeaf(1n, Field(2));
tree.setLeaf(2n, Field(3));

// Generate witness for provable operations
const witness = tree.getWitness(1n);
const root = tree.getRoot();

// Verify leaf in circuit
function verifyLeafInclusion(
  leaf: Field,
  index: Field,
  witness: MerkleWitness,
  expectedRoot: Field
): Bool {
  const calculatedRoot = witness.calculateRoot(leaf);
  return calculatedRoot.equals(expectedRoot);
}
```

### IndexedMerkleMap: Key-Value Storage

**Note**: IndexedMerkleMap was promoted from `Experimental` namespace to public API in o1js 2.7.0

```typescript
// IndexedMerkleMap for key-value storage (max height 52)
const map = new IndexedMerkleMap(40); // 2^40 possible keys

// Set key-value pairs
map.set(Field(123), Field(456));
map.set(Field(789), Field(101112));

// Get values and witnesses
const value = map.get(Field(123)); // Returns Field(456)
const witness = map.getWitness(Field(123));
const root = map.getRoot();

// Provable map operations
function proveMapUpdate(
  key: Field,
  oldValue: Field,
  newValue: Field,
  witness: any, // MerkleMapWitness type
  oldRoot: Field
): Field {
  // Verify old value was in map
  const calculatedOldRoot = witness.calculateRoot(key, oldValue);
  calculatedOldRoot.assertEquals(oldRoot);

  // Calculate new root with updated value
  const newRoot = witness.calculateRoot(key, newValue);
  return newRoot;
}
```

### MerkleList: Dynamic Lists

```typescript
// MerkleList: Dynamic list with single hash commitment
class NumberList extends MerkleList.create(Field, Field.empty(), Field) {}

// Create and manipulate list
const list = NumberList.empty();
list.push(Field(1));
list.push(Field(2));
list.push(Field(3));

// Get list properties
const length = list.length;
const commitment = list.commitment; // Single Field representing entire list

// Provable list operations
function proveListAppend(list: NumberList, newElement: Field): NumberList {
  const newList = list.clone();
  newList.push(newElement);

  // Prove the list was extended correctly
  newList.length.assertEquals(list.length.add(1));

  return newList;
}
```

## Hashing and Cryptographic Operations

### Poseidon Hash: ZK-Friendly Hash Function

```typescript
// Single field hashing
const hash1 = Poseidon.hash([Field(1)]);

// Multiple field hashing
const hash2 = Poseidon.hash([Field(1), Field(2), Field(3)]);

// Hash large data structures
function hashComplexData(vote: Vote): Field {
  return Poseidon.hash([
    vote.hasVoted.toField(),
    vote.inFavor.toField(),
    vote.voteCount.value,
    ...vote.voterAddress.toFields(),
  ]);
}

// Hash chaining for large datasets
function hashArray(fields: Field[]): Field {
  let hash = Field(0);

  // Process in chunks for efficiency
  for (let i = 0; i < fields.length; i += 4) {
    const chunk = fields.slice(i, i + 4);
    // Pad chunk if necessary
    while (chunk.length < 4) {
      chunk.push(Field(0));
    }
    hash = Poseidon.hash([hash, ...chunk]);
  }

  return hash;
}
```

### Pedersen Hash: Alternative Hash Function

```typescript
// Pedersen hash with Group elements
const group = Group.generator;
const scalar = Scalar.from(123);
const pedersenHash = Pedersen.hash(group.scale(scalar));

// Hash Fields to Group
const fields = [Field(1), Field(2), Field(3)];
const groupHash = Pedersen.hashToGroup(fields);
```

### HMAC Implementation Pattern

```typescript
// HMAC-SHA256 using o1js (simplified example)
function hmacSha256(key: UInt8[], message: UInt8[]): UInt8[] {
  const HMAC_BLOCK_SIZE = 64;

  // Key preparation (compile-time optimization)
  let processedKey: UInt8[];
  if (key.length > HMAC_BLOCK_SIZE) {
    // Only add SHA-256 constraints for long keys
    processedKey = sha256Hash(key);
  } else {
    // Use padding-only constraints for short keys
    processedKey = padKey(key, HMAC_BLOCK_SIZE);
  }

  // HMAC computation
  const innerKey = processedKey.map((k) => k.xor(UInt8.from(0x36)));
  const outerKey = processedKey.map((k) => k.xor(UInt8.from(0x5c)));

  const innerHash = sha256Hash([...innerKey, ...message]);
  const finalHash = sha256Hash([...outerKey, ...innerHash]);

  return finalHash;
}
```

## Transaction Fees and Error Handling

**Critical AI Agent Knowledge**: Mina has a unique fee system and error handling patterns that differ from other blockchains.

### **Fee System in Mina**

```typescript
import { Mina, PrivateKey, UInt64 } from "o1js";

// Mina uses a constant fee structure
const fee = UInt64.from(100_000_000); // 0.1 MINA (in nanomina)

// Transaction with explicit fee
const tx = await Mina.transaction(
  {
    sender: senderAccount,
    fee: fee, // Optional - defaults to reasonable fee
    memo: "My zkApp transaction", // Optional transaction memo
  },
  () => {
    contract.myMethod(args);
  }
);

// Common fee patterns
const STANDARD_FEE = UInt64.from(100_000_000); // 0.1 MINA
const PRIORITY_FEE = UInt64.from(1_000_000_000); // 1 MINA for faster processing
```

### **Error Handling Patterns**

```typescript
// 1. Assertion Errors (Most Common)
class ValidatedContract extends SmartContract {
  @method async withdraw(amount: UInt64) {
    const balance = this.account.balance.getAndRequireEquals();

    // This creates a constraint - transaction fails if false
    amount.assertLessThanOrEqual(balance);

    // Alternative with custom error context
    amount.assertGreaterThan(UInt64.zero, 'Amount must be positive');

    // Send with proper validation
    this.send({ to: this.sender, amount });
  }
}

// 2. Precondition Failures
@method async restrictedMethod() {
  // This creates a precondition on the account state
  const currentNonce = this.account.nonce.getAndRequireEquals();

  // If account nonce changes between proof generation and validation,
  // transaction will fail with precondition error
}

// 3. Network State Errors
try {
  const account = await Mina.getAccount(publicKey);
  console.log('Account exists:', account);
} catch (error) {
  if (error.message.includes('Account not found')) {
    console.log('Account does not exist yet');
    // Handle new account case
  }
}
```

### **Complete Transaction Lifecycle with Error Handling**

```typescript
export class TransactionManager {
  static async executeTransaction(
    contract: SmartContract,
    method: string,
    args: any[],
    options: {
      fee?: UInt64;
      sender?: PrivateKey;
      memo?: string;
      retries?: number;
    } = {}
  ): Promise<{ success: boolean; txHash?: string; error?: string }> {
    const {
      fee = UInt64.from(100_000_000),
      sender = PrivateKey.random(), // Should be provided
      memo = "",
      retries = 3,
    } = options;

    const senderAccount = sender.toPublicKey();

    for (let attempt = 0; attempt < retries; attempt++) {
      try {
        // 1. Create transaction
        console.log(`Attempt ${attempt + 1}: Creating transaction...`);

        const tx = await Mina.transaction(
          {
            sender: senderAccount,
            fee,
            memo,
          },
          () => {
            // Call the contract method dynamically
            (contract as any)[method](...args);
          }
        );

        // 2. Generate proof
        console.log("Generating proof...");
        await tx.prove();

        // 3. Sign transaction
        console.log("Signing transaction...");
        const signedTx = tx.sign([sender]);

        // 4. Send transaction
        console.log("Sending transaction...");
        const pendingTx = await signedTx.send();

        // 5. Wait for confirmation
        console.log("Waiting for confirmation...");
        await this.waitForConfirmation(pendingTx.hash);

        return {
          success: true,
          txHash: pendingTx.hash,
        };
      } catch (error) {
        console.log(`Attempt ${attempt + 1} failed:`, error.message);

        // Check if it's a retryable error
        if (this.isRetryableError(error)) {
          if (attempt < retries - 1) {
            console.log("Retrying in 5 seconds...");
            await new Promise((resolve) => setTimeout(resolve, 5000));
            continue;
          }
        }

        return {
          success: false,
          error: error.message,
        };
      }
    }

    return {
      success: false,
      error: "Max retries exceeded",
    };
  }

  private static isRetryableError(error: any): boolean {
    const retryableErrors = [
      "Account_precondition_unsatisfied",
      "Network_error",
      "Timeout",
      "Invalid_nonce",
    ];

    return retryableErrors.some((retryableError) =>
      error.message.includes(retryableError)
    );
  }

  private static async waitForConfirmation(txHash: string): Promise<void> {
    const maxAttempts = 60; // 10 minutes with 10-second intervals

    for (let i = 0; i < maxAttempts; i++) {
      try {
        const status = await Mina.getTransactionStatus(txHash);

        if (status === "INCLUDED") {
          console.log("Transaction confirmed!");
          return;
        } else if (status === "REJECTED") {
          throw new Error("Transaction was rejected");
        }

        // Transaction still pending
        await new Promise((resolve) => setTimeout(resolve, 10000));
      } catch (error) {
        if (i === maxAttempts - 1) {
          throw new Error("Transaction confirmation timeout");
        }

        // Continue waiting
        await new Promise((resolve) => setTimeout(resolve, 10000));
      }
    }
  }
}

// Usage example with comprehensive error handling
async function safeContractInteraction() {
  try {
    const result = await TransactionManager.executeTransaction(
      myContract,
      "transfer",
      [recipientAddress, UInt64.from(1000)],
      {
        fee: UInt64.from(200_000_000), // Higher fee for priority
        sender: myPrivateKey,
        memo: "Token transfer",
        retries: 5,
      }
    );

    if (result.success) {
      console.log("Transaction successful:", result.txHash);
    } else {
      console.error("Transaction failed:", result.error);
    }
  } catch (error) {
    console.error("Unexpected error:", error);
  }
}
```

### **Common Error Types and Solutions**

```typescript
// Error patterns AI agents should handle
export const MinaErrorPatterns = {
  INSUFFICIENT_BALANCE: {
    pattern: /insufficient.*balance/i,
    solution: "Check account balance before transaction",
    code: `
const balance = await Mina.getBalance(account);
if (balance.lessThan(amount.add(fee))) {
  throw new Error('Insufficient balance');
}`,
  },

  PRECONDITION_FAILED: {
    pattern: /precondition.*failed/i,
    solution: "Account state changed during proof generation",
    code: `
// Use getAndRequireEquals() carefully
const balance = this.account.balance.getAndRequireEquals();
// Consider using get() if you can handle state changes
const balance = this.account.balance.get();`,
  },

  INVALID_PROOF: {
    pattern: /invalid.*proof/i,
    solution: "Proof generation failed or circuit constraints not satisfied",
    code: `
// Ensure all assertions are satisfiable
amount.assertGreaterThan(UInt64.zero);
// Check constraint system
const analysis = Provable.constraintSystem(() => {
  myMethod(testInputs);
});`,
  },

  NONCE_MISMATCH: {
    pattern: /nonce.*mismatch/i,
    solution: "Account nonce changed between transactions",
    code: `
// Fetch fresh account state
const account = await Mina.getAccount(publicKey);
const currentNonce = account.nonce;
// Wait for previous transaction to confirm before sending next`,
  },
};
```

## Serialization and Encoding

### Field Packing and Unpacking

```typescript
// Pack multiple small values into single Field
function packUInt8Array(values: UInt8[]): Field {
  let packed = Field(0);
  let shift = Field(1);

  for (let i = 0; i < values.length; i++) {
    packed = packed.add(values[i].value.mul(shift));
    shift = shift.mul(256); // 2^8 for UInt8
  }

  return packed;
}

// Unpack Field into multiple UInt8 values
function unpackToUInt8Array(packed: Field, count: number): UInt8[] {
  const values: UInt8[] = [];
  let remaining = packed;

  for (let i = 0; i < count; i++) {
    const value = remaining.mod(256);
    values.push(UInt8.from(value));
    remaining = remaining.div(256);
  }

  return values;
}
```

### Packed<T>: Optimized Field Packing

**AI Agent Note**: `Packed<T>` automatically packs types with many small fields (Bools, UInt32s) into fewer Field elements. Reduces constraint count for operations like `Provable.if()`. Source: `o1js/src/lib/provable/packed.ts:14-146`

```typescript
import { Packed, Bool, UInt32, Struct } from 'o1js';

// Example: struct with many small fields
class MyStruct extends Struct({
  flag1: Bool,     // 1 bit
  flag2: Bool,     // 1 bit
  flag3: Bool,     // 1 bit
  counter: UInt32, // 32 bits
  id: UInt32,      // 32 bits
  // Total: 67 bits -> can fit in 1 Field (254 bits)
}) {}

// Without packing: 5 Field elements (1 per field)
// With packing: 1 Field element (all packed together)

// Create a packed version
const PackedStruct = Packed.create(MyStruct);

// Pack a value
const myValue = new MyStruct({ flag1: Bool(true), flag2: Bool(false), ... });
const packed = PackedStruct.pack(myValue);

// Packed operations (more efficient)
const result = Provable.if(
  condition,
  PackedStruct,
  packed1,  // Only 1 Field
  packed2   // Only 1 Field
  // Cost: O(1) constraints vs O(5) unpacked
);

// Unpack when you need the original value
const unpacked: MyStruct = result.unpack();
```

**Key Benefits:**

- **Reduced Constraint Cost**: `Provable.if(bool, x, y)` costs O(n) where n = number of Fields
  - Unpacked MyStruct: 5 Fields = 5× constraint cost
  - Packed MyStruct: 1 Field = 1× constraint cost
- **Storage Efficiency**: Fewer Fields stored on-chain
- **Automatic Packing**: No manual bit manipulation required

**When Packing Helps:**

✅ **Good candidates for packing:**
- Multiple Bools (each 1 bit)
- Multiple UInt8/UInt32/UInt64 (small integers)
- Structs with many small fields
- Values used in `Provable.if()` or `Provable.switch()`

❌ **Bad candidates for packing:**
- Single Bool or UInt32 (no size reduction)
- Large types (PublicKey, Field arrays)
- Types already at max Field size

**Important Limitations:**

- **Safety Warning**: Only use with types that have proper `check()` methods
- **Packing Overhead**: Unpacking requires constraints to verify correct packing
- **One-Time Optimization**: Best used when value is packed once, used many times

### Hashed<T>: Represent Values by Hash

**AI Agent Note**: `Hashed<T>` represents a type by its Poseidon hash (1 Field). Reduces `Provable.if()` to O(1) constraints. Trade-off: pay hash computation cost. Source: `o1js/src/lib/provable/packed.ts:165-270`

```typescript
import { Hashed, Poseidon, PublicKey, Struct } from 'o1js';

// Example: large struct (many Fields)
class UserProfile extends Struct({
  publicKey: PublicKey,  // 2 Fields
  username: CircuitString, // 128 Fields
  balance: UInt64,         // 1 Field
  reputation: UInt32,      // 1 Field
  // Total: 132 Fields
}) {}

// Without hashing: 132 Field elements
// With hashing: 1 Field element (hash)

// Create a hashed version
const HashedProfile = Hashed.create(UserProfile);

// Hash a value
const profile = new UserProfile({ ... });
const hashed = HashedProfile.hash(profile);

// Hashed operations (O(1) constraints)
const selectedProfile = Provable.if(
  condition,
  HashedProfile,
  hashed1,  // Only 1 Field (hash)
  hashed2   // Only 1 Field (hash)
  // Cost: O(1) constraints vs O(132) unhashed!
);

// Unhash when you need the original (verifies hash matches)
const original: UserProfile = selectedProfile.unhash();
```

**Key Benefits:**

- **Constant Constraint Cost**: `Provable.if()` becomes O(1) regardless of type size
- **Massive Savings for Large Types**: 132 Fields → 1 Field = 132× reduction
- **Commitment Pattern**: Hash commits to value, unhash reveals and verifies

**Performance Trade-offs:**

```typescript
// Cost analysis
const unhashed = Provable.if(bool, UserProfile, a, b);
// Cost: O(132) constraints for conditional

const hashed = Provable.if(bool, HashedProfile, hashA, hashB);
const result = hashed.unhash();
// Cost: O(1) for conditional + O(hash) for unhash
// Total: ~50-100 constraints (much better!)
```

**Common Use Cases:**

1. **Conditional Large Structs:**
```typescript
@method async selectWinner(condition: Bool, user1: Hashed<UserProfile>, user2: Hashed<UserProfile>) {
  // O(1) constraint cost for selection
  const winner = Provable.if(condition, HashedProfile, user1, user2);

  // Unhash only the winner (pay hash verification once)
  const profile = winner.unhash();
  this.winnerHash.set(winner.hash);
}
```

2. **Merkle Tree Leaves:**
```typescript
// Store hash in Merkle tree, unhash on demand
const leaf = HashedProfile.hash(userProfile);
tree.setLeaf(index, leaf.hash);
```

**When to Use:**
- ✅ Large structs (>10 Fields) in conditional logic
- ✅ Repeated use of same value in `Provable.if()`
- ✅ Commitment/reveal patterns
- ❌ Don't use for already-small types (1-2 Fields)
- ❌ Don't use if you never need to unhash (just use `Poseidon.hash()`)

**Custom Hash Functions:**

```typescript
// Provide custom hash (e.g., for compatibility)
const HashedType = Hashed.create(
  MyType,
  (value) => Poseidon.hash([value.field1, value.field2]) // Custom hash
);
```

### Object Serialization Patterns

```typescript
// Convert complex objects to/from Fields for storage
class SerializableVote extends Struct({
  hasVoted: Bool,
  inFavor: Bool,
  voteCount: UInt64,
  voterAddress: PublicKey,
}) {
  // Serialize to Field array
  toFields(): Field[] {
    return [
      this.hasVoted.toField(),
      this.inFavor.toField(),
      this.voteCount.value,
      ...this.voterAddress.toFields(),
    ];
  }

  // Deserialize from Field array
  static fromFields(fields: Field[]): SerializableVote {
    return new SerializableVote({
      hasVoted: Bool.fromFields([fields[0]]),
      inFavor: Bool.fromFields([fields[1]]),
      voteCount: UInt64.fromFields([fields[2]]),
      voterAddress: PublicKey.fromFields(fields.slice(3, 5)),
    });
  }

  // Compute hash for storage in merkle structures
  hash(): Field {
    return Poseidon.hash(this.toFields());
  }
}
```

## Performance Analysis and Optimization

**AI Agent Debugging Guide**: Use this section to help users analyze and optimize circuit performance. High constraint counts are the primary performance issue.

### Constraint System Analysis

```typescript
// Analyze constraint count for optimization
function analyzeConstraints() {
  const analysis = Provable.constraintSystem(() => {
    // Your provable code here
    const x = Field(1);
    const y = Field(2);
    const z = x.add(y).mul(Field(3));
    z.assertEquals(Field(9));
  });

  console.log("Constraint Summary:", analysis.summary);
  console.log("Total Constraints:", analysis.numConstraints);
  console.log("Constraint Details:", analysis.constraints);
}

// Compare different implementations
function compareImplementations() {
  const impl1 = Provable.constraintSystem(() => {
    // Implementation 1
    const result = expensiveMethod1();
  });

  const impl2 = Provable.constraintSystem(() => {
    // Implementation 2
    const result = expensiveMethod2();
  });

  console.log("Implementation 1 constraints:", impl1.numConstraints);
  console.log("Implementation 2 constraints:", impl2.numConstraints);
}
```

### Optimization Techniques

```typescript
// Use appropriate types for range checking
function optimizeRangeChecks() {
  // ❌ Less efficient - Field requires explicit range checks
  const inefficient = Field(100);
  inefficient.assertLessThan(Field(1000));

  // ✅ More efficient - UInt32 has built-in range checking
  const efficient = UInt32.from(100);
  efficient.assertLessThan(UInt32.from(1000));
}

// Batch operations when possible
function batchOperations(values: Field[]): Field {
  // ❌ Less efficient - multiple hash calls
  let result = Field(0);
  for (const value of values) {
    result = result.add(Poseidon.hash([value]));
  }

  // ✅ More efficient - single hash call
  const batchedHash = Poseidon.hash(values);
  return batchedHash;
}

// Minimize state reads/writes
function optimizeStateAccess() {
  // ❌ Multiple state reads
  const state1 = this.someState.get();
  const state2 = this.someState.get(); // Redundant read

  // ✅ Single state read
  const state = this.someState.get();
  // Use 'state' variable multiple times
}
```

## ZkFunction API: Simple Circuit Creation (New in 2.8.0)

### Introduction to ZkFunction

**ZkFunction** is a new API introduced in o1js 2.8.0 that provides a simpler alternative to the deprecated `Circuit` API. It creates proofs with Kimchi directly (bypassing Pickles), making it much faster but without recursion support.

**Key Characteristics:**

- **Faster proving**: Direct Kimchi proof generation
- **No recursion**: Cannot be used in recursive proof composition
- **Not zkApp compatible**: Proofs cannot be verified by Mina zkApps
- **Simpler API**: More ergonomic than the old Circuit API

### Basic ZkFunction Usage

```typescript
import { Field, Experimental } from "o1js";
const { ZkFunction } = Experimental;

// Define a simple ZkFunction
const simpleFunction = ZkFunction({
  name: "simple-computation",
  publicInputType: Field,
  privateInputTypes: [Field],
  main: (publicInput: Field, privateInput: Field) => {
    // Simple computation: square the private input and assert equals public
    const result = privateInput.mul(privateInput);
    result.assertEquals(publicInput);
  },
});

// Compile the function
const { verificationKey } = await simpleFunction.compile();

// Create a proof (proving we know square root of 25)
const publicInput = Field(25);
const privateInput = Field(5); // Secret square root
const proof = await simpleFunction.prove(publicInput, privateInput);

// Verify the proof
const isValid = await simpleFunction.verify(proof, verificationKey);
console.log("Proof valid:", isValid); // true
```

### ZkFunction with Complex Logic

```typescript
import { Field, Bool, Experimental, Provable, Gadgets } from "o1js";
const { ZkFunction } = Experimental;

// More complex ZkFunction with range checks
const conditionalFunction = ZkFunction({
  name: "conditional-computation",
  publicInputType: Field,
  privateInputTypes: [Field, Field],
  main: (threshold: Field, value: Field, multiplier: Field) => {
    // Ensure inputs are within valid range
    Gadgets.rangeCheck64(value);
    Gadgets.rangeCheck64(multiplier);

    // Check if value is above threshold
    const isAbove = value.greaterThan(threshold);

    // Conditional computation
    const result = Provable.if(
      isAbove,
      value.mul(multiplier), // If above threshold, multiply
      value.add(Field(1)) // If below, just add 1
    );

    // The result is constrained but not directly returned
    // ZkFunction proves the computation was done correctly
  },
});

// Usage
const { verificationKey } = await conditionalFunction.compile();

const proof = await conditionalFunction.prove(
  Field(5), // threshold (public)
  Field(10), // value (private)
  Field(3) // multiplier (private)
);

const isValid = await conditionalFunction.verify(proof, verificationKey);
console.log("Proof valid:", isValid); // true
```

### ZkFunction vs ZkProgram Comparison

| Feature                 | ZkFunction           | ZkProgram                    |
| ----------------------- | -------------------- | ---------------------------- |
| **Proving Speed**       | Fast (direct Kimchi) | Slower (through Pickles)     |
| **Recursion Support**   | ❌ No                | ✅ Yes                       |
| **zkApp Compatibility** | ❌ No                | ✅ Yes                       |
| **API Complexity**      | Simple               | More complex                 |
| **Use Case**            | Standalone proofs    | zkApp integration, recursion |

### When to Use ZkFunction

**Choose ZkFunction for:**

- Standalone proof generation
- Performance-critical applications
- Simple computational proofs
- Non-recursive use cases
- Applications outside Mina ecosystem

**Choose ZkProgram for:**

- zkApp development
- Recursive proof composition
- Complex multi-method circuits
- Mina Protocol integration

### Performance Benefits

```typescript
// Benchmark example
const benchmarkFunction = ZkFunction(
  {
    name: "benchmark",
    publicInput: Field,
    publicOutput: Field,
  },
  (x: Field) => {
    let result = x;
    for (let i = 0; i < 100; i++) {
      result = result.add(Field(1));
    }
    return result;
  }
);

// Compile once
await benchmarkFunction.compile();

// Measure proving time
const startTime = Date.now();
const proof = await benchmarkFunction.prove(Field(0));
const provingTime = Date.now() - startTime;

console.log(`Proving time: ${provingTime}ms`);
console.log(`Result: ${proof.publicOutput.toString()}`); // "100"
```

### Migration from Circuit API

**Old Circuit API (Deprecated):**

```typescript
// ❌ Deprecated - don't use
import { Circuit, Field } from "o1js";

const oldCircuit = Circuit.withSecret([Field], (x) => {
  return x.mul(x);
});
```

**New ZkFunction API:**

```typescript
// ✅ Use this instead
import { Field, Experimental } from "o1js";
const { ZkFunction } = Experimental;

const newFunction = ZkFunction({
  name: "square",
  publicInputType: Field,
  privateInputTypes: [Field],
  main: (publicSquare: Field, privateInput: Field) => {
    privateInput.mul(privateInput).assertEquals(publicSquare);
  },
});
```

## Core Namespace: Low-Level Bindings (Unreleased)

### Introduction to Core Namespace

**Note**: The `Core` namespace is an **unreleased feature** in the latest o1js development that exposes low-level bindings and protocol constants for advanced users.

```typescript
import { Core } from "o1js";

// Access to internal protocol constants, hashes, and prefixes
// This provides direct access to the underlying protocol layer
// Use with caution - these are advanced internal APIs
```

### When to Use Core Namespace

**Intended for:**

- Advanced protocol developers
- Custom cryptographic implementations
- Deep integration with Mina Protocol internals
- Performance-critical applications requiring low-level access

**Not recommended for:**

- General application development
- Standard zkApp development
- Learning o1js basics

### Expected Core Features

Based on the changelog, the Core namespace will likely expose:

- **Protocol Constants**: Network parameters and consensus constants
- **Hash Functions**: Internal hash algorithms and configurations
- **Prefix Values**: Transaction and proof prefixes
- **Binding Functions**: Direct access to OCaml bindings

**Warning**: This is an advanced API that may change before official release. Use only for experimental development.

This completes Part 2 covering the o1js framework in detail. Next we'll dive into smart contract development patterns and zkApp-specific implementations.
