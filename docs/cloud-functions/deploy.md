# Deploy Functions

The Deploy Functions module provides comprehensive deployment capabilities for smart contracts, diamond proxies, and complete project infrastructures on multiple blockchain networks. It handles automated deployment pipelines, configuration management, and post-deployment verification.

## Overview

The Deploy Functions provide:

- **Smart Contract Deployment**: Deploy individual contracts and contract suites
- **Diamond Deployment**: Deploy and configure diamond proxy contracts
- **Multi-Network Support**: Deploy across multiple blockchain networks
- **Configuration Management**: Manage deployment configurations and parameters
- **Verification**: Automated contract verification and validation
- **Rollback Support**: Rollback capabilities for failed deployments

## Key Features

### Deployment Types
- **Single Contract**: Deploy individual smart contracts
- **Diamond Proxy**: Deploy upgradeable diamond contracts
- **Project Suite**: Deploy complete project infrastructures
- **Template Deployment**: Deploy from predefined templates

### Network Management
- **Multi-Network**: Support for multiple blockchain networks
- **Gas Optimization**: Automatic gas price optimization
- **Network Validation**: Pre-deployment network validation
- **Cross-Chain Coordination**: Coordinate deployments across networks

### Automation Features
- **Pipeline Automation**: Automated deployment pipelines
- **Configuration Validation**: Validate deployment configurations
- **Post-Deployment Testing**: Automated testing after deployment
- **Monitoring Integration**: Integration with monitoring systems

## Core Functions

### deployContract()
Deploys a single smart contract to a specified network.

**Parameters:**
```typescript
interface DeployContractRequest {
  contractName: string;
  network: string;
  constructorArgs?: any[];
  deploymentConfig?: {
    gasLimit?: string;
    gasPrice?: string;
    confirmations?: number;
    timeout?: number;
  };
  verificationConfig?: {
    verify: boolean;
    apiKey?: string;
    constructorArgsEncoded?: string;
  };
  metadata?: {
    projectId?: string;
    version?: string;
    description?: string;
  };
}
```

**Returns:**
```typescript
interface DeployContractResponse {
  success: boolean;
  contractAddress?: string;
  transactionHash?: string;
  gasUsed?: string;
  deploymentCost?: string;
  verificationStatus?: 'pending' | 'verified' | 'failed';
  message: string;
}
```

**Usage:**
```typescript
const result = await Parse.Cloud.run('deployContract', {
  contractName: 'MyToken',
  network: 'polygon',
  constructorArgs: ['My Token', 'MTK', 18, '1000000000000000000000000'],
  deploymentConfig: {
    gasLimit: '2000000',
    confirmations: 3
  },
  verificationConfig: {
    verify: true
  },
  metadata: {
    projectId: 'proj_123',
    version: '1.0.0',
    description: 'ERC20 token for my project'
  }
});
```

### deployDiamond()
Deploys a diamond proxy contract with specified facets.

**Parameters:**
```typescript
interface DeployDiamondRequest {
  network: string;
  diamondName: string;
  facets: FacetDeployment[];
  initContract?: string;
  initData?: string;
  owner?: string;
  deploymentConfig?: DeploymentConfig;
  verificationConfig?: VerificationConfig;
}

interface FacetDeployment {
  name: string;
  address?: string; // If already deployed
  constructorArgs?: any[];
  functionSelectors?: string[];
}
```

**Returns:**
```typescript
interface DeployDiamondResponse {
  success: boolean;
  diamondAddress?: string;
  facetAddresses?: { [facetName: string]: string };
  transactionHashes?: string[];
  totalGasUsed?: string;
  deploymentCost?: string;
  message: string;
}
```

### deployProject()
Deploys a complete project infrastructure including all contracts and configurations.

