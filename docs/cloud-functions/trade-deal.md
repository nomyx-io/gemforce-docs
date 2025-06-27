# Trade Deal Functions

The Trade Deal Functions module provides comprehensive trade deal management capabilities for the Gemforce platform, enabling users to create, manage, and execute collateralized trade deals with invoice NFTs and automated settlement.

## Overview

The Trade Deal Functions provide:

- **Trade Deal Creation**: Create collateralized trade deals with customizable terms
- **Invoice Management**: Generate and manage invoice NFTs for trade deals
- **Collateral Management**: Handle collateral deposits and releases
- **Settlement Automation**: Automated trade deal settlement and dispute resolution
- **Payment Processing**: Integrate with various payment methods and currencies
- **Compliance**: KYC/AML compliance and regulatory reporting

## Key Features

### Trade Deal Lifecycle
- **Deal Creation**: Create trade deals with flexible terms and conditions
- **Collateral Handling**: Secure collateral deposit and management
- **Invoice Generation**: Automatic invoice NFT creation and management
- **Settlement Processing**: Automated settlement upon deal completion
- **Dispute Resolution**: Built-in dispute resolution mechanisms

### Financial Integration
- **Multi-Currency Support**: Support for various cryptocurrencies and stablecoins
- **Payment Rails**: Integration with traditional and crypto payment systems
- **Escrow Services**: Secure escrow for high-value transactions
- **Fee Management**: Transparent fee structure and collection

### Compliance Features
- **KYC Integration**: Know Your Customer verification
- **AML Monitoring**: Anti-Money Laundering compliance
- **Regulatory Reporting**: Automated compliance reporting
- **Audit Trail**: Complete transaction audit trail

## Core Functions

### createTradeDeal()
Creates a new trade deal with specified terms and collateral requirements.

**Parameters:**
```typescript
interface CreateTradeDealRequest {
  dealType: 'purchase_order' | 'service_agreement' | 'supply_contract';
  buyer: string;
  seller: string;
  dealTerms: {
    description: string;
    totalAmount: string;
    currency: string;
    deliveryDate: Date;
    paymentTerms: string;
    specifications?: any;
  };
  collateralRequirements: {
    buyerCollateral: string;
    sellerCollateral: string;
    collateralCurrency: string;
  };
  milestones?: TradeDealMilestone[];
  disputeResolution?: DisputeResolutionTerms;
}

interface TradeDealMilestone {
  id: string;
  description: string;
  amount: string;
  dueDate: Date;
  deliverables: string[];
}

interface DisputeResolutionTerms {
  arbitrator?: string;
  timeoutPeriod: number;
  penaltyPercentage: number;
}
```

**Returns:**
```typescript
interface CreateTradeDealResponse {
  success: boolean;
  dealId: string;
  contractAddress?: string;
  collateralAddresses?: {
    buyer: string;
    seller: string;
  };
  message: string;
}
```

**Usage:**
```typescript
const result = await Parse.Cloud.run('createTradeDeal', {
  dealType: 'purchase_order',
  buyer: 'buyer@company.com',
  seller: 'seller@supplier.com',
  dealTerms: {
    description: 'Supply of 1000 units of Product X',
    totalAmount: '50000',
    currency: 'USDC',
    deliveryDate: new Date('2024-12-31'),
    paymentTerms: 'Net 30',
    specifications: {
      quantity: 1000,
      quality: 'Grade A',
      packaging: 'Standard'
    }
  },
  collateralRequirements: {
    buyerCollateral: '5000',
    sellerCollateral: '2500',
    collateralCurrency: 'USDC'
  }
});
```

### depositCollateral()
Deposits collateral for a trade deal.

**Parameters:**
```typescript
interface DepositCollateralRequest {
  dealId: string;
  party: 'buyer' | 'seller';
  amount: string;
  currency: string;
  paymentMethod: 'crypto' | 'bank_transfer' | 'credit_card';
}
```

