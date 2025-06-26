# Smart Contract Reference Documentation

## Overview

The Gemforce platform is built on a sophisticated smart contract architecture using the Diamond Standard (EIP-2535) for upgradeable contracts. This documentation provides comprehensive reference material for all smart contracts in the system.

## Architecture Overview

The Gemforce smart contract system consists of:

- **Diamond Contracts**: Core proxy contracts implementing EIP-2535
- **Facets**: Modular contract implementations providing specific functionality
- **Libraries**: Shared utility code for common operations
- **Interfaces**: Standard interfaces for contract interaction
- **Tokens**: ERC20, ERC721, and ERC1155 token implementations
- **Utilities**: Helper contracts for access control and security

## Contract Categories

### Core Diamond System
- [Diamond](diamond.md) - Main diamond proxy contract
- [DiamondFactory](diamond-factory.md) - Factory for diamond deployment
- [IdentityFactory](identity-factory.md) - Identity contract factory

### Business Logic Facets
- [CarbonCreditFacet](facets/carbon-credit-facet.md) - Carbon credit management
- [MarketplaceFacet](facets/marketplace-facet.md) - NFT marketplace operations
- [MultiSaleFacet](facets/multi-sale-facet.md) - Multi-token sales
- [TradeDealManagementFacet](facets/trade-deal-management-facet.md) - Trade deal lifecycle
- [TradeDealOperationsFacet](facets/trade-deal-operations-facet.md) - Trade deal operations
- [IdentityRegistryFacet](facets/identity-registry-facet.md) - Identity management
- [TrustedIssuersRegistryFacet](facets/trusted-issuers-registry-facet.md) - Trusted issuer management

### Token Management Facets
- [GemforceMinterFacet](facets/gemforce-minter-facet.md) - Token minting functionality
- [CollateralTokenFactoryFacet](facets/collateral-token-factory-facet.md) - Collateral token creation
- [FeeDistributorFacet](facets/fee-distributor-facet.md) - Fee distribution logic

### Utility Facets
- [SVGTemplatesFacet](facets/svg-templates-facet.md) - Dynamic SVG generation
- [DiamondCutFacet](facets/diamond-cut-facet.md) - Diamond upgrade functionality
- [DiamondLoupeFacet](facets/diamond-loupe-facet.md) - Diamond introspection
- [OwnershipFacet](facets/ownership-facet.md) - Contract ownership management

### Core Interfaces
- [IDiamond](interfaces/idiamond.md) - Diamond standard interface
- [IMarketplace](interfaces/imarketplace.md) - Marketplace interface
- [ITradeDeal](interfaces/itrade-deal.md) - Trade deal interface
- [ICarbonCredit](interfaces/icarbon-credit.md) - Carbon credit interface
- [IIdentity](interfaces/iidentity.md) - Identity interface

### Libraries
- [DiamondLib](libraries/diamond-lib.md) - Diamond pattern utilities
- [CarbonCreditLib](libraries/carbon-credit-lib.md) - Carbon credit utilities
- [TradeDealLib](libraries/trade-deal-lib.md) - Trade deal utilities
- [MultiSaleLib](libraries/multi-sale-lib.md) - Multi-sale utilities

## Getting Started

### For Developers
1. Start with the [Diamond](diamond.md) documentation to understand the core architecture
2. Review the [Interfaces](interfaces/) to understand contract APIs
3. Examine specific [Facets](facets/) for functionality you need to integrate
4. Check [Libraries](libraries/) for utility functions

### For Integrators
1. Review the [Integration Guide](../integration-guide.md)
2. Examine relevant interface documentation
3. Check deployment procedures in [Deployment Guide](../deployment-guide.md)
4. Review security considerations in each contract's documentation

## Security Considerations

All contracts implement:
- Reentrancy protection using OpenZeppelin's ReentrancyGuard
- Access control through role-based permissions
- Input validation and sanitization
- Event logging for transparency and monitoring

## Gas Optimization

The contracts are optimized for gas efficiency through:
- Diamond pattern for reduced deployment costs
- Packed structs for storage optimization
- Batch operations where applicable
- Efficient algorithms in libraries

## Upgrade Patterns

The Diamond Standard enables:
- Modular upgrades through facet replacement
- Backward compatibility maintenance
- Gradual feature rollouts
- Emergency upgrade procedures

---

*For detailed information about each contract, click on the links above or navigate through the documentation sections.*