# Integration Guides

> **Developer Documentation for Building on Kirobo Prediction Markets**

---

## 🔗 Contract ABIs

Before diving into the guides, download the contract ABIs:

- **[PredictionMarket.json](../abi/PredictionMarket.json)** - Market contract ABI ([Raw](https://raw.githubusercontent.com/kiroboio/prediction-ai-docs/main/abi/PredictionMarket.json))
- **[PredictionMarketFactory.json](../abi/PredictionMarketFactory.json)** - Factory contract ABI ([Raw](https://raw.githubusercontent.com/kiroboio/prediction-ai-docs/main/abi/PredictionMarketFactory.json))

See the **[ABI Documentation](../abi/)** for code examples in multiple languages.

---

## 📚 Available Guides

### For DApp Developers

#### [Market DApp Guide](market-dapp-guide.md)
**Full Developer Guide**
- Complete API reference with all functions
- Market lifecycle management
- Staking, settlement, and rewards
- UI/UX best practices
- Error handling and security

**Target Audience**: Frontend developers, full-stack developers building prediction market interfaces

#### [Market DApp Guide (AI-Optimized)](market-dapp-guide-ai.md)
**Concise Guide (70% shorter)**
- Business logic and non-obvious concepts only
- Assumes access to `IPredictionMarket` ABI
- Focuses on edge cases and gotchas
- Quick reference for experienced developers

**Target Audience**: AI agents, developers with ABI access, experienced DeFi developers

---

### For Protocol Integrators

#### [Factory Integration Guide](factory-integration-guide.md)
**Full Integration Guide**
- Complete factory API reference
- Market creation and deployment
- Creator management and tracking
- Fee configuration
- Module system integration
- Network configurations

**Target Audience**: Protocol developers, AI agents creating markets, platform builders

#### [Factory Integration Guide (AI-Optimized)](factory-integration-guide-ai.md)
**Concise Guide (70% shorter)**
- Business logic and non-obvious concepts only
- Assumes access to `IPredictionMarketFactory` ABI
- Focuses on rate limits, fees, and validation
- Quick reference for market creation

**Target Audience**: AI agents with ABI access, experienced protocol developers

---

## Quick Links

| What You Want to Build | Recommended Guide |
|-------------------------|-------------------|
| **Prediction Market UI** | [Market DApp Guide](market-dapp-guide.md) |
| **AI Market Creation Bot** | [Factory Guide (AI)](factory-integration-guide-ai.md) |
| **Protocol Integration** | [Factory Integration Guide](factory-integration-guide.md) |
| **Trading Bot** | [Market DApp Guide (AI)](market-dapp-guide-ai.md) |
| **White-Label Platform** | Both Factory + Market DApp Guides |

---

## Getting Started

### Prerequisites

- **Solidity Version**: 0.8.26
- **Required Tools**: ethers.js or web3.js, TypeScript (recommended)
- **Networks**: Ethereum Mainnet, Base, Sepolia, Base Sepolia
- **Tokens**: USDC (stake token)

### Basic Setup

```typescript
// 1. Import ABIs
import PredictionMarketFactoryABI from './abis/PredictionMarketFactory.json';
import PredictionMarketABI from './abis/PredictionMarket.json';

// 2. Connect to factory
const factoryAddress = "0x..."; // See network configs
const factory = new ethers.Contract(factoryAddress, PredictionMarketFactoryABI, signer);

// 3. Create or connect to market
const marketAddress = await factory.allMarkets(0);
const market = new ethers.Contract(marketAddress, PredictionMarketABI, signer);
```

---

## Key Features

### Factory Features
- ⚡ **90% Gas Savings**: EIP-1167 clone pattern
- 📊 **Creator Tracking**: On-chain statistics and reputation
- 🔒 **Rate Limiting**: 1-minute cooldown, 100 markets/day
- 💰 **Flexible Fees**: Custom fee structures per creator
- 🔍 **Market Discovery**: Filter by category, tags, time
- 🔧 **Module System**: Swappable AMM/Oracle/Staking

### Market Features
- 💱 **Constant Product AMM**: x*y=k pricing with 0.3% fee
- 🔮 **UMA Oracle**: Optimistic Oracle V3 settlement
- 💸 **Multi-Tier Fees**: Platform 1.2%, Creator 1.8%, Activator 2.0%
- 🎯 **Slippage Protection**: User-defined price limits
- ⏰ **Time Safety**: 2-hour buffer on timing constraints
- 🏆 **Fair Rewards**: Proportional distribution to winners

---

## Network Configurations

### Mainnet

**Ethereum (Chain ID: 1)**
```
Factory: TBD
UMA OOv3: 0xfb55F43fB9F48F63f9269DB7Dde3BbBe1ebDC0dE
USDC: 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
```

**Base (Chain ID: 8453)**
```
Factory: TBD
UMA OOv3: 0x2aBf1Bd76655de80eDb3086114315Eec75AF500c
USDC: 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
```

### Testnet

**Sepolia (Chain ID: 11155111)**
```
Factory: TBD
UMA OOv3: 0xFd9e2642a170aDD10F53Ee14a93FcF2F31924944
USDC: 0x94a9D9AC8a22534E3FaCa9F4e7F2E2cf85d5E4C8
```

**Base Sepolia (Chain ID: 84532)**
```
Factory: TBD
UMA OOv3: 0x0F7fC5E6482f096380db6158f978167b57388deE
USDC: 0x036CbD53842c5426634e7929541eC2318f3dCF7e
```

---

## Support

- **GitHub**: [kiroboio/ki-prediction-ai](https://github.com/kiroboio/ki-prediction-ai)
- **Documentation**: [kiroboio.github.io/prediction-ai-docs](https://kiroboio.github.io/prediction-ai-docs)
- **Issues**: [Report a bug](https://github.com/kiroboio/ki-prediction-ai/issues)

---

## License

MIT License - see [LICENSE](../LICENSE) for details