**Returns:**
```typescript
interface DepositCollateralResponse {
  success: boolean;
  transactionHash?: string;
  escrowAddress?: string;
  confirmationRequired?: boolean;
  message: string;
}
```

### generateInvoice()
Generates an invoice NFT for a trade deal milestone.

**Parameters:**
```typescript
interface GenerateInvoiceRequest {
  dealId: string;
  milestoneId?: string;
  invoiceDetails: {
    invoiceNumber: string;
    amount: string;
    currency: string;
    dueDate: Date;
    lineItems: InvoiceLineItem[];
    taxDetails?: TaxDetails;
  };
  recipient: string;
}

interface InvoiceLineItem {
  description: string;
  quantity: number;
  unitPrice: string;
  totalPrice: string;
}

interface TaxDetails {
  taxRate: number;
  taxAmount: string;
  taxJurisdiction: string;
}
```

**Returns:**
```typescript
interface GenerateInvoiceResponse {
  success: boolean;
  invoiceId: string;
  nftTokenId?: string;
  invoiceNFTAddress?: string;
  metadataURI?: string;
  message: string;
}
```

### settleTradeDeal()
Settles a completed trade deal and releases collateral.

**Parameters:**
```typescript
interface SettleTradeDealRequest {
  dealId: string;
  settlementType: 'successful' | 'disputed' | 'cancelled';
  settlementDetails?: {
    actualDeliveryDate?: Date;
    qualityRating?: number;
    performanceNotes?: string;
  };
  disputeResolution?: {
    ruling: string;
    collateralDistribution: {
      buyerReturn: string;
      sellerReturn: string;
      penaltyAmount: string;
    };
  };
}
```

**Returns:**
```typescript
interface SettleTradeDealResponse {
  success: boolean;
  settlementTransactionHash?: string;
  collateralReleaseHashes?: string[];
  finalStatus: string;
  message: string;
}
```

### getTradeDeal()
Retrieves trade deal information and current status.

**Parameters:**
```typescript
interface GetTradeDealRequest {
  dealId: string;
}
```

**Returns:**
```typescript
interface GetTradeDealResponse {
  success: boolean;
  tradeDeal?: {
    id: string;
    dealType: string;
    buyer: string;
    seller: string;
    status: 'created' | 'collateral_pending' | 'active' | 'completed' | 'disputed' | 'cancelled';
    dealTerms: any;
    collateralStatus: {
      buyerDeposited: boolean;
      sellerDeposited: boolean;
      amounts: any;
    };
    milestones: TradeDealMilestone[];
    invoices: InvoiceInfo[];
    timeline: TradeDealEvent[];
    createdAt: Date;
    updatedAt: Date;
  };
  message: string;
}

interface InvoiceInfo {
  id: string;
  invoiceNumber: string;
  amount: string;
  status: 'generated' | 'sent' | 'paid' | 'overdue';
  nftTokenId?: string;
  createdAt: Date;
}

interface TradeDealEvent {
  type: string;
  description: string;
  timestamp: Date;
  actor: string;
  transactionHash?: string;
}
```

## Implementation Example

### Trade Deal Creation with Smart Contract Deployment