**Parameters:**
```typescript
interface DeployProjectRequest {
  projectId: string;
  network: string;
  deploymentTemplate: string;
  configuration: ProjectConfiguration;
  deploymentOptions?: {
    skipVerification?: boolean;
    dryRun?: boolean;
    parallelDeployment?: boolean;
    rollbackOnFailure?: boolean;
  };
}

interface ProjectConfiguration {
  contracts: ContractConfig[];
  diamonds: DiamondConfig[];
  initializations: InitializationStep[];
  permissions: PermissionConfig[];
}
```

**Returns:**
```typescript
interface DeployProjectResponse {
  success: boolean;
  deploymentId: string;
  deployedContracts?: { [name: string]: string };
  deployedDiamonds?: { [name: string]: string };
  totalGasUsed?: string;
  deploymentCost?: string;
  deploymentTime?: number;
  message: string;
}
```

### getDeploymentStatus()
Retrieves the status of an ongoing or completed deployment.

**Parameters:**
```typescript
interface DeploymentStatusRequest {
  deploymentId: string;
}
```

**Returns:**
```typescript
interface DeploymentStatusResponse {
  success: boolean;
  deployment?: {
    id: string;
    status: 'pending' | 'deploying' | 'completed' | 'failed' | 'rolled_back';
    network: string;
    progress: number;
    currentStep?: string;
    deployedContracts: { [name: string]: string };
    errors?: string[];
    startedAt: Date;
    completedAt?: Date;
    gasUsed: string;
    cost: string;
  };
  message: string;
}
```

### rollbackDeployment()
Rolls back a failed or problematic deployment.

**Parameters:**
```typescript
interface RollbackDeploymentRequest {
  deploymentId: string;
  reason: string;
  preserveData?: boolean;
}
```

**Returns:**
```typescript
interface RollbackDeploymentResponse {
  success: boolean;
  rollbackTransactionHashes?: string[];
  message: string;
}
```

## Implementation Example

### Smart Contract Deployment

