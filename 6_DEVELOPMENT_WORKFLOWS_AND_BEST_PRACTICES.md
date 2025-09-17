# Part 6: Development Workflows and Best Practices - Complete Development Guide

## Complete Development Environment Setup

### zkApp CLI and Project Initialization

```bash
# Install zkApp CLI globally
npm install -g zkapp-cli

# Create new zkApp project
zk project my-zkapp
cd my-zkapp

# Project structure created:
# ├── src/
# │   ├── Add.ts          # Example contract
# │   ├── Add.test.ts     # Tests
# │   ├── interact.ts     # Interaction script
# │   ├── index.ts        # Main exports
# │   └── run.ts          # Deployment script
# ├── config.json         # Deployment configuration
# ├── keys/              # Generated keys
# ├── build/             # Compiled output
# └── package.json

# Development commands
npm run build          # Compile TypeScript
npm run buildw         # Build in watch mode
npm run test          # Run Jest tests
npm run testw         # Run tests in watch mode
npm run coverage      # Generate coverage report
```

### Advanced Project Structure for Production

```
my-production-zkapp/
├── src/
│   ├── contracts/           # Smart contracts
│   │   ├── base/           # Base contract classes
│   │   ├── tokens/         # Token contracts
│   │   ├── governance/     # Governance contracts
│   │   └── utils/          # Contract utilities
│   ├── programs/           # ZkPrograms
│   │   ├── recursive/      # Recursive programs
│   │   └── verification/   # Verification programs
│   ├── lib/               # Shared libraries
│   │   ├── merkle/        # Merkle tree implementations
│   │   ├── crypto/        # Cryptographic utilities
│   │   └── types/         # Custom type definitions
│   ├── tests/             # Comprehensive test suites
│   │   ├── unit/          # Unit tests
│   │   ├── integration/   # Integration tests
│   │   └── e2e/           # End-to-end tests
│   └── scripts/           # Deployment and interaction scripts
├── config/                # Environment configurations
│   ├── development.json
│   ├── staging.json
│   └── production.json
├── docs/                 # Documentation
├── .github/              # CI/CD workflows
└── docker/              # Docker configurations
```

### Environment Configuration Management

```typescript
// config/ConfigManager.ts
interface NetworkConfig {
  networkId: string;
  name: string;
  mina: string;
  archive?: string;
  explorer?: string;
  faucet?: string;
}

interface DeploymentConfig {
  network: NetworkConfig;
  deployerPrivateKey: string;
  contractAddresses: Record<string, string>;
  gasSettings: {
    fee: string;
    memo: string;
  };
}

class ConfigManager {
  private configs: Map<string, DeploymentConfig> = new Map();

  constructor() {
    this.loadConfigs();
  }

  private loadConfigs(): void {
    // Load from config files
    const environments = ['development', 'staging', 'production'];

    environments.forEach(env => {
      try {
        const config = require(`../config/${env}.json`);
        this.configs.set(env, config);
      } catch (error) {
        console.warn(`Config file for ${env} not found`);
      }
    });
  }

  getConfig(environment: string): DeploymentConfig {
    const config = this.configs.get(environment);
    if (!config) {
      throw new Error(`Configuration for ${environment} not found`);
    }
    return config;
  }

  getCurrentConfig(): DeploymentConfig {
    const env = process.env.NODE_ENV || 'development';
    return this.getConfig(env);
  }

  async setupNetwork(environment: string): Promise<void> {
    const config = this.getConfig(environment);

    const network = Mina.Network({
      mina: config.network.mina,
      archive: config.network.archive,
    });

    Mina.setActiveInstance(network);

    console.log(`Connected to ${config.network.name}`);
  }
}

// Usage example
const configManager = new ConfigManager();
await configManager.setupNetwork('development');
```

## Comprehensive Testing Strategy

### Unit Testing with Advanced Patterns