```typescript
Parse.Cloud.define('createTradeDeal', async (request) => {
  const { dealType, buyer, seller, dealTerms, collateralRequirements, milestones, disputeResolution } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Validate participants
    const buyerUser = await validateTradeDealParticipant(buyer);
    const sellerUser = await validateTradeDealParticipant(seller);
    
    if (!buyerUser || !sellerUser) {
      throw new Error('Invalid trade deal participants');
    }
    
    // Check KYC compliance
    const buyerKYC = await checkKYCStatus(buyerUser.id);
    const sellerKYC = await checkKYCStatus(sellerUser.id);
    
    if (!buyerKYC.verified || !sellerKYC.verified) {
      throw new Error('KYC verification required for all participants');
    }
    
    // Validate deal terms
    await validateTradeDealTerms(dealTerms, collateralRequirements);
    
    // Create trade deal record
    const tradeDeal = new Parse.Object('TradeDeal');
    tradeDeal.set('dealType', dealType);
    tradeDeal.set('buyer', buyerUser);
    tradeDeal.set('seller', sellerUser);
    tradeDeal.set('creator', user);
    tradeDeal.set('dealTerms', dealTerms);
    tradeDeal.set('collateralRequirements', collateralRequirements);
    tradeDeal.set('milestones', milestones || []);
    tradeDeal.set('disputeResolution', disputeResolution);
    tradeDeal.set('status', 'created');
    tradeDeal.set('createdAt', new Date());
    tradeDeal.set('updatedAt', new Date());
    
    await tradeDeal.save();
    
    // Deploy trade deal smart contract
    const contractDeployment = await deployTradeDealContract({
      dealId: tradeDeal.id,
      buyer: buyerUser.get('walletAddress'),
      seller: sellerUser.get('walletAddress'),
      dealTerms,
      collateralRequirements
    });
    
    if (!contractDeployment.success) {
      tradeDeal.set('status', 'failed');
      tradeDeal.set('deploymentError', contractDeployment.message);
      await tradeDeal.save();
      throw new Error(`Contract deployment failed: ${contractDeployment.message}`);
    }
    
    // Update trade deal with contract information
    tradeDeal.set('contractAddress', contractDeployment.contractAddress);
    tradeDeal.set('deploymentTxHash', contractDeployment.txHash);
    tradeDeal.set('status', 'collateral_pending');
    await tradeDeal.save();
    
    // Create collateral escrow contracts
    const escrowDeployment = await deployCollateralEscrows(tradeDeal.id, collateralRequirements);
    
    if (escrowDeployment.success) {
      tradeDeal.set('collateralAddresses', escrowDeployment.addresses);
      await tradeDeal.save();
    }
    
    // Send notifications to participants
    await sendTradeDealNotifications(tradeDeal.id, 'created');
    
    // Log trade deal creation event
    await logTradeDealEvent(tradeDeal.id, 'deal_created', 'Trade deal created successfully', user.id);
    
    return {
      success: true,
      dealId: tradeDeal.id,
      contractAddress: contractDeployment.contractAddress,
      collateralAddresses: escrowDeployment.addresses,
      message: 'Trade deal created successfully'
    };
    
  } catch (error) {
    console.error('Trade deal creation error:', error);
    return {
      success: false,
      message: error.message || 'Failed to create trade deal'
    };
  }
});
```

### Collateral Management