```typescript
Parse.Cloud.define('deployContract', async (request) => {
  const { contractName, network, constructorArgs, deploymentConfig, verificationConfig, metadata } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Validate deployment permissions
    const hasPermission = await checkDeploymentPermissions(user.id, network);
    if (!hasPermission) {
      throw new Error('Insufficient permissions for deployment');
    }
    
    // Validate network
    if (!isSupportedNetwork(network)) {
      throw new Error('Unsupported network');
    }
    
    // Get contract artifacts
    const contractArtifacts = await getContractArtifacts(contractName);
    if (!contractArtifacts) {
      throw new Error('Contract artifacts not found');
    }
    
    // Validate constructor arguments
    if (constructorArgs) {
      await validateConstructorArgs(contractArtifacts.abi, constructorArgs);
    }
    
    // Create deployment record
    const deployment = new Parse.Object('Deployment');
    deployment.set('type', 'contract');
    deployment.set('contractName', contractName);
    deployment.set('network', network);
    deployment.set('deployer', user);
    deployment.set('status', 'pending');
    deployment.set('constructorArgs', constructorArgs);
    deployment.set('deploymentConfig', deploymentConfig);
    deployment.set('metadata', metadata);
    deployment.set('createdAt', new Date());
    
    await deployment.save();
    
    // Get network configuration
    const networkConfig = await getNetworkConfig(network);
    const provider = getProvider(networkConfig);
    const wallet = getDeploymentWallet(networkConfig);
    
    // Estimate gas
    const gasEstimate = await estimateDeploymentGas(
      contractArtifacts,
      constructorArgs,
      provider
    );
    
    // Configure deployment transaction
    const deploymentTx = {
      gasLimit: deploymentConfig?.gasLimit || gasEstimate.mul(120).div(100), // 20% buffer
      gasPrice: deploymentConfig?.gasPrice || await provider.getGasPrice(),
      ...networkConfig.deploymentDefaults
    };
    
    deployment.set('estimatedGas', gasEstimate.toString());
    deployment.set('gasPrice', deploymentTx.gasPrice.toString());
    deployment.set('status', 'deploying');
    await deployment.save();
    
    // Deploy contract
    const contractFactory = new ethers.ContractFactory(
      contractArtifacts.abi,
      contractArtifacts.bytecode,
      wallet
    );
    
    const contract = await contractFactory.deploy(...(constructorArgs || []), deploymentTx);
    
    deployment.set('transactionHash', contract.deployTransaction.hash);
    await deployment.save();
    
    // Wait for deployment confirmation
    const confirmations = deploymentConfig?.confirmations || 3;
    const receipt = await contract.deployTransaction.wait(confirmations);
    
    deployment.set('contractAddress', contract.address);
    deployment.set('blockNumber', receipt.blockNumber);
    deployment.set('gasUsed', receipt.gasUsed.toString());
    deployment.set('deploymentCost', receipt.gasUsed.mul(deploymentTx.gasPrice).toString());
    deployment.set('status', 'deployed');
    deployment.set('deployedAt', new Date());
    await deployment.save();
    
    // Verify contract if requested
    let verificationStatus = 'not_requested';
    if (verificationConfig?.verify) {
      try {
        const verificationResult = await verifyContract({
          contractAddress: contract.address,
          contractName,
          network,
          constructorArgs,
          apiKey: verificationConfig.apiKey
        });
        
        verificationStatus = verificationResult.success ? 'verified' : 'failed';
        deployment.set('verificationStatus', verificationStatus);
        deployment.set('verificationResult', verificationResult);
        await deployment.save();
      } catch (verificationError) {
        console.error('Contract verification failed:', verificationError);
        verificationStatus = 'failed';
      }
    }
    
    // Update project if specified
    if (metadata?.projectId) {
      await updateProjectDeployment(metadata.projectId, {
        contractName,
        contractAddress: contract.address,
        network,
        deploymentId: deployment.id
      });
    }
    
    // Send deployment notification
    await sendDeploymentNotification(deployment.id, 'completed');
    
    return {
      success: true,
      contractAddress: contract.address,
      transactionHash: contract.deployTransaction.hash,
      gasUsed: receipt.gasUsed.toString(),
      deploymentCost: receipt.gasUsed.mul(deploymentTx.gasPrice).toString(),
      verificationStatus,
      message: 'Contract deployed successfully'
    };
    
  } catch (error) {
    console.error('Contract deployment error:', error);
    
    // Update deployment record with error
    const deployment = await new Parse.Query('Deployment')
      .equalTo('contractName', contractName)
      .equalTo('deployer', user)
      .equalTo('status', 'deploying')
      .first({ useMasterKey: true });
    
    if (deployment) {
      deployment.set('status', 'failed');
      deployment.set('errorMessage', error.message);
      deployment.set('failedAt', new Date());
      await deployment.save();
    }
    
    return {
      success: false,
      message: error.message || 'Contract deployment failed'
    };
  }
});
```

### Diamond Deployment