```typescript
// tests/unit/TokenContract.test.ts
import { TokenContract } from '../../src/contracts/tokens/TokenContract';
import { Field, UInt64, PrivateKey, PublicKey, AccountUpdate, Mina } from 'o1js';

describe('TokenContract', () => {
  let tokenContract: TokenContract;
  let deployerKey: PrivateKey;
  let deployerAccount: PublicKey;
  let userKey: PrivateKey;
  let userAccount: PublicKey;

  beforeAll(async () => {
    // Compile contract once for all tests
    console.log('Compiling TokenContract...');
    await TokenContract.compile();
  });

  beforeEach(async () => {
    // Setup fresh local blockchain for each test
    const Local = Mina.LocalBlockchain({ proofsEnabled: true });
    Mina.setActiveInstance(Local);

    // Setup accounts
    deployerKey = Local.testAccounts[0].privateKey;
    deployerAccount = deployerKey.toPublicKey();
    userKey = Local.testAccounts[1].privateKey;
    userAccount = userKey.toPublicKey();

    // Deploy contract
    const contractKey = PrivateKey.random();
    const contractAddress = contractKey.toPublicKey();
    tokenContract = new TokenContract(contractAddress);

    await localDeploy(contractKey, deployerKey);
  });

  describe('Token Minting', () => {
    it('should mint tokens to specified address', async () => {
      const mintAmount = UInt64.from(1000);

      const tx = await Mina.transaction(deployerAccount, () => {
        tokenContract.mint(userAccount, mintAmount);
      });
      await tx.prove();
      await tx.send();

      // Verify token balance
      const balance = Mina.getBalance(userAccount, tokenContract.token.id);
      expect(balance).toEqual(mintAmount);
    });

    it('should update total supply when minting', async () => {
      const mintAmount = UInt64.from(500);
      const initialSupply = tokenContract.totalSupply.get();

      const tx = await Mina.transaction(deployerAccount, () => {
        tokenContract.mint(userAccount, mintAmount);
      });
      await tx.prove();
      await tx.send();

      const newSupply = tokenContract.totalSupply.get();
      expect(newSupply).toEqual(initialSupply.add(mintAmount));
    });

    it('should fail when non-owner tries to mint', async () => {
      const mintAmount = UInt64.from(100);

      await expect(async () => {
        const tx = await Mina.transaction(userAccount, () => {
          tokenContract.mint(userAccount, mintAmount);
        });
        await tx.prove();
        await tx.send();
      }).rejects.toThrow();
    });
  });

  describe('Token Transfers', () => {
    beforeEach(async () => {
      // Mint tokens to user for transfer tests
      const tx = await Mina.transaction(deployerAccount, () => {
        tokenContract.mint(userAccount, UInt64.from(1000));
      });
      await tx.prove();
      await tx.send();
    });

    it('should transfer tokens between accounts', async () => {
      const recipientKey = PrivateKey.random();
      const recipientAccount = recipientKey.toPublicKey();
      const transferAmount = UInt64.from(200);

      const tx = await Mina.transaction(userAccount, () => {
        AccountUpdate.fundNewAccount(userAccount); // Fund new account
        tokenContract.transfer(userAccount, recipientAccount, transferAmount);
      });
      await tx.prove();
      await tx.send();

      // Verify balances
      const senderBalance = Mina.getBalance(userAccount, tokenContract.token.id);
      const recipientBalance = Mina.getBalance(recipientAccount, tokenContract.token.id);

      expect(senderBalance).toEqual(UInt64.from(800));
      expect(recipientBalance).toEqual(transferAmount);
    });
  });

  // Helper function for contract deployment
  async function localDeploy(contractKey: PrivateKey, deployerKey: PrivateKey): Promise<void> {
    const tx = await Mina.transaction(deployerKey, () => {
      AccountUpdate.fundNewAccount(deployerKey);
      tokenContract.deploy({ zkappKey: contractKey });
    });
    await tx.prove();
    await tx.send();
  }
});
```

### Integration Testing with Cross-Contract Interactions