```typescript
Parse.Cloud.define('depositCollateral', async (request) => {
  const { dealId, party, amount, currency, paymentMethod } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Get trade deal
    const tradeDeal = await new Parse.Query('TradeDeal')
      .equalTo('objectId', dealId)
      .first({ useMasterKey: true });
    
    if (!tradeDeal) {
      throw new Error('Trade deal not found');
    }
    
    // Verify user is authorized party
    const isAuthorized = (party === 'buyer' && tradeDeal.get('buyer').id === user.id) ||
                        (party === 'seller' && tradeDeal.get('seller').id === user.id);
    
    if (!isAuthorized) {
      throw new Error('Unauthorized to deposit collateral for this party');
    }
    
    // Check if collateral already deposited
    const collateralStatus = tradeDeal.get('collateralStatus') || {};
    const partyKey = `${party}Deposited`;
    
    if (collateralStatus[partyKey]) {
      throw new Error('Collateral already deposited');
    }
    
    // Validate collateral amount
    const requiredAmount = tradeDeal.get('collateralRequirements')[`${party}Collateral`];
    if (amount !== requiredAmount) {
      throw new Error(`Incorrect collateral amount. Required: ${requiredAmount}`);
    }
    
    // Process collateral deposit based on payment method
    let depositResult;
    
    switch (paymentMethod) {
      case 'crypto':
        depositResult = await processCryptoCollateralDeposit(dealId, party, amount, currency, user);
        break;
      case 'bank_transfer':
        depositResult = await processBankTransferCollateral(dealId, party, amount, currency, user);
        break;
      case 'credit_card':
        depositResult = await processCreditCardCollateral(dealId, party, amount, currency, user);
        break;
      default:
        throw new Error('Unsupported payment method');
    }
    
    if (!depositResult.success) {
      throw new Error(`Collateral deposit failed: ${depositResult.message}`);
    }
    
    // Update collateral status
    collateralStatus[partyKey] = true;
    collateralStatus[`${party}Amount`] = amount;
    collateralStatus[`${party}TxHash`] = depositResult.transactionHash;
    collateralStatus[`${party}DepositedAt`] = new Date();
    
    tradeDeal.set('collateralStatus', collateralStatus);
    
    // Check if both parties have deposited collateral
    if (collateralStatus.buyerDeposited && collateralStatus.sellerDeposited) {
      tradeDeal.set('status', 'active');
      tradeDeal.set('activatedAt', new Date());
      
      // Activate the trade deal contract
      await activateTradeDealContract(tradeDeal.get('contractAddress'));
      
      // Send activation notifications
      await sendTradeDealNotifications(dealId, 'activated');
    }
    
    await tradeDeal.save();
    
    // Log collateral deposit event
    await logTradeDealEvent(dealId, 'collateral_deposited', `${party} deposited collateral`, user.id, depositResult.transactionHash);
    
    return {
      success: true,
      transactionHash: depositResult.transactionHash,
      escrowAddress: depositResult.escrowAddress,
      confirmationRequired: depositResult.confirmationRequired,
      message: 'Collateral deposited successfully'
    };
    
  } catch (error) {
    console.error('Collateral deposit error:', error);
    return {
      success: false,
      message: error.message || 'Failed to deposit collateral'
    };
  }
});
```

### Invoice NFT Generation

```typescript
Parse.Cloud.define('generateInvoice', async (request) => {
  const { dealId, milestoneId, invoiceDetails, recipient } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Get trade deal
    const tradeDeal = await new Parse.Query('TradeDeal')
      .equalTo('objectId', dealId)
      .first({ useMasterKey: true });
    
    if (!tradeDeal) {
      throw new Error('Trade deal not found');
    }
    
    // Verify user is authorized to generate invoice
    const isAuthorized = tradeDeal.get('seller').id === user.id || tradeDeal.get('buyer').id === user.id;
    if (!isAuthorized) {
      throw new Error('Unauthorized to generate invoice for this trade deal');
    }
    
    // Validate invoice details
    await validateInvoiceDetails(invoiceDetails);
    
    // Create invoice record
    const invoice = new Parse.Object('Invoice');
    invoice.set('tradeDeal', tradeDeal);
    invoice.set('milestoneId', milestoneId);
    invoice.set('invoiceNumber', invoiceDetails.invoiceNumber);
    invoice.set('amount', invoiceDetails.amount);
    invoice.set('currency', invoiceDetails.currency);
    invoice.set('dueDate', invoiceDetails.dueDate);
    invoice.set('lineItems', invoiceDetails.lineItems);
    invoice.set('taxDetails', invoiceDetails.taxDetails);
    invoice.set('recipient', recipient);
    invoice.set('issuer', user);
    invoice.set('status', 'generated');
    invoice.set('createdAt', new Date());
    
    await invoice.save();
    
    // Generate invoice metadata
    const metadata = await generateInvoiceMetadata(invoice.id, invoiceDetails);
    
    // Mint invoice NFT
    const nftMinting = await mintInvoiceNFT({
      invoiceId: invoice.id,
      recipient,
      metadata,
      tradeDealContract: tradeDeal.get('contractAddress')
    });
    
    if (!nftMinting.success) {
      invoice.set('status', 'failed');
      invoice.set('mintingError', nftMinting.message);
      await invoice.save();
      throw new Error(`Invoice NFT minting failed: ${nftMinting.message}`);
    }
    
    // Update invoice with NFT information
    invoice.set('nftTokenId', nftMinting.tokenId);
    invoice.set('nftAddress', nftMinting.contractAddress);
    invoice.set('metadataURI', nftMinting.metadataURI);
    invoice.set('mintingTxHash', nftMinting.txHash);
    invoice.set('status', 'minted');
    await invoice.save();
    
    // Update trade deal with invoice reference
    const invoices = tradeDeal.get('invoices') || [];
    invoices.push({
      id: invoice.id,
      invoiceNumber: invoiceDetails.invoiceNumber,
      amount: invoiceDetails.amount,
      nftTokenId: nftMinting.tokenId,
      createdAt: new Date()
    });
    tradeDeal.set('invoices', invoices);
    await tradeDeal.save();
    
    // Send invoice notification
    await sendInvoiceNotification(invoice.id, recipient);
    
    // Log invoice generation event
    await logTradeDealEvent(dealId, 'invoice_generated', `Invoice ${invoiceDetails.invoiceNumber} generated`, user.id, nftMinting.txHash);
    
    return {
      success: true,
      invoiceId: invoice.id,
      nftTokenId: nftMinting.tokenId,
      invoiceNFTAddress: nftMinting.contractAddress,
      metadataURI: nftMinting.metadataURI,
      message: 'Invoice generated successfully'
    };
    
  } catch (error) {
    console.error('Invoice generation error:', error);
    return {
      success: false,
      message: error.message || 'Failed to generate invoice'
    };
  }
});
```