```typescript
Parse.Cloud.define('deployDiamond', async (request) => {
  const { network, diamondName, facets, initContract, initData, owner, deploymentConfig, verificationConfig } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Validate deployment permissions
    const hasPermission = await checkDeploymentPermissions(user.id, network);
    if (!hasPermission) {
      throw new Error('Insufficient permissions for deployment');
    }
    
    // Create deployment record
    const deployment = new Parse.Object('Deployment');
    deployment.set('type', 'diamond');
    deployment.set('diamondName', diamondName);
    deployment.set('network', network);
    deployment.set('deployer', user);
    deployment.set('status', 'pending');
    deployment.set('facets', facets);
    deployment.set('createdAt', new Date());
    
    await deployment.save();
    
    const deployedContracts: { [name: string]: string } = {};
    const transactionHashes: string[] = [];
    let totalGasUsed = ethers.BigNumber.from(0);
    
    // Get network configuration
    const networkConfig = await getNetworkConfig(network);
    const provider = getProvider(networkConfig);
    const wallet = getDeploymentWallet(networkConfig);
    
    deployment.set('status', 'deploying');
    await deployment.save();
    
    // Deploy core diamond contracts first
    const diamondCutFacet = await deployCoreFacet('DiamondCutFacet', wallet, deploymentConfig);
    deployedContracts['DiamondCutFacet'] = diamondCutFacet.address;
    transactionHashes.push(diamondCutFacet.deployTransaction.hash);
    totalGasUsed = totalGasUsed.add(await diamondCutFacet.deployTransaction.wait().then(r => r.gasUsed));
    
    const diamondLoupeFacet = await deployCoreFacet('DiamondLoupeFacet', wallet, deploymentConfig);
    deployedContracts['DiamondLoupeFacet'] = diamondLoupeFacet.address;
    transactionHashes.push(diamondLoupeFacet.deployTransaction.hash);
    totalGasUsed = totalGasUsed.add(await diamondLoupeFacet.deployTransaction.wait().then(r => r.gasUsed));
    
    const ownershipFacet = await deployCoreFacet('OwnershipFacet', wallet, deploymentConfig);
    deployedContracts['OwnershipFacet'] = ownershipFacet.address;
    transactionHashes.push(ownershipFacet.deployTransaction.hash);
    totalGasUsed = totalGasUsed.add(await ownershipFacet.deployTransaction.wait().then(r => r.gasUsed));
    
    // Deploy custom facets
    for (const facet of facets) {
      if (!facet.address) {
        const facetContract = await deployFacet(facet, wallet, deploymentConfig);
        deployedContracts[facet.name] = facetContract.address;
        transactionHashes.push(facetContract.deployTransaction.hash);
        totalGasUsed = totalGasUsed.add(await facetContract.deployTransaction.wait().then(r => r.gasUsed));
      } else {
        deployedContracts[facet.name] = facet.address;
      }
    }
    
    // Prepare diamond cut for initialization
    const diamondCut = prepareDiamondCut([
      {
        facetAddress: diamondCutFacet.address,
        action: 0, // Add
        functionSelectors: getDiamondCutSelectors()
      },
      {
        facetAddress: diamondLoupeFacet.address,
        action: 0, // Add
        functionSelectors: getDiamondLoupeSelectors()
      },
      {
        facetAddress: ownershipFacet.address,
        action: 0, // Add
        functionSelectors: getOwnershipSelectors()
      },
      ...facets.map(facet => ({
        facetAddress: deployedContracts[facet.name],
        action: 0, // Add
        functionSelectors: facet.functionSelectors || getFacetSelectors(facet.name)
      }))
    ]);
    
    // Deploy diamond
    const diamondFactory = await getDiamondFactory(wallet);
    const diamondTx = await diamondFactory.deployDiamond(
      owner || wallet.address,
      diamondCut,
      initContract || ethers.constants.AddressZero,
      initData || '0x',
      deploymentConfig || {}
    );
    
    const diamondReceipt = await diamondTx.wait();
    const diamondAddress = await getDiamondAddressFromReceipt(diamondReceipt);
    
    deployedContracts['Diamond'] = diamondAddress;
    transactionHashes.push(diamondTx.hash);
    totalGasUsed = totalGasUsed.add(diamondReceipt.gasUsed);
    
    // Update deployment record
    deployment.set('diamondAddress', diamondAddress);
    deployment.set('deployedContracts', deployedContracts);
    deployment.set('transactionHashes', transactionHashes);
    deployment.set('totalGasUsed', totalGasUsed.toString());
    deployment.set('status', 'deployed');
    deployment.set('deployedAt', new Date());
    await deployment.save();
    
    // Verify contracts if requested
    if (verificationConfig?.verify) {
      await verifyDiamondContracts(deployedContracts, network, verificationConfig);
    }
    
    // Send deployment notification
    await sendDeploymentNotification(deployment.id, 'completed');
    
    return {
      success: true,
      diamondAddress,
      facetAddresses: deployedContracts,
      transactionHashes,
      totalGasUsed: totalGasUsed.toString(),
      deploymentCost: totalGasUsed.mul(await provider.getGasPrice()).toString(),
      message: 'Diamond deployed successfully'
    };
    
  } catch (error) {
    console.error('Diamond deployment error:', error);
    
    // Update deployment record with error
    const deployment = await new Parse.Query('Deployment')
      .equalTo('diamondName', diamondName)
      .equalTo('deployer', user)
      .equalTo('status', 'deploying')
      .first({ useMasterKey: true });
    
    if (deployment) {
      deployment.set('status', 'failed');
      deployment.set('errorMessage', error.message);
      deployment.set('failedAt', new Date());
      await deployment.save();
    }
    
    return {
      success: false,
      message: error.message || 'Diamond deployment failed'
    };
  }
});
```