```typescript
// tests/integration/DEX.integration.test.ts
import { DEXContract } from '../../src/contracts/DEXContract';
import { TokenContract } from '../../src/contracts/tokens/TokenContract';
import { LiquidityPoolContract } from '../../src/contracts/LiquidityPoolContract';

describe('DEX Integration Tests', () => {
  let dex: DEXContract;
  let tokenA: TokenContract;
  let tokenB: TokenContract;
  let liquidityPool: LiquidityPoolContract;

  beforeAll(async () => {
    // Compile all contracts
    await Promise.all([
      DEXContract.compile(),
      TokenContract.compile(),
      LiquidityPoolContract.compile(),
    ]);
  });

  beforeEach(async () => {
    const Local = Mina.LocalBlockchain({ proofsEnabled: true });
    Mina.setActiveInstance(Local);

    // Deploy all contracts
    await deployContracts();
    await setupInitialLiquidity();
  });

  it('should execute complete swap flow', async () => {
    const trader = Local.testAccounts[2];
    const swapAmount = UInt64.from(100);

    // 1. Trader gets some tokens
    await mintTokensToTrader(trader, swapAmount);

    // 2. Approve DEX to spend tokens
    await approveTokenSpending(trader, swapAmount);

    // 3. Execute swap
    const expectedOutput = await calculateSwapOutput(swapAmount);
    await executeSwap(trader, swapAmount, expectedOutput);

    // 4. Verify final balances
    await verifySwapResults(trader, swapAmount, expectedOutput);
  });

  it('should handle liquidity provision correctly', async () => {
    const liquidityProvider = Local.testAccounts[3];
    const amountA = UInt64.from(1000);
    const amountB = UInt64.from(2000);

    await provideLiquidity(liquidityProvider, amountA, amountB);
    await verifyLiquidityPoolState(amountA, amountB);
  });

  // Helper methods for complex integration scenarios
  async function deployContracts(): Promise<void> {
    // Implementation for deploying multiple contracts
  }

  async function setupInitialLiquidity(): Promise<void> {
    // Setup initial liquidity for testing
  }

  async function mintTokensToTrader(trader: any, amount: UInt64): Promise<void> {
    // Mint tokens to trader account
  }

  // Additional helper methods...
});
```

### End-to-End Testing with Real Network

```typescript
// tests/e2e/deployment.e2e.test.ts
import { ConfigManager } from '../../src/config/ConfigManager';
import { DeploymentManager } from '../../src/scripts/DeploymentManager';

describe('E2E Deployment Tests', () => {
  let configManager: ConfigManager;
  let deploymentManager: DeploymentManager;

  beforeAll(async () => {
    configManager = new ConfigManager();
    deploymentManager = new DeploymentManager(configManager);

    // Connect to testnet
    await configManager.setupNetwork('staging');
  });

  it('should deploy contracts to testnet', async () => {
    const deploymentResult = await deploymentManager.deployAll();

    expect(deploymentResult.success).toBe(true);
    expect(deploymentResult.contractAddresses).toBeDefined();

    // Verify contracts are actually deployed
    for (const [name, address] of Object.entries(deploymentResult.contractAddresses)) {
      const accountInfo = await Mina.getAccount(PublicKey.fromBase58(address));
      expect(accountInfo).toBeDefined();
    }
  });

  it('should interact with deployed contracts', async () => {
    const addresses = await deploymentManager.getDeployedAddresses();
    const tokenContract = new TokenContract(addresses.tokenContract);

    // Test contract interaction
    const initialSupply = tokenContract.totalSupply.get();
    expect(initialSupply).toEqual(UInt64.zero);

    // Execute transaction
    const mintTx = await Mina.transaction(() => {
      tokenContract.mint(PublicKey.fromBase58(addresses.owner), UInt64.from(1000));
    });

    await mintTx.prove();
    await mintTx.send();

    // Wait for transaction confirmation
    await waitForTransaction(mintTx.hash());

    // Verify state change
    const newSupply = tokenContract.totalSupply.get();
    expect(newSupply).toEqual(UInt64.from(1000));
  });

  async function waitForTransaction(txHash: string): Promise<void> {
    // Implementation for waiting for transaction confirmation
    let confirmed = false;
    let attempts = 0;
    const maxAttempts = 30;

    while (!confirmed && attempts < maxAttempts) {
      try {
        // Query transaction status
        const status = await Mina.getTransactionStatus(txHash);
        confirmed = status === 'INCLUDED';
      } catch {
        // Transaction not yet included
      }

      if (!confirmed) {
        await new Promise(resolve => setTimeout(resolve, 10000)); // Wait 10 seconds
        attempts++;
      }
    }

    if (!confirmed) {
      throw new Error('Transaction confirmation timeout');
    }
  }
});
```