### Trade Deal Settlement

```typescript
Parse.Cloud.define('settleTradeDeal', async (request) => {
  const { dealId, settlementType, settlementDetails, disputeResolution } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Get trade deal
    const tradeDeal = await new Parse.Query('TradeDeal')
      .equalTo('objectId', dealId)
      .first({ useMasterKey: true });
    
    if (!tradeDeal) {
      throw new Error('Trade deal not found');
    }
    
    // Verify user is authorized to settle
    const isAuthorized = tradeDeal.get('buyer').id === user.id || 
                        tradeDeal.get('seller').id === user.id ||
                        isArbitrator(user.id, tradeDeal);
    
    if (!isAuthorized) {
      throw new Error('Unauthorized to settle this trade deal');
    }
    
    // Validate settlement type and current status
    if (tradeDeal.get('status') !== 'active' && tradeDeal.get('status') !== 'disputed') {
      throw new Error('Trade deal is not in a settleable state');
    }
    
    // Process settlement based on type
    let settlementResult;
    
    switch (settlementType) {
      case 'successful':
        settlementResult = await processSuccessfulSettlement(tradeDeal, settlementDetails);
        break;
      case 'disputed':
        settlementResult = await processDisputedSettlement(tradeDeal, disputeResolution);
        break;
      case 'cancelled':
        settlementResult = await processCancelledSettlement(tradeDeal);
        break;
      default:
        throw new Error('Invalid settlement type');
    }
    
    if (!settlementResult.success) {
      throw new Error(`Settlement failed: ${settlementResult.message}`);
    }
    
    // Update trade deal status
    tradeDeal.set('status', settlementType === 'successful' ? 'completed' : settlementType);
    tradeDeal.set('settlementType', settlementType);
    tradeDeal.set('settlementDetails', settlementDetails);
    tradeDeal.set('settlementTxHash', settlementResult.transactionHash);
    tradeDeal.set('settledAt', new Date());
    tradeDeal.set('settledBy', user);
    
    if (disputeResolution) {
      tradeDeal.set('disputeResolution', disputeResolution);
    }
    
    await tradeDeal.save();
    
    // Release collateral according to settlement terms
    const collateralRelease = await releaseCollateral(tradeDeal, settlementType, disputeResolution);
    
    // Send settlement notifications
    await sendSettlementNotifications(dealId, settlementType);
    
    // Update compliance records
    await updateComplianceRecords(tradeDeal, settlementType);
    
    // Log settlement event
    await logTradeDealEvent(dealId, 'deal_settled', `Trade deal settled: ${settlementType}`, user.id, settlementResult.transactionHash);
    
    return {
      success: true,
      settlementTransactionHash: settlementResult.transactionHash,
      collateralReleaseHashes: collateralRelease.transactionHashes,
      finalStatus: tradeDeal.get('status'),
      message: 'Trade deal settled successfully'
    };
    
  } catch (error) {
    console.error('Trade deal settlement error:', error);
    return {
      success: false,
      message: error.message || 'Failed to settle trade deal'
    };
  }
});
```