### Project Deployment

```typescript
Parse.Cloud.define('deployProject', async (request) => {
  const { projectId, network, deploymentTemplate, configuration, deploymentOptions } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Get project
    const project = await new Parse.Query('Project')
      .equalTo('objectId', projectId)
      .first({ useMasterKey: true });
    
    if (!project) {
      throw new Error('Project not found');
    }
    
    // Check project access
    const hasAccess = await checkProjectAccess(projectId, user.id);
    if (!hasAccess) {
      throw new Error('Access denied');
    }
    
    // Create deployment record
    const deploymentId = generateDeploymentId();
    const deployment = new Parse.Object('ProjectDeployment');
    deployment.set('deploymentId', deploymentId);
    deployment.set('project', project);
    deployment.set('network', network);
    deployment.set('template', deploymentTemplate);
    deployment.set('configuration', configuration);
    deployment.set('deployer', user);
    deployment.set('status', 'pending');
    deployment.set('createdAt', new Date());
    
    await deployment.save();
    
    // Validate deployment configuration
    await validateProjectConfiguration(configuration, deploymentTemplate);
    
    // Perform dry run if requested
    if (deploymentOptions?.dryRun) {
      const dryRunResult = await performDryRun(configuration, network);
      return {
        success: true,
        deploymentId,
        dryRunResult,
        message: 'Dry run completed successfully'
      };
    }
    
    deployment.set('status', 'deploying');
    await deployment.save();
    
    const deployedContracts: { [name: string]: string } = {};
    const deployedDiamonds: { [name: string]: string } = {};
    let totalGasUsed = ethers.BigNumber.from(0);
    const startTime = Date.now();
    
    try {
      // Deploy contracts in dependency order
      for (const contractConfig of configuration.contracts) {
        const contractResult = await Parse.Cloud.run('deployContract', {
          contractName: contractConfig.name,
          network,
          constructorArgs: contractConfig.constructorArgs,
          deploymentConfig: contractConfig.deploymentConfig,
          verificationConfig: { verify: !deploymentOptions?.skipVerification }
        });
        
        if (!contractResult.success) {
          throw new Error(`Failed to deploy ${contractConfig.name}: ${contractResult.message}`);
        }
        
        deployedContracts[contractConfig.name] = contractResult.contractAddress;
        totalGasUsed = totalGasUsed.add(contractResult.gasUsed || '0');
        
        // Update deployment progress
        deployment.set('currentStep', `Deployed ${contractConfig.name}`);
        deployment.set('deployedContracts', deployedContracts);
        await deployment.save();
      }
      
      // Deploy diamonds
      for (const diamondConfig of configuration.diamonds) {
        const diamondResult = await Parse.Cloud.run('deployDiamond', {
          network,
          diamondName: diamondConfig.name,
          facets: diamondConfig.facets,
          initContract: diamondConfig.initContract,
          initData: diamondConfig.initData,
          owner: diamondConfig.owner,
          verificationConfig: { verify: !deploymentOptions?.skipVerification }
        });
        
        if (!diamondResult.success) {
          throw new Error(`Failed to deploy ${diamondConfig.name}: ${diamondResult.message}`);
        }
        
        deployedDiamonds[diamondConfig.name] = diamondResult.diamondAddress;
        totalGasUsed = totalGasUsed.add(diamondResult.totalGasUsed || '0');
        
        // Update deployment progress
        deployment.set('currentStep', `Deployed ${diamondConfig.name}`);
        deployment.set('deployedDiamonds', deployedDiamonds);
        await deployment.save();
      }
      
      // Execute initialization steps
      for (const initStep of configuration.initializations) {
        await executeInitializationStep(initStep, deployedContracts, deployedDiamonds, network);
        
        deployment.set('currentStep', `Executed ${initStep.name}`);
        await deployment.save();
      }
      
      // Configure permissions
      for (const permissionConfig of configuration.permissions) {
        await configurePermissions(permissionConfig, deployedContracts, deployedDiamonds, network);
        
        deployment.set('currentStep', `Configured ${permissionConfig.name}`);
        await deployment.save();
      }
      
      const deploymentTime = Date.now() - startTime;
      
      // Update deployment record
      deployment.set('status', 'completed');
      deployment.set('deployedContracts', deployedContracts);
      deployment.set('deployedDiamonds', deployedDiamonds);
      deployment.set('totalGasUsed', totalGasUsed.toString());
      deployment.set('deploymentTime', deploymentTime);
      deployment.set('completedAt', new Date());
      await deployment.save();
      
      // Update project with deployment information
      project.set('deployments', [...(project.get('deployments') || []), {
        deploymentId,
        network,
        status: 'completed',
        deployedAt: new Date(),
        contracts: deployedContracts,
        diamonds: deployedDiamonds
      }]);
      await project.save();
      
      // Send deployment notification
      await sendDeploymentNotification(deploymentId, 'completed');
      
      return {
        success: true,
        deploymentId,
        deployedContracts,
        deployedDiamonds,
        totalGasUsed: totalGasUsed.toString(),
        deploymentCost: totalGasUsed.mul(await getNetworkGasPrice(network)).toString(),
        deploymentTime,
        message: 'Project deployed successfully'
      };
      
    } catch (deploymentError) {
      // Handle deployment failure
      if (deploymentOptions?.rollbackOnFailure) {
        await rollbackProjectDeployment(deploymentId, deployedContracts, deployedDiamonds);
      }
      
      deployment.set('status', 'failed');
      deployment.set('errorMessage', deploymentError.message);
      deployment.set('failedAt', new Date());
      await deployment.save();
      
      throw deploymentError;
    }
    
  } catch (error) {
    console.error('Project deployment error:', error);
    return {
      success: false,
      message: error.message || 'Project deployment failed'
    };
  }
});
```