## Production Deployment Strategies

### Comprehensive Deployment Manager

```typescript
// scripts/DeploymentManager.ts
import { PrivateKey, PublicKey, Mina, AccountUpdate } from 'o1js';
import { ConfigManager } from '../config/ConfigManager';

interface DeploymentResult {
  success: boolean;
  contractAddresses: Record<string, string>;
  transactionHashes: Record<string, string>;
  error?: string;
}

interface ContractInfo {
  name: string;
  contractClass: any;
  constructorArgs: any[];
  initArgs?: any[];
}

class DeploymentManager {
  private configManager: ConfigManager;
  private deployerKey: PrivateKey;

  constructor(configManager: ConfigManager) {
    this.configManager = configManager;
    const config = configManager.getCurrentConfig();
    this.deployerKey = PrivateKey.fromBase58(config.deployerPrivateKey);
  }

  async deployAll(): Promise<DeploymentResult> {
    try {
      console.log('Starting deployment process...');

      // Define contracts to deploy in order
      const contracts: ContractInfo[] = [
        {
          name: 'TokenContract',
          contractClass: TokenContract,
          constructorArgs: [],
          initArgs: ['MyToken', 'MTK', 9], // name, symbol, decimals
        },
        {
          name: 'DEXContract',
          contractClass: DEXContract,
          constructorArgs: [],
        },
        // Add more contracts as needed
      ];

      const results = await this.deployContracts(contracts);
      await this.saveDeploymentResults(results);

      return results;

    } catch (error) {
      console.error('Deployment failed:', error);
      return {
        success: false,
        contractAddresses: {},
        transactionHashes: {},
        error: error.message,
      };
    }
  }

  private async deployContracts(contracts: ContractInfo[]): Promise<DeploymentResult> {
    const contractAddresses: Record<string, string> = {};
    const transactionHashes: Record<string, string> = {};

    for (const contractInfo of contracts) {
      console.log(`Deploying ${contractInfo.name}...`);

      // Compile contract
      console.log(`Compiling ${contractInfo.name}...`);
      await contractInfo.contractClass.compile();

      // Generate key pair for contract
      const contractKey = PrivateKey.random();
      const contractAddress = contractKey.toPublicKey();

      // Create contract instance
      const contract = new contractInfo.contractClass(contractAddress, ...contractInfo.constructorArgs);

      // Deploy transaction
      const tx = await Mina.transaction(this.deployerKey, () => {
        AccountUpdate.fundNewAccount(this.deployerKey);
        contract.deploy({ zkappKey: contractKey });

        // Call init if provided
        if (contractInfo.initArgs) {
          contract.init(...contractInfo.initArgs);
        }
      });

      console.log(`Proving ${contractInfo.name} deployment...`);
      await tx.prove();

      console.log(`Sending ${contractInfo.name} deployment transaction...`);
      const pendingTx = await tx.send();

      // Wait for confirmation
      await this.waitForTransaction(pendingTx.hash);

      contractAddresses[contractInfo.name] = contractAddress.toBase58();
      transactionHashes[contractInfo.name] = pendingTx.hash;

      console.log(`✅ ${contractInfo.name} deployed at: ${contractAddress.toBase58()}`);
    }

    return {
      success: true,
      contractAddresses,
      transactionHashes,
    };
  }

  private async waitForTransaction(txHash: string): Promise<void> {
    console.log(`Waiting for transaction confirmation: ${txHash}`);

    let confirmed = false;
    let attempts = 0;
    const maxAttempts = 50; // ~8 minutes with 10s intervals

    while (!confirmed && attempts < maxAttempts) {
      try {
        const status = await Mina.getTransactionStatus(txHash);
        if (status === 'INCLUDED') {
          confirmed = true;
          console.log(`✅ Transaction confirmed: ${txHash}`);
        }
      } catch (error) {
        console.log(`Transaction not yet confirmed, attempt ${attempts + 1}/${maxAttempts}`);
      }

      if (!confirmed) {
        await new Promise(resolve => setTimeout(resolve, 10000)); // Wait 10 seconds
        attempts++;
      }
    }

    if (!confirmed) {
      throw new Error(`Transaction confirmation timeout: ${txHash}`);
    }
  }

  private async saveDeploymentResults(results: DeploymentResult): Promise<void> {
    const config = this.configManager.getCurrentConfig();
    const timestamp = new Date().toISOString();

    const deploymentRecord = {
      timestamp,
      network: config.network.name,
      networkId: config.network.networkId,
      deployer: this.deployerKey.toPublicKey().toBase58(),
      results,
    };

    // Save to file
    const fs = await import('fs/promises');
    const filename = `deployments/${config.network.networkId}-${timestamp}.json`;
    await fs.writeFile(filename, JSON.stringify(deploymentRecord, null, 2));

    console.log(`Deployment results saved to: ${filename}`);
  }

  async getDeployedAddresses(): Promise<Record<string, string>> {
    // Load latest deployment results
    const config = this.configManager.getCurrentConfig();
    return config.contractAddresses;
  }
}
```