## Compliance and Reporting

### KYC Integration

```typescript
async function checkKYCStatus(userId: string): Promise<{ verified: boolean; level: string; expiresAt?: Date }> {
  try {
    const kycRecord = await new Parse.Query('KYCRecord')
      .equalTo('user', userId)
      .first({ useMasterKey: true });
    
    if (!kycRecord) {
      return { verified: false, level: 'none' };
    }
    
    const status = kycRecord.get('status');
    const level = kycRecord.get('level');
    const expiresAt = kycRecord.get('expiresAt');
    
    // Check if KYC is still valid
    if (expiresAt && new Date() > expiresAt) {
      return { verified: false, level, expiresAt };
    }
    
    return {
      verified: status === 'verified',
      level,
      expiresAt
    };
    
  } catch (error) {
    console.error('KYC check error:', error);
    return { verified: false, level: 'error' };
  }
}
```

### AML Monitoring

```typescript
async function performAMLCheck(tradeDeal: any): Promise<{ passed: boolean; riskScore: number; flags: string[] }> {
  try {
    const buyer = tradeDeal.get('buyer');
    const seller = tradeDeal.get('seller');
    const dealAmount = parseFloat(tradeDeal.get('dealTerms').totalAmount);
    
    let riskScore = 0;
    const flags: string[] = [];
    
    // Check transaction amount
    if (dealAmount > 10000) {
      riskScore += 10;
      flags.push('high_value_transaction');
    }
    
    // Check user risk profiles
    const buyerRisk = await getUserRiskProfile(buyer.id);
    const sellerRisk = await getUserRiskProfile(seller.id);
    
    riskScore += buyerRisk.score + sellerRisk.score;
    flags.push(...buyerRisk.flags, ...sellerRisk.flags);
    
    // Check for suspicious patterns
    const suspiciousActivity = await detectSuspiciousActivity(buyer.id, seller.id, dealAmount);
    if (suspiciousActivity.detected) {
      riskScore += 20;
      flags.push(...suspiciousActivity.flags);
    }
    
    // Determine if AML check passed
    const passed = riskScore < 50; // Threshold for automatic approval
    
    return { passed, riskScore, flags };
    
  } catch (error) {
    console.error('AML check error:', error);
    return { passed: false, riskScore: 100, flags: ['aml_check_error'] };
  }
}
```

## Integration Examples

### Frontend Integration

```typescript
class TradeDealService {
  async createTradeDeal(dealData: CreateTradeDealRequest): Promise<CreateTradeDealResponse> {
    return await Parse.Cloud.run('createTradeDeal', dealData);
  }
  
  async depositCollateral(
    dealId: string,
    party: 'buyer' | 'seller',
    amount: string,
    currency: string,
    paymentMethod: string
  ): Promise<DepositCollateralResponse> {
    return await Parse.Cloud.run('depositCollateral', {
      dealId,
      party,
      amount,
      currency,
      paymentMethod
    });
  }
  
  async generateInvoice(invoiceData: GenerateInvoiceRequest): Promise<GenerateInvoiceResponse> {
    return await Parse.Cloud.run('generateInvoice', invoiceData);
  }
  
  async settleTradeDeal(settlementData: SettleTradeDealRequest): Promise<SettleTradeDealResponse> {
    return await Parse.Cloud.run('settleTradeDeal', settlementData);
  }
  
  async getTradeDeal(dealId: string): Promise<GetTradeDealResponse> {
    return await Parse.Cloud.run('getTradeDeal', { dealId });
  }
}
```