## Deployment Templates

### NFT Collection Template

```typescript
const nftCollectionTemplate = {
  name: 'NFT Collection',
  description: 'Complete NFT collection with marketplace',
  contracts: [
    {
      name: 'CollectionToken',
      type: 'ERC721',
      constructorArgs: ['{{tokenName}}', '{{tokenSymbol}}', '{{baseURI}}'],
      deploymentConfig: {
        gasLimit: '3000000'
      }
    }
  ],
  diamonds: [
    {
      name: 'CollectionDiamond',
      facets: [
        { name: 'ERC721Facet' },
        { name: 'MarketplaceFacet' },
        { name: 'MetadataFacet' },
        { name: 'OwnershipFacet' }
      ],
      initContract: 'CollectionInit',
      initData: '{{initData}}'
    }
  ],
  initializations: [
    {
      name: 'SetupMarketplace',
      contract: 'CollectionDiamond',
      function: 'setupMarketplace',
      args: ['{{marketplaceFee}}', '{{feeRecipient}}']
    }
  ],
  permissions: [
    {
      name: 'MinterRole',
      contract: 'CollectionDiamond',
      role: 'MINTER_ROLE',
      accounts: ['{{owner}}']
    }
  ]
};
```

### DeFi Protocol Template

```typescript
const defiProtocolTemplate = {
  name: 'DeFi Protocol',
  description: 'DeFi protocol with staking and governance',
  contracts: [
    {
      name: 'GovernanceToken',
      type: 'ERC20',
      constructorArgs: ['{{tokenName}}', '{{tokenSymbol}}', '{{totalSupply}}']
    },
    {
      name: 'Timelock',
      type: 'TimelockController',
      constructorArgs: ['{{timelockDelay}}', ['{{admin}}'], ['{{admin}}'], '{{admin}}']
    }
  ],
  diamonds: [
    {
      name: 'ProtocolDiamond',
      facets: [
        { name: 'StakingFacet' },
        { name: 'GovernanceFacet' },
        { name: 'FeeDistributorFacet' },
        { name: 'OwnershipFacet' }
      ]
    }
  ],
  initializations: [
    {
      name: 'SetupStaking',
      contract: 'ProtocolDiamond',
      function: 'setupStaking',
      args: ['{{stakingToken}}', '{{rewardRate}}']
    }
  ]
};
```

