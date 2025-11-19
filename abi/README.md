# Contract ABIs

> **Application Binary Interfaces for Kirobo Prediction Market Contracts**

---

## 📥 Available ABIs

### Core Contracts

#### [PredictionMarket.json](PredictionMarket.json)
**Individual Market Contract**

Download the ABI for interacting with prediction market instances:
- **Direct JSON Link**: [PredictionMarket.json](https://kiroboio.github.io/prediction-ai-docs/abi/PredictionMarket.json)
- **GitHub Raw**: [View on GitHub](https://raw.githubusercontent.com/kiroboio/prediction-ai-docs/main/abi/PredictionMarket.json)

**Key Functions:**
- `stake(uint256 amount, bool outcome, uint256 minTokensOut, uint256 deadline)` - Place a bet
- `getCurrentPrices()` - Get current YES/NO prices
- `claimRewards()` - Claim winnings after settlement
- `getMarketInfo()` - Get market details
- `getUserPosition(address user)` - Get user's position

---

#### [PredictionMarketFactory.json](PredictionMarketFactory.json)
**Factory Contract for Market Creation**

Download the ABI for creating and managing markets:
- **Direct JSON Link**: [PredictionMarketFactory.json](https://kiroboio.github.io/prediction-ai-docs/abi/PredictionMarketFactory.json)
- **GitHub Raw**: [View on GitHub](https://raw.githubusercontent.com/kiroboio/prediction-ai-docs/main/abi/PredictionMarketFactory.json)

**Key Functions:**
- `createMarket(MarketParams memory params)` - Create new market
- `getAllMarkets()` - Get all markets
- `getMarketsByCreator(address creator)` - Get creator's markets
- `getMarketsByCategory(string memory category)` - Filter by category
- `getCreatorProfile(address creator)` - Get creator stats

---

## 🚀 Quick Start

### JavaScript/TypeScript (ethers.js v6)

```typescript
import { ethers } from 'ethers';

// Import ABIs
import PredictionMarketABI from './abi/PredictionMarket.json';
import PredictionMarketFactoryABI from './abi/PredictionMarketFactory.json';

// Connect to provider
const provider = new ethers.JsonRpcProvider('https://mainnet.base.org');
const signer = await provider.getSigner();

// Factory contract
const factoryAddress = '0x...'; // See network configs
const factory = new ethers.Contract(
  factoryAddress,
  PredictionMarketFactoryABI,
  signer
);

// Market contract
const marketAddress = '0x...';
const market = new ethers.Contract(
  marketAddress,
  PredictionMarketABI,
  signer
);

// Create a market
const tx = await factory.createMarket({
  statementTrue: "ETH > $4000 on Jan 1, 2026",
  statementFalse: "ETH <= $4000 on Jan 1, 2026",
  category: "crypto",
  tags: ["ethereum", "price"],
  stakeToken: "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", // USDC on Base
  stakingDeadline: Math.floor(Date.now() / 1000) + 86400 * 7, // 7 days
  settlementTime: Math.floor(Date.now() / 1000) + 86400 * 14, // 14 days
  ammModuleId: "0x...", // ConstantProductAMM
  stakingModuleId: "0x...", // StandardStaking
  oracleModuleId: "0x...", // UMAOracleModule
  // ... fee configuration
});

await tx.wait();
```

### JavaScript/TypeScript (ethers.js v5)

```typescript
import { ethers } from 'ethers';
import PredictionMarketABI from './abi/PredictionMarket.json';
import PredictionMarketFactoryABI from './abi/PredictionMarketFactory.json';

const provider = new ethers.providers.JsonRpcProvider('https://mainnet.base.org');
const signer = provider.getSigner();

const factory = new ethers.Contract(factoryAddress, PredictionMarketFactoryABI, signer);
const market = new ethers.Contract(marketAddress, PredictionMarketABI, signer);
```

### Python (web3.py)

```python
from web3 import Web3
import json

# Load ABIs
with open('abi/PredictionMarket.json') as f:
    market_abi = json.load(f)

with open('abi/PredictionMarketFactory.json') as f:
    factory_abi = json.load(f)

# Connect
w3 = Web3(Web3.HTTPProvider('https://mainnet.base.org'))
account = w3.eth.account.from_key('YOUR_PRIVATE_KEY')

# Factory contract
factory_address = '0x...'
factory = w3.eth.contract(address=factory_address, abi=factory_abi)

# Market contract
market_address = '0x...'
market = w3.eth.contract(address=market_address, abi=market_abi)

# Call functions
all_markets = factory.functions.getAllMarkets().call()
market_info = market.functions.getMarketInfo().call()
```

### Rust (ethers-rs)

```rust
use ethers::prelude::*;

// Load ABIs
abigen!(
    PredictionMarket,
    "./abi/PredictionMarket.json"
);

abigen!(
    PredictionMarketFactory,
    "./abi/PredictionMarketFactory.json"
);

// Connect
let provider = Provider::<Http>::try_from("https://mainnet.base.org")?;
let wallet = "YOUR_PRIVATE_KEY".parse::<LocalWallet>()?;
let client = SignerMiddleware::new(provider, wallet);

// Factory contract
let factory_address = "0x...".parse::<Address>()?;
let factory = PredictionMarketFactory::new(factory_address, client.clone());

// Market contract
let market_address = "0x...".parse::<Address>()?;
let market = PredictionMarket::new(market_address, client);
```

---

## 📖 Integration Guides

For detailed integration instructions, see:

- **[Factory Integration Guide](../guides/factory-integration-guide.md)** - Creating markets programmatically
- **[Market DApp Guide](../guides/market-dapp-guide.md)** - Building prediction market UIs
- **[AI-Optimized Guides](../guides/)** - Concise versions for AI agents

---

## 🌐 Network Configurations

### Mainnet Networks

**Ethereum Mainnet (Chain ID: 1)**
```json
{
  "factory": "TBD",
  "umaOracle": "0xfb55F43fB9F48F63f9269DB7Dde3BbBe1ebDC0dE",
  "usdc": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "rpcUrl": "https://eth.llamarpc.com"
}
```

**Base Mainnet (Chain ID: 8453)**
```json
{
  "factory": "TBD",
  "umaOracle": "0x2aBf1Bd76655de80eDb3086114315Eec75AF500c",
  "usdc": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "rpcUrl": "https://mainnet.base.org"
}
```

### Testnet Networks

**Ethereum Sepolia (Chain ID: 11155111)**
```json
{
  "factory": "TBD",
  "umaOracle": "0xFd9e2642a170aDD10F53Ee14a93FcF2F31924944",
  "usdc": "0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8",
  "rpcUrl": "https://ethereum-sepolia.publicnode.com"
}
```

**Base Sepolia (Chain ID: 84532)**
```json
{
  "factory": "TBD",
  "umaOracle": "0x0F7fC5E6482f096380db6158f978167b57388deE",
  "usdc": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
  "rpcUrl": "https://sepolia.base.org"
}
```

---

## 📦 NPM Package (Coming Soon)

We're planning to publish these ABIs as an NPM package:

```bash
npm install @kirobo/prediction-market-abis
```

```typescript
import { PredictionMarketABI, PredictionMarketFactoryABI } from '@kirobo/prediction-market-abis';
```

---

## 🔄 Version History

| Version | Date | Changes |
|---------|------|---------|
| **1.0.0** | Nov 2025 | Initial release with core contracts |

---

## 📝 ABI Checksums

**PredictionMarket.json**
- Size: 75 KB
- Functions: 50+
- Events: 20+

**PredictionMarketFactory.json**
- Size: 41 KB
- Functions: 30+
- Events: 15+

---

## 🔗 Additional Resources

- **[Documentation Home](https://kiroboio.github.io/prediction-ai-docs/)**
- **[Technical Whitepaper](../WHITEPAPER.md)**
- **[GitHub Repository](https://github.com/kiroboio/ki-prediction-ai)**
- **[Integration Guides](../guides/)**

---

## 📄 License

MIT License - see [LICENSE](../LICENSE) for details