### CI/CD Pipeline Configuration

```yaml
# .github/workflows/zkapp-ci.yml
name: zkApp CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm run test

      - name: Run coverage
        run: npm run coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  build:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build contracts
        run: npm run build

      - name: Archive build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: build/

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
          path: build/

      - name: Deploy to staging
        env:
          NODE_ENV: staging
          DEPLOYER_PRIVATE_KEY: ${{ secrets.STAGING_DEPLOYER_PRIVATE_KEY }}
        run: npm run deploy:staging

  deploy-production:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
          path: build/

      - name: Deploy to production
        env:
          NODE_ENV: production
          DEPLOYER_PRIVATE_KEY: ${{ secrets.PRODUCTION_DEPLOYER_PRIVATE_KEY }}
        run: npm run deploy:production

      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.run_number }}
          release_name: Release v${{ github.run_number }}
          draft: false
          prerelease: false
```

## Performance Optimization and Monitoring

### Circuit Optimization Strategies

```typescript
// tools/CircuitAnalyzer.ts
import { Provable } from 'o1js';

interface CircuitMetrics {
  constraintCount: number;
  publicInputCount: number;
  witnessCount: number;
  estimatedProofTime: number;
  estimatedVerificationTime: number;
  memoryUsage: number;
}

class CircuitAnalyzer {
  static analyzeMethod(method: Function): CircuitMetrics {
    const analysis = Provable.constraintSystem(() => {
      // Execute method to build constraint system
      method();
    });

    return {
      constraintCount: analysis.numConstraints,
      publicInputCount: analysis.publicInputSize,
      witnessCount: analysis.witnessSize,
      estimatedProofTime: this.estimateProofTime(analysis.numConstraints),
      estimatedVerificationTime: this.estimateVerificationTime(analysis.numConstraints),
      memoryUsage: this.estimateMemoryUsage(analysis.numConstraints),
    };
  }

  static compareImplementations(implementations: Record<string, Function>): void {
    console.log('Circuit Implementation Comparison:');
    console.log('=====================================');

    const results = Object.entries(implementations).map(([name, impl]) => {
      const metrics = this.analyzeMethod(impl);
      return { name, ...metrics };
    });

    // Sort by constraint count
    results.sort((a, b) => a.constraintCount - b.constraintCount);

    results.forEach((result, index) => {
      console.log(`${index + 1}. ${result.name}`);
      console.log(`   Constraints: ${result.constraintCount.toLocaleString()}`);
      console.log(`   Est. Proof Time: ${result.estimatedProofTime.toFixed(2)}s`);
      console.log(`   Est. Memory: ${(result.memoryUsage / 1024 / 1024).toFixed(2)}MB`);
      console.log('');
    });
  }

  private static estimateProofTime(constraints: number): number {
    // Rough estimation based on constraint count
    // These are approximate values and may vary by hardware
    return (constraints / 1000) * 0.5; // ~0.5ms per 1000 constraints
  }

  private static estimateVerificationTime(constraints: number): number {
    // Verification is typically much faster than proving
    return (constraints / 10000) * 0.1; // ~0.1ms per 10000 constraints
  }

  private static estimateMemoryUsage(constraints: number): number {
    // Rough memory estimation in bytes
    return constraints * 256; // ~256 bytes per constraint
  }
}

// Usage example
const implementations = {
  'Naive Implementation': () => {
    // Inefficient approach
    let sum = Field(0);
    for (let i = 0; i < 100; i++) {
      sum = sum.add(Field(i).mul(Field(i)));
    }
  },
  'Optimized Implementation': () => {
    // More efficient approach
    const values = Array.from({ length: 100 }, (_, i) => Field(i));
    const sum = values.reduce((acc, val) => acc.add(val.square()), Field(0));
  },
  'Batched Implementation': () => {
    // Batch operations
    const batch = Provable.Array(Field, 100);
    const values = batch.from(Array.from({ length: 100 }, (_, i) => Field(i)));
    const sum = values.reduce((acc, val) => acc.add(val.square()), Field(0));
  },
};

CircuitAnalyzer.compareImplementations(implementations);
```

### Performance Monitoring and Metrics

```typescript
// monitoring/PerformanceMonitor.ts
interface PerformanceMetrics {
  timestamp: number;
  operation: string;
  duration: number;
  constraintCount?: number;
  memoryUsage?: number;
  success: boolean;
  error?: string;
}

class PerformanceMonitor {
  private metrics: PerformanceMetrics[] = [];
  private metricsEndpoint?: string;

  constructor(metricsEndpoint?: string) {
    this.metricsEndpoint = metricsEndpoint;
  }

  async measureAsync<T>(
    operation: string,
    fn: () => Promise<T>
  ): Promise<T> {
    const startTime = performance.now();
    const startMemory = this.getMemoryUsage();

    try {
      const result = await fn();
      const endTime = performance.now();
      const endMemory = this.getMemoryUsage();

      this.recordMetric({
        timestamp: Date.now(),
        operation,
        duration: endTime - startTime,
        memoryUsage: endMemory - startMemory,
        success: true,
      });

      return result;
    } catch (error) {
      const endTime = performance.now();

      this.recordMetric({
        timestamp: Date.now(),
        operation,
        duration: endTime - startTime,
        success: false,
        error: error.message,
      });

      throw error;
    }
  }

  measureSync<T>(operation: string, fn: () => T): T {
    const startTime = performance.now();
    const startMemory = this.getMemoryUsage();

    try {
      const result = fn();
      const endTime = performance.now();
      const endMemory = this.getMemoryUsage();

      this.recordMetric({
        timestamp: Date.now(),
        operation,
        duration: endTime - startTime,
        memoryUsage: endMemory - startMemory,
        success: true,
      });

      return result;
    } catch (error) {
      const endTime = performance.now();

      this.recordMetric({
        timestamp: Date.now(),
        operation,
        duration: endTime - startTime,
        success: false,
        error: error.message,
      });

      throw error;
    }
  }

  getMetrics(): PerformanceMetrics[] {
    return [...this.metrics];
  }

  getAverageTime(operation: string): number {
    const operationMetrics = this.metrics.filter(m => m.operation === operation && m.success);
    if (operationMetrics.length === 0) return 0;

    const total = operationMetrics.reduce((sum, metric) => sum + metric.duration, 0);
    return total / operationMetrics.length;
  }

  getSuccessRate(operation: string): number {
    const operationMetrics = this.metrics.filter(m => m.operation === operation);
    if (operationMetrics.length === 0) return 0;

    const successful = operationMetrics.filter(m => m.success).length;
    return successful / operationMetrics.length;
  }

  generateReport(): string {
    const operations = [...new Set(this.metrics.map(m => m.operation))];

    let report = 'Performance Report\n';
    report += '==================\n\n';

    operations.forEach(operation => {
      const avgTime = this.getAverageTime(operation);
      const successRate = this.getSuccessRate(operation);
      const count = this.metrics.filter(m => m.operation === operation).length;

      report += `Operation: ${operation}\n`;
      report += `  Executions: ${count}\n`;
      report += `  Average Time: ${avgTime.toFixed(2)}ms\n`;
      report += `  Success Rate: ${(successRate * 100).toFixed(1)}%\n\n`;
    });

    return report;
  }

  private recordMetric(metric: PerformanceMetrics): void {
    this.metrics.push(metric);

    // Send to monitoring endpoint if configured
    if (this.metricsEndpoint) {
      this.sendMetric(metric);
    }

    // Keep only last 1000 metrics in memory
    if (this.metrics.length > 1000) {
      this.metrics = this.metrics.slice(-1000);
    }
  }

  private async sendMetric(metric: PerformanceMetrics): Promise<void> {
    try {
      await fetch(this.metricsEndpoint!, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(metric),
      });
    } catch (error) {
      console.warn('Failed to send metric to monitoring endpoint:', error);
    }
  }

  private getMemoryUsage(): number {
    if (typeof process !== 'undefined' && process.memoryUsage) {
      return process.memoryUsage().heapUsed;
    }
    return 0;
  }
}

// Global performance monitor instance
export const performanceMonitor = new PerformanceMonitor(
  process.env.METRICS_ENDPOINT
);

// Usage in contracts
class OptimizedContract extends SmartContract {
  @method async performComplexOperation(input: Field): Promise<Field> {
    return performanceMonitor.measureAsync('complexOperation', async () => {
      // Complex operation implementation
      return this.complexCalculation(input);
    });
  }

  private complexCalculation(input: Field): Field {
    return performanceMonitor.measureSync('complexCalculation', () => {
      // Actual calculation
      return input.mul(input).add(Field(1));
    });
  }
}
```

## Security Best Practices and Audit Preparation

### Security Checklist and Patterns

```typescript
// security/SecurityChecklist.ts
interface SecurityIssue {
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  category: string;
  description: string;
  location: string;
  recommendation: string;
}

class SecurityAnalyzer {
  private issues: SecurityIssue[] = [];

  analyzeContract(contractCode: string, contractName: string): SecurityIssue[] {
    this.issues = [];

    // Check for common security issues
    this.checkStateValidation(contractCode, contractName);
    this.checkAccessControl(contractCode, contractName);
    this.checkIntegerOverflow(contractCode, contractName);
    this.checkSignatureValidation(contractCode, contractName);
    this.checkReentrancy(contractCode, contractName);

    return this.issues;
  }

  private checkStateValidation(code: string, contractName: string): void {
    // Check for missing state preconditions
    const stateGetPattern = /\.get\(\)/g;
    const requireEqualsPattern = /\.requireEquals\(/g;

    const getMatches = code.match(stateGetPattern)?.length || 0;
    const requireMatches = code.match(requireEqualsPattern)?.length || 0;

    if (getMatches > requireMatches * 2) {
      this.issues.push({
        severity: 'HIGH',
        category: 'State Validation',
        description: 'Potential missing state preconditions',
        location: contractName,
        recommendation: 'Use .getAndRequireEquals() or add explicit .requireEquals() calls',
      });
    }
  }

  private checkAccessControl(code: string, contractName: string): void {
    // Check for methods without access control
    const methodPattern = /@method\s+async\s+(\w+)/g;
    const methods = [];
    let match;

    while ((match = methodPattern.exec(code)) !== null) {
      methods.push(match[1]);
    }

    methods.forEach(method => {
      const methodStart = code.indexOf(`async ${method}`);
      const methodEnd = code.indexOf('}', methodStart);
      const methodCode = code.substring(methodStart, methodEnd);

      if (!methodCode.includes('this.sender') &&
          !methodCode.includes('signature.verify') &&
          !methodCode.includes('requireEquals')) {
        this.issues.push({
          severity: 'MEDIUM',
          category: 'Access Control',
          description: `Method ${method} may lack access control`,
          location: `${contractName}.${method}`,
          recommendation: 'Add appropriate access control checks',
        });
      }
    });
  }

  private checkIntegerOverflow(code: string, contractName: string): void {
    // Check for potential overflow in arithmetic operations
    const arithmeticPattern = /(\w+)\.add\(|(\w+)\.mul\(/g;
    const rangeCheckPattern = /assertLessThan|assertGreaterThan/g;

    const arithmeticOps = code.match(arithmeticPattern)?.length || 0;
    const rangeChecks = code.match(rangeCheckPattern)?.length || 0;

    if (arithmeticOps > rangeChecks) {
      this.issues.push({
        severity: 'MEDIUM',
        category: 'Integer Overflow',
        description: 'Arithmetic operations without range checks',
        location: contractName,
        recommendation: 'Add range checks for arithmetic operations',
      });
    }
  }

  private checkSignatureValidation(code: string, contractName: string): void {
    // Check for proper signature validation
    if (code.includes('Signature') && !code.includes('.verify(')) {
      this.issues.push({
        severity: 'HIGH',
        category: 'Signature Validation',
        description: 'Signature parameter without verification',
        location: contractName,
        recommendation: 'Always verify signatures before using them',
      });
    }
  }

  private checkReentrancy(code: string, contractName: string): void {
    // Check for potential reentrancy issues
    if (code.includes('this.send(') && code.includes('.set(')) {
      const sendIndex = code.indexOf('this.send(');
      const setState = code.indexOf('.set(', sendIndex);

      if (setState !== -1 && setState > sendIndex) {
        this.issues.push({
          severity: 'HIGH',
          category: 'Reentrancy',
          description: 'State modification after external call',
          location: contractName,
          recommendation: 'Update state before making external calls',
        });
      }
    }
  }
}

// Secure contract template
abstract class SecureContract extends SmartContract {
  // Owner/admin management
  @state(PublicKey) owner = State<PublicKey>();

  // Emergency pause mechanism
  @state(Bool) paused = State<Bool>();

  init() {
    super.init();
    this.owner.set(this.sender);
    this.paused.set(Bool(false));
  }

  // Access control modifier pattern
  protected requireOwner(): void {
    const owner = this.owner.getAndRequireEquals();
    this.sender.assertEquals(owner);
  }

  protected requireNotPaused(): void {
    const paused = this.paused.getAndRequireEquals();
    paused.assertFalse();
  }

  // Safe arithmetic operations
  protected safeAdd(a: UInt64, b: UInt64): UInt64 {
    const result = a.add(b);
    // Check for overflow (simplified)
    result.assertGreaterThanOrEqual(a);
    return result;
  }

  protected safeSub(a: UInt64, b: UInt64): UInt64 {
    a.assertGreaterThanOrEqual(b);
    return a.sub(b);
  }

  // Emergency functions
  @method async pause(signature: Signature) {
    this.requireOwner();

    const owner = this.owner.getAndRequireEquals();
    signature.verify(owner, [Bool(true).toField()]).assertTrue();

    this.paused.set(Bool(true));
  }

  @method async unpause(signature: Signature) {
    this.requireOwner();

    const owner = this.owner.getAndRequireEquals();
    signature.verify(owner, [Bool(false).toField()]).assertTrue();

    this.paused.set(Bool(false));
  }

  // Ownership transfer with two-step process
  @state(PublicKey) pendingOwner = State<PublicKey>();

  @method async transferOwnership(newOwner: PublicKey, signature: Signature) {
    this.requireOwner();

    const owner = this.owner.getAndRequireEquals();
    signature.verify(owner, newOwner.toFields()).assertTrue();

    this.pendingOwner.set(newOwner);
  }

  @method async acceptOwnership() {
    const pendingOwner = this.pendingOwner.getAndRequireEquals();
    this.sender.assertEquals(pendingOwner);

    this.owner.set(pendingOwner);
    this.pendingOwner.set(PublicKey.empty());
  }
}
```

This completes the comprehensive 6-part documentation covering everything from core architecture to production deployment and security best practices. The documentation provides complete coverage of Mina Protocol, o1js, and Zeko Labs with practical examples and production-ready patterns.

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "Create Part 1: Core Architecture and Philosophy", "status": "completed", "activeForm": "Creating Part 1 documentation"}, {"content": "Create Part 2: o1js Framework Deep Dive", "status": "completed", "activeForm": "Creating Part 2 documentation"}, {"content": "Create Part 3: Smart Contract Development Patterns", "status": "completed", "activeForm": "Creating Part 3 documentation"}, {"content": "Create Part 4: Advanced Features and Recursion", "status": "completed", "activeForm": "Creating Part 4 documentation"}, {"content": "Create Part 5: Zeko L2 Architecture and Integration", "status": "completed", "activeForm": "Creating Part 5 documentation"}, {"content": "Create Part 6: Development Workflows and Best Practices", "status": "completed", "activeForm": "Creating Part 6 documentation"}]