## Verification and Monitoring

### Contract Verification

```typescript
async function verifyContract(params: {
  contractAddress: string;
  contractName: string;
  network: string;
  constructorArgs?: any[];
  apiKey?: string;
}): Promise<{ success: boolean; message: string }> {
  try {
    const { contractAddress, contractName, network, constructorArgs, apiKey } = params;
    
    // Get network verification configuration
    const verificationConfig = getVerificationConfig(network);
    const explorerApiKey = apiKey || verificationConfig.defaultApiKey;
    
    if (!explorerApiKey) {
      throw new Error('API key required for verification');
    }
    
    // Get contract source code and metadata
    const contractArtifacts = await getContractArtifacts(contractName);
    const sourceCode = await getContractSourceCode(contractName);
    
    // Prepare verification request
    const verificationRequest = {
      apikey: explorerApiKey,
      module: 'contract',
      action: 'verifysourcecode',
      contractaddress: contractAddress,
      sourceCode: sourceCode,
      codeformat: 'solidity-single-file',
      contractname: contractName,
      compilerversion: contractArtifacts.compiler.version,
      optimizationUsed: contractArtifacts.compiler.optimizer.enabled ? 1 : 0,
      runs: contractArtifacts.compiler.optimizer.runs,
      constructorArguements: constructorArgs ? encodeConstructorArgs(constructorArgs) : ''
    };
    
    // Submit verification request
    const response = await submitVerificationRequest(verificationConfig.apiUrl, verificationRequest);
    
    if (response.status !== '1') {
      throw new Error(`Verification submission failed: ${response.result}`);
    }
    
    // Poll for verification result
    const verificationResult = await pollVerificationResult(
      verificationConfig.apiUrl,
      explorerApiKey,
      response.result
    );
    
    return {
      success: verificationResult.status === '1',
      message: verificationResult.result
    };
    
  } catch (error) {
    console.error('Contract verification error:', error);
    return {
      success: false,
      message: error.message || 'Verification failed'
    };
  }
}
```

### Deployment Monitoring

```typescript
Parse.Cloud.define('getDeploymentStatus', async (request) => {
  const { deploymentId } = request.params;
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Get deployment record
    const deployment = await new Parse.Query('Deployment')
      .equalTo('deploymentId', deploymentId)
      .first({ useMasterKey: true });
    
    if (!deployment) {
      throw new Error('Deployment not found');
    }
    
    // Check access permissions
    if (deployment.get('deployer').id !== user.id) {
      const hasAccess = await checkDeploymentAccess(deploymentId, user.id);
      if (!hasAccess) {
        throw new Error('Access denied');
      }
    }
    
    // Calculate progress
    const progress = calculateDeploymentProgress(deployment);
    
    return {
      success: true,
      deployment: {
        id: deployment.get('deploymentId'),
        status: deployment.get('status'),
        network: deployment.get('network'),
        progress,
        currentStep: deployment.get('currentStep'),
        deployedContracts: deployment.get('deployedContracts') || {},
        errors: deployment.get('errors') || [],
        startedAt: deployment.get('createdAt'),
        completedAt: deployment.get('completedAt