### React Component Example

```typescript
export function TradeDealDashboard() {
  const [tradeDeals, setTradeDeals] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    loadTradeDeals();
  }, []);
  
  const loadTradeDeals = async () => {
    try {
      const result = await Parse.Cloud.run('listTradeDeals', { status: 'active' });
      if (result.success) {
        setTradeDeals(result.tradeDeals);
      }
    } catch (error) {
      console.error('Failed to load trade deals:', error);
    } finally {
      setLoading(false);
    }
  };
  
  const handleSettlement = async (dealId: string, settlementType: string) => {
    try {
      const result = await new TradeDealService().settleTradeDeal({
        dealId,
        settlementType,
        settlementDetails: {
          actualDeliveryDate: new Date(),
          qualityRating: 5,
          performanceNotes: 'Delivered as expected'
        }
      });
      
      if (result.success) {
        await loadTradeDeals(); // Refresh list
      }
    } catch (error) {
      console.error('Settlement failed:', error);
    }
  };
  
  return (
    <div>
      <h1>Trade Deals</h1>
      {loading ? (
        <div>Loading...</div>
      ) : (
        <div>
          {tradeDeals.map(deal => (
            <TradeDealCard 
              key={deal.id} 
              deal={deal} 
              onSettle={handleSettlement}
            />
          ))}
        </div>
      )}
    </div>
  );
}
```

## Error Handling

### Common Errors
- `"Authentication required"`: User not logged in
- `"KYC verification required"`: KYC not completed
- `"Invalid trade deal participants"`: Participant validation failed
- `"Collateral already deposited"`: Attempting to deposit collateral twice
- `"Unauthorized to settle"`: User lacks settlement permissions
- `"Trade deal not in settleable state"`: Invalid status for settlement

### Error Recovery
```typescript
async function handleTradeDealError(dealId: string, error: any, operation: string): Promise<void> {
  try {
    const tradeDeal = await new Parse.Query('TradeDeal')
      .equalTo('objectId', dealId)
      .first({ useMasterKey: true });
    
    if (tradeDeal) {
      // Log error event
      await logTradeDealEvent(dealId, 'error', `${operation} failed: ${error.message}`, 'system');
      
      // Send error notification to participants
      await sendErrorNotification(tradeDeal, error, operation);
      
      // Attempt automatic recovery for certain errors
      if (isRecoverableError(error)) {
        await attemptErrorRecovery(dealId, operation, error);
      }
    }
  } catch (recoveryError) {
    console.error('Error recovery failed:', recoveryError);
  }
}
```

## Best Practices

### Trade Deal Management
1. **KYC Compliance**: Ensure all participants are KYC verified
2. **Collateral Security**: Use secure escrow for collateral management
3. **Clear Terms**: Define clear and unambiguous deal terms
4. **Dispute Resolution**: Establish clear dispute resolution mechanisms

### Development Guidelines
1. **Error Handling**: Comprehensive error handling and recovery
2. **Event Logging**: Log all trade deal operations for auditing
3. **Testing**: Thorough testing of trade deal workflows
4. **Security**: Implement proper access controls and validation

## Related Documentation

- [Trade Deal Management Facet](../smart-contracts/facets/trade-deal-management-facet.md)
- [Trade Deal Operations Facet](../smart-contracts/facets/trade-deal-operations-facet.md)
- [Trade Deal Admin Facet](../smart-contracts/facets/trade-deal-admin-facet.md)
- [ITradeDeal Interface](../smart-contracts/interfaces/itradedeal.md)
- [Trade Deal Library](../smart-contracts/libraries/trade-deal-lib.md)
-