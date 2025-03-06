# Gemforce Deployer Guide

## Table of Contents

- [Deployment Prerequisites](#deployment-prerequisites)
- [Smart Contract Deployment](#smart-contract-deployment)
- [Cloud Functions Deployment](#cloud-functions-deployment)
- [Environment Configuration](#environment-configuration)
- [Deployment Automation](#deployment-automation)
- [Upgrade Procedures](#upgrade-procedures)
- [Rollback Procedures](#rollback-procedures)
- [Testing and Verification](#testing-and-verification)
- [Network Management](#network-management)
- [Version Control](#version-control)

## Deployment Prerequisites

### Development Environment Setup

To deploy the Gemforce platform, you'll need the following tools and software:

1. **Node.js and npm**:
   - Node.js v16 or later
   - npm v7 or later

   ```bash
   # Install using nvm (recommended)
   nvm install 16
   nvm use 16
   
   # Verify installation
   node -v
   npm -v
   ```

2. **Hardhat**:
   - Used for smart contract development and deployment

   ```bash
   # Install hardhat
   npm install --save-dev hardhat
   
   # Verify installation
   npx hardhat --version
   ```

3. **MongoDB**:
   - Version 4.4 or later
   - Required for Parse Server

   ```bash
   # Installation varies by OS
   # macOS (using Homebrew)
   brew install mongodb-community
   
   # Verify installation
   mongod --version
   ```

4. **Git**:
   - For version control

   ```bash
   # Verify installation
   git --version
   ```

5. **TypeScript**:
   - Latest version

   ```bash
   # Install TypeScript
   npm install -g typescript
   
   # Verify installation
   tsc --version
   ```

### Required Tools and Software

1. **Development IDE**:
   - Visual Studio Code (recommended)
   - Plugins:
     - Solidity
     - TypeScript
     - ESLint
     - Prettier

2. **Blockchain Tools**:
   - MetaMask or similar wallet
   - Etherscan account (for contract verification)
   - Infura or Alchemy account (for RPC endpoints)

3. **Deployment Tools**:
   - PM2 (for process management)
   - Docker (optional, for containerized deployment)

   ```bash
   # Install PM2
   npm install -g pm2
   
   # Verify installation
   pm2 --version
   ```

4. **Database Tools**:
   - MongoDB Compass (GUI for MongoDB)
   - MongoDB Database Tools (mongodump, mongorestore)

### Network Access Requirements

Ensure your deployment environment has access to:

1. **Blockchain Networks**:
   - Ethereum Mainnet (if deploying to production)
   - BaseSepolia (for testing)
   - Other EVM-compatible networks as needed

2. **External APIs**:
   - DFNS API
   - Bridge API
   - SendGrid API

3. **Database Access**:
   - MongoDB server (local or hosted)
   - Redis (if using for caching)

4. **Github/Version Control**:
   - Access to Gemforce repositories

### Key Management Setup

1. **Create a secure key management strategy**:

   ```bash
   # Create a directory for keys (outside of repository)
   mkdir -p ~/.gemforce/keys
   chmod 700 ~/.gemforce/keys
   ```

2. **Generate deployment keys**:

   ```bash
   # Generate a private key for deployment
   openssl genpkey -algorithm RSA -out ~/.gemforce/keys/deployment_key.pem -pkeyopt rsa_keygen_bits:2048
   chmod 600 ~/.gemforce/keys/deployment_key.pem
   
   # Generate a DFNS private key
   openssl genpkey -algorithm RSA -out ~/.gemforce/keys/dfns_private.key -pkeyopt rsa_keygen_bits:2048
   chmod 600 ~/.gemforce/keys/dfns_private.key
   ```

3. **Use environment files for sensitive data**:

   ```bash
   # Create a .env file for local development
   cp .env.example .env
   
   # Edit to add your keys and endpoints
   nano .env
   ```

### Environment Preparation

1. **Clone the repository**:

   ```bash
   git clone https://github.com/your-org/gemforce.git
   cd gemforce
   ```

2. **Install dependencies**:

   ```bash
   npm install
   ```

3. **Set up environment files**:

   ```bash
   # Copy sample environment files
   cp .env.example .env
   cp gemforce.config.example.ts gemforce.config.ts
   
   # Edit the files with your configuration
   nano .env
   nano gemforce.config.ts
   ```

4. **Create deployment directories**:

   ```bash
   mkdir -p deployments
   ```

## Smart Contract Deployment

### Contract Compilation

1. **Compile contracts using Hardhat**:

   ```bash
   npx hardhat compile
   ```

2. **Verify compilation output**:
   - Check for successful compilation in `artifacts/` directory
   - Resolve any compilation errors

3. **Configuration for different networks**:

   ```javascript
   // hardhat.config.ts
   module.exports = {
     solidity: {
       version: "0.8.17",
       settings: {
         optimizer: {
           enabled: true,
           runs: 200
         }
       }
     },
     networks: {
       hardhat: {
         chainId: 31337
       },
       baseSepolia: {
         url: `https://sepolia.base.org`,
         accounts: [process.env.PRIVATE_KEY],
         chainId: 84532
       },
       mainnet: {
         url: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
         accounts: [process.env.PRIVATE_KEY],
         chainId: 1
       }
     }
   };
   ```

### Diamond Pattern Deployment Workflow

The Gemforce platform uses the Diamond pattern (EIP-2535) for smart contract deployment:

1. **Deploy libraries first**:

   ```bash
   npx hardhat deploy --tags Libraries --network baseSepolia
   ```

2. **Deploy facets**:

   ```bash
   npx hardhat deploy --tags Facets --network baseSepolia
   ```

3. **Deploy Diamond contract**:

   ```bash
   npx hardhat deploy --tags Diamond --network baseSepolia
   ```

4. **Verify the deployment**:

   ```bash
   npx hardhat verify --network baseSepolia <DIAMOND_ADDRESS>
   ```

### Facet Deployment Process

1. **Develop new facets** in the `/contracts/facets` directory
2. **Add to deployment scripts** in `/deploy` directory
3. **Set up facet cut data** for the Diamond contract:

   ```javascript
   // Example facet cut data
   const facetCuts = [
     {
       facetAddress: newFacetAddress,
       action: FacetCutAction.Add,
       functionSelectors: selectors
     }
   ];
   ```

4. **Update the Diamond** with new facets:

   ```javascript
   // Using the DiamondFactory
   await diamondFactory.setFacets(setName, facetCuts);
   ```

### Contract Initialization

1. **Prepare initialization data**:

   ```javascript
   // Example initialization data
   const initData = diamondInit.interface.encodeFunctionData("init", [
     [
       // Initial parameters
       tokenName,
       tokenSymbol,
       baseURI,
       // ...other params
     ]
   ]);
   ```

2. **Initialize the Diamond**:

   ```javascript
   const settings = {
     name: "Gemforce Token",
     symbol: "GEM",
     // other settings
   };
   
   await diamondFactory.createFromSet(
     settings,
     diamondInit.address,
     initData,
     "defaultFacetSet"
   );
   ```

### Contract Verification

1. **Verify Diamond contract on Etherscan/Basescan**:

   ```bash
   npx hardhat verify --network baseSepolia <DIAMOND_ADDRESS>
   ```

2. **Verify individual facets**:

   ```bash
   npx hardhat verify --network baseSepolia <FACET_ADDRESS>
   ```

3. **Manual verification** (if automatic verification fails):
   - Flatten the contract using Hardhat
   - Upload the flattened contract to the block explorer
   - Verify with the correct compiler settings

### Gas Optimization Strategies

1. **Use optimized Solidity patterns**:
   - Minimize storage operations
   - Batch operations where possible
   - Use efficient data structures

2. **Configure gas prices for deployment**:

   ```javascript
   // Example configuration for gas
   const tx = await contract.functionName(params, {
     gasPrice: (await ethers.provider.getGasPrice()).mul(2), // 2x current gas price
     gasLimit: 5000000
   });
   ```

3. **Monitor gas usage during testing**:

   ```javascript
   // Add gas reporter to Hardhat config
   gasReporter: {
     enabled: true,
     currency: 'USD',
     gasPrice: 100,
     coinmarketcap: process.env.COINMARKETCAP_API_KEY
   }
   ```

## Cloud Functions Deployment

### Parse Server Deployment

1. **Prepare Parse Server configuration**:

   ```javascript
   // Example Parse Server configuration
   const parseServerConfig = {
     appId: process.env.APP_ID,
     masterKey: process.env.MASTER_KEY,
     databaseURI: process.env.DATABASE_URI,
     serverURL: process.env.SERVER_URL,
     cloud: "./dist/src/cloud-functions.js",
     allowClientClassCreation: false,
     enableAnonymousUsers: false,
     maxUploadSize: "20mb",
     // ... other configuration
   };
   ```

2. **Deploy Parse Server**:
   
   Using PM2:

   ```bash
   # Start Parse Server with PM2
   pm2 start app.js --name gemforce-server
   
   # Save PM2 configuration
   pm2 save
   
   # Set up PM2 to start on system boot
   pm2 startup
   ```
   
   Using Docker:

   ```bash
   # Build Docker image
   docker build -t gemforce-server .
   
   # Run container
   docker run -d -p 1337:1337 \
     --env-file .env \
     --name gemforce-server \
     gemforce-server
   ```

3. **Verify Parse Server deployment**:

   ```bash
   # Check server status
   curl https://your-server-url.com/parse/health
   
   # Expected response: {"status":"ok"}
   ```

### Cloud Function Deployment

1. **Compile TypeScript files**:

   ```bash
   # Build the project
   npm run build
   ```

2. **Deploy cloud functions**:

   If using PM2:

   ```bash
   # Restart the server to apply changes
   pm2 restart gemforce-server
   ```

   If using Docker:

   ```bash
   # Rebuild and redeploy
   docker build -t gemforce-server .
   docker stop gemforce-server
   docker rm gemforce-server
   docker run -d -p 1337:1337 \
     --env-file .env \
     --name gemforce-server \
     gemforce-server
   ```

3. **Verify cloud function deployment**:

   ```bash
   # Test a simple cloud function
   curl -X POST \
     -H "X-Parse-Application-Id: ${APP_ID}" \
     -H "Content-Type: application/json" \
     -d '{}' \
     https://your-server-url.com/parse/functions/loadAllBlockchains
   ```

### Environment Configuration

1. **Set environment variables**:

   ```bash
   # Set environment variables in .env file
   cat > .env << EOL
   # Parse Server
   APP_ID=your_app_id
   MASTER_KEY=your_master_key
   DATABASE_URI=mongodb://username:password@host:port/database
   SERVER_URL=https://your-server-url.com/parse
   
   # Blockchain
   ETH_NODE_URI_BASESEP=https://sepolia.base.org
   CHAIN_ID=base-sepolia
   PRIVATE_KEY=your_private_key
   
   # External Services
   DFNS_APP_ID=your_dfns_app_id
   DFNS_API_URL=https://api.dfns.io
   DFNS_CRED_ID=your_dfns_credential_id
   
   # Additional Configuration
   METADATA_BASE_URI=https://metadata.gemforce.com/
   EOL
   ```

2. **Configure database**:
   - Set up MongoDB
   - Create database user
   - Configure connection string

3. **Set up web server** (nginx example):

   ```nginx
   # /etc/nginx/sites-available/gemforce
   server {
       listen 80;
       server_name api.gemforce.com;
       
       location / {
           return 301 https://$host$request_uri;
       }
   }
   
   server {
       listen 443 ssl;
       server_name api.gemforce.com;
       
       ssl_certificate /etc/letsencrypt/live/api.gemforce.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/api.gemforce.com/privkey.pem;
       
       location / {
           proxy_pass http://localhost:1337;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

### Database Migration

1. **Create database migrations**:

   ```javascript
   // Example migration script (migrations/001_add_indexes.js)
   async function up(db) {
     await db.collection('User').createIndex({ walletAddress: 1 });
     await db.collection('Identity').createIndex({ walletAddress: 1 }, { unique: true });
     await db.collection('Transaction').createIndex({ hash: 1 }, { unique: true });
   }
   
   async function down(db) {
     await db.collection('User').dropIndex({ walletAddress: 1 });
     await db.collection('Identity').dropIndex({ walletAddress: 1 });
     await db.collection('Transaction').dropIndex({ hash: 1 });
   }
   
   module.exports = { up, down };
   ```

2. **Run migrations**:

   ```bash
   # Example using a simple migration tool
   npx migrate-mongo up
   ```

3. **Verify migrations**:

   ```javascript
   // Check indexes
   db.User.getIndexes()
   db.Identity.getIndexes()
   db.Transaction.getIndexes()
   ```

### WebSocket Setup

1. **Configure WebSocket endpoints**:

   ```javascript
   // Example WebSocket server setup
   const wss = new WebSocketServer({ 
     server: httpServer,
     path: "/ws"
   });
   
   wss.on('connection', (ws) => {
     // Handle connection
     ws.on('message', (message) => {
       // Handle message
     });
     
     ws.on('close', () => {
       // Handle disconnection
     });
   });
   ```

2. **Set up blockchain event listeners**:

   ```javascript
   // Example blockchain event listener
   function setupEventListeners(provider, diamondAddress) {
     const diamond = new ethers.Contract(
       diamondAddress,
       DiamondABI,
       provider
     );
     
     diamond.on("Transfer", (from, to, tokenId) => {
       // Handle transfer event
       // Broadcast to connected WebSocket clients
       wss.clients.forEach((client) => {
         if (client.readyState === WebSocket.OPEN) {
           client.send(JSON.stringify({
             event: "Transfer",
             data: { from, to, tokenId: tokenId.toString() }
           }));
         }
       });
     });
   }
   ```

## Environment Configuration

### Development Environment

1. **Local environment setup**:

   ```bash
   # Start local hardhat node
   npx hardhat node
   
   # Deploy contracts to local node
   npx hardhat deploy --network localhost
   
   # Start Parse Server locally
   npm run dev
   ```

2. **Environment configuration file for development**:

   ```
   # .env.development
   APP_ID=gemforce_dev
   MASTER_KEY=your_dev_master_key
   DATABASE_URI=mongodb://localhost:27017/gemforce_dev
   SERVER_URL=http://localhost:1337/parse
   
   # Use local hardhat node
   ETH_NODE_URI_LOCAL=http://localhost:8545
   CHAIN_ID=31337
   
   # Development keys
   PRIVATE_KEY=your_dev_private_key
   ```

3. **Development database setup**:

   ```bash
   # Start MongoDB locally
   mongod --dbpath ./data
   
   # Create database and user
   mongo
   > use gemforce_dev
   > db.createUser({
       user: "gemforce_user",
       pwd: "password",
       roles: [{ role: "readWrite", db: "gemforce_dev" }]
     })
   ```

### Testing Environment

1. **Testing environment configuration**:

   ```
   # .env.test
   APP_ID=gemforce_test
   MASTER_KEY=your_test_master_key
   DATABASE_URI=mongodb://localhost:27017/gemforce_test
   SERVER_URL=http://localhost:1337/parse
   
   # Use BaseSepolia for testing
   ETH_NODE_URI_BASESEP=https://sepolia.base.org
   CHAIN_ID=84532
   
   # Test keys
   PRIVATE_KEY=your_test_private_key
   ```

2. **Automated testing setup**:

   ```bash
   # Run tests
   npm test
   
   # Run specific tests
   npx mocha test/specific-test.js
   ```

3. **Test database initialization**:

   ```javascript
   // Initialize test database
   before(async () => {
     const client = await MongoClient.connect(process.env.DATABASE_URI);
     const db = client.db();
     await db.collection('User').deleteMany({});
     await db.collection('Identity').deleteMany({});
     // Seed with test data
     await db.collection('User').insertMany(testUsers);
   });
   ```

### Staging Environment

1. **Staging environment configuration**:

   ```
   # .env.staging
   APP_ID=gemforce_staging
   MASTER_KEY=your_staging_master_key
   DATABASE_URI=mongodb://user:password@staging-db:27017/gemforce_staging
   SERVER_URL=https://staging-api.gemforce.com/parse
   
   # Use BaseSepolia for staging
   ETH_NODE_URI_BASESEP=https://sepolia.base.org
   CHAIN_ID=84532
   
   # Staging keys
   PRIVATE_KEY=your_staging_private_key
   ```

2. **Staging deployment**:

   ```bash
   # Deploy to staging
   npm run deploy:staging
   
   # Run database migrations
   npm run migrate:staging
   ```

3. **Staging verification**:

   ```bash
   # Verify staging deployment
   curl https://staging-api.gemforce.com/parse/health
   ```

### Production Environment

1. **Production environment configuration**:

   ```
   # .env.production
   APP_ID=gemforce_prod
   MASTER_KEY=your_production_master_key
   DATABASE_URI=mongodb://user:password@production-db:27017/gemforce_production
   SERVER_URL=https://api.gemforce.com/parse
   
   # Use production endpoints
   ETH_NODE_URI_MAINNET=https://mainnet.infura.io/v3/your_infura_key
   CHAIN_ID=1
   
   # Production keys stored securely
   # PRIVATE_KEY should be handled with extra security
   ```

2. **Production deployment steps**:

   ```bash
   # Deploy to production
   npm run deploy:production
   
   # Run database migrations
   npm run migrate:production
   ```

3. **Production verification**:

   ```bash
   # Verify production deployment
   curl https://api.gemforce.com/parse/health
   
   # Monitor logs
   pm2 logs gemforce-server
   ```

### Environment-Specific Settings

1. **Configuration management**:

   ```javascript
   // Load environment-specific configuration
   const envConfig = require(`./config/${process.env.NODE_ENV || 'development'}`);
   
   module.exports = {
     // Base configuration
     appName: 'Gemforce',
     
     // Merge with environment-specific config
     ...envConfig
   };
   ```

2. **Feature flags**:

   ```javascript
   // Example feature flags configuration
   const featureFlags = {
     development: {
       enableNewMarketplace: true,
       enableCarbonCredits: true
     },
     staging: {
       enableNewMarketplace: true,
       enableCarbonCredits: false
     },
     production: {
       enableNewMarketplace: false,
       enableCarbonCredits: false
     }
   };
   
   // Usage
   if (featureFlags[process.env.NODE_ENV].enableNewMarketplace) {
     // Initialize new marketplace
   }
   ```

## Deployment Automation

### CI/CD Pipeline Setup

1. **GitHub Actions workflow**:

   ```yaml
   # .github/workflows/deploy.yml
   name: Deploy Gemforce
   
   on:
     push:
       branches: [ main, staging ]
   
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         - name: Setup Node.js
           uses: actions/setup-node@v2
           with:
             node-version: '16'
         - name: Install dependencies
           run: npm ci
         - name: Run tests
           run: npm test
           
     build:
       needs: test
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         - name: Setup Node.js
           uses: actions/setup-node@v2
           with:
             node-version: '16'
         - name: Install dependencies
           run: npm ci
         - name: Build project
           run: npm run build
         - name: Upload build artifacts
           uses: actions/upload-artifact@v2
           with:
             name: build
             path: dist/
             
     deploy:
       needs: build
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         - name: Download build artifacts
           uses: actions/download-artifact@v2
           with:
             name: build
             path: dist/
         - name: Deploy to server
           uses: appleboy/ssh-action@master
           with:
             host: ${{ secrets.SERVER_HOST }}
             username: ${{ secrets.SERVER_USERNAME }}
             key: ${{ secrets.SSH_PRIVATE_KEY }}
             script: |
               cd /var/www/gemforce
               git pull
               npm ci
               cp -r ${{ github.workspace }}/dist/* ./dist/
               pm2 restart gemforce-server
   ```

2. **Automatic testing**:

   ```yaml
   # .github/workflows/test.yml
   name: Test Gemforce
   
   on:
     pull_request:
       branches: [ main, staging ]
   
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v2
         - name: Setup Node.js
           uses: actions/setup-node@v2
           with:
             node-version: '16'
         - name: Install dependencies
           run: npm ci
         - name: Run linting
           run: npm run lint
         - name: Run tests
           run: npm test
   ```

### Automated Testing

1. **Unit tests for smart contracts**:

   ```javascript
   // test/GemforceMinter.test.ts
   describe("GemforceMinter", function() {
     let owner, user;
     let diamond, gemforceMinter;
     
     beforeEach(async function() {
       [owner, user] = await ethers.getSigners();
       
       // Deploy diamond with GemforceMinterFacet
       // ...deployment code
       
       gemforceMinter = await ethers.getContractAt("GemforceMinterFacet", diamond.address);
     });
     
     it("should mint a token with metadata", async function() {
       const metadata = [
         { key: "name", attributeType: 0, value: "Carbon Credit" },
         { key: "amount", attributeType: 1, value: "100" }
       ];
       
       await expect(gemforceMinter.connect(owner).gemforceMint(metadata))
         .to.emit(gemforceMinter, "GemforceMinted")
         .withArgs(0, owner.address, metadata);
     });
   });
   ```

2. **API endpoint tests**:

   ```javascript
   // test/cloud-functions.test.js
   describe("Cloud Functions", function() {
     before(async function() {
       // Initialize Parse Server for testing
       Parse.initialize("test_app_id", "test_js_key", "test_master_key");
       Parse.serverURL = "http://localhost:1337/parse";
     });
     
     it("should retrieve blockchain data", async function() {
       const result = await Parse.Cloud.run("loadAllBlockchains");
       expect(result).to.be.an("array");
     });
   });
   ```

3. **End-to-end tests**:

   ```javascript
   // test/e2e/user-flow.test.js
   describe("User Flow", function() {
     it("should register, create identity, and mint token", async function() {
       // Register user
       const user = await registerUser("test@example.com", "password");
       
       // Create DFNS wallet
       const { walletId } = await createDFNSWallet(user);
       
       // Create identity
       const { identityAddress } = await createIdentity(user, walletId);
       
       // Mint token
       const { tokenId } = await mintToken(user, walletId);
       
       // Verify token ownership
       const owner = await getTokenOwner(tokenId);
       expect(owner).to.equal(user.get("walletAddress"));
     });
   });
   ```

### Deployment Scripts

1. **Smart contract deployment script**:

   ```javascript
   // scripts/deploy-contracts.js
   async function main() {
     // Get deployer
     const [deployer] = await ethers.getSigners();
     console.log("Deploying contracts with account:", deployer.address);
     
     // Deploy libraries
     const LibraryA = await ethers.getContractFactory("LibraryA");
     const libraryA = await LibraryA.deploy();
     await libraryA.deployed();
     console.log("LibraryA deployed to:", libraryA.address);
     
     // Deploy facets with libraries
     const FacetA = await ethers.getContractFactory("FacetA", {
       libraries: {
         LibraryA: libraryA.address
       }
     });
     const facetA = await FacetA.deploy();
     await facetA.deployed();
     console.log("FacetA deployed to:", facetA.address);
     
     // More deployments...
   }
   
   main()
     .then(() => process.exit(0))
     .catch(error => {
       console.error(error);
       process.exit(1);
     });
   ```

2. **Parse Server deployment script**:

   ```bash
   #!/bin/bash
   # scripts/deploy-parse.sh
   
   # Load environment variables
   source .env.${NODE_ENV:-production}
   
   # Build the project
   echo "Building project..."
   npm run build
   
   # Deploy to server
   echo "Deploying to server..."
   rsync -avz --exclude node_modules --exclude .git . user@server:/var/www/gemforce/
   
   # SSH into server and restart
   ssh user@server << EOF
     cd /var/www/gemforce
     npm ci --production
     pm2 restart gemforce-server
   EOF
   
   echo "Deployment complete!"
   ```

### Infrastructure as Code

1. **Terraform configuration**:

   ```hcl
   # main.tf
   provider "aws" {
     region = "us-east-1"
   }
   
   resource "aws_instance" "gemforce_server" {
     ami           = "ami-0c55b159cbfafe1f0"
     instance_type = "t2.medium"
     key_name      = "gemforce-key"
     
     tags = {
       Name = "gemforce-server"
       Environment = var.environment
     }
     
     root_block_device {
       volume_size = 50
       volume_type = "gp2"
     }
     
     vpc_security_group_ids = [aws_security_group.gemforce_sg.id]
   }
   
   resource "aws_security_group" "gemforce_sg" {
     name        = "gemforce-sg"
     description = "Allow web and SSH traffic"
     
     ingress {
       from_port   = 80
       to_port     = 80
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }
     
     ingress {
       from_port   = 443
       to_port     = 443
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }
     
     ingress {
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = ["0.0.0.0/0"]
     }
     
     egress {
       from_port   = 0
       to_port     = 0
       protocol    = "-1"
       cidr_blocks = ["0.0.0.0/0"]
     }
   }
   
   output "server_ip" {
     value = aws_instance.gemforce_server.public_ip
   }
   ```

2. **Docker Compose setup**:

   ```yaml
   # docker-compose.yml
   version: '3'
   
   services:
     mongodb:
       image: mongo:4.4
       ports:
         - "27017:27017"
       volumes:
         - mongo-data:/data/db
       environment:
         MONGO_INITDB_ROOT_USERNAME: ${MONGO_USERNAME}
         MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
       restart: always
       
     parse-server:
       build: .
       ports:
         - "1337:1337"
       environment:
         - APP_ID=${APP_ID}
         - MASTER_KEY=${MASTER_KEY}
         - DATABASE_URI=mongodb://${MONGO_USERNAME}:${MONGO_PASSWORD}@mongodb:27017/${DATABASE_NAME}?authSource=admin
         - SERVER_URL=${SERVER_URL}
         - CLOUD_PATH=/parse-server/cloud/main.js
       volumes:
         - ./cloud:/parse-server/cloud
       depends_on:
         - mongodb
       restart: always
       
     nginx:
       image: nginx:latest
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - ./nginx/conf.d:/etc/nginx/conf.d
         - ./nginx/ssl:/etc/nginx/ssl
         - ./nginx/www:/var/www/html
       depends_on:
         - parse-server
       restart: always
       
   volumes:
     mongo-data:
   ```

### Continuous Monitoring

1. **Setup monitoring tools**:

   ```javascript
   // Monitoring setup in app.js
   const prometheus = require('prom-client');
   const collectDefaultMetrics = prometheus.collectDefaultMetrics;
   
   // Enable default metrics
   collectDefaultMetrics({ timeout: 5000 });
   
   // Custom metrics
   const httpRequestDurationMicroseconds = new prometheus.Histogram({
     name: 'http_request_duration_ms',
     help: 'Duration of HTTP requests in ms',
     labelNames: ['method', 'route', 'status_code'],
     buckets: [0.1, 5, 15, 50, 100, 500]
   });
   
   // Endpoint for metrics
   app.get('/metrics', (req, res) => {
     res.set('Content-Type', prometheus.register.contentType);
     res.end(prometheus.register.metrics());
   });
   ```

2. **Logging configuration**:

   ```javascript
   // Logging setup in app.js
   const winston = require('winston');
   
   const logger = winston.createLogger({
     level: process.env.LOG_LEVEL || 'info',
     format: winston.format.combine(
       winston.format.timestamp(),
       winston.format.json()
     ),
     transports: [
       new winston.transports.Console(),
       new winston.transports.File({ filename: 'error.log', level: 'error' }),
       new winston.transports.File({ filename: 'combined.log' })
     ]
   });
   
   // Use in application
   logger.info('Server started', { port: 1337 });
   ```

## Upgrade Procedures

### Smart Contract Upgrades

1. **Upgrading a facet**:

   ```javascript
   // scripts/upgrade-facet.js
   async function main() {
     // Get signer
     const [signer] = await ethers.getSigners();
     
     // Deploy new version of the facet
     const NewFacet = await ethers.getContractFactory("NewFacetV2");
     const newFacet = await NewFacet.deploy();
     await newFacet.deployed();
     console.log("New facet deployed to:", newFacet.address);
     
     // Get diamond contract
     const diamond = await ethers.getContractAt("Diamond", DIAMOND_ADDRESS);
     
     // Get selectors for the facet
     const selectors = getSelectors(newFacet);
     
     // Create facet cut
     const facetCut = {
       facetAddress: newFacet.address,
       action: FacetCutAction.Replace, // Replace existing facet
       functionSelectors: selectors
     };
     
     // Perform the upgrade
     const tx = await diamond.diamondCut(
       [facetCut],
       ethers.constants.AddressZero, // No initialization
       "0x"
     );
     await tx.wait();
     
     console.log("Facet upgraded successfully");
   }
   
   main()
     .then(() => process.exit(0))
     .catch(error => {
       console.error(error);
       process.exit(1);
     });
   ```

2. **Adding a new facet**:

   ```javascript
   // scripts/add-facet.js
   async function main() {
     // Get signer
     const [signer] = await ethers.getSigners();
     
     // Deploy new facet
     const NewFacet = await ethers.getContractFactory("NewFacet");
     const newFacet = await NewFacet.deploy();
     await newFacet.deployed();
     console.log("New facet deployed to:", newFacet.address);
     
     // Get diamond contract
     const diamond = await ethers.getContractAt("Diamond", DIAMOND_ADDRESS);
     
     // Get selectors for the facet
     const selectors = getSelectors(newFacet);
     
     // Create facet cut
     const facetCut = {
       facetAddress: newFacet.address,
       action: FacetCutAction.Add, // Add new facet
       functionSelectors: selectors
     };
     
     // Perform the upgrade
     const tx = await diamond.diamondCut(
       [facetCut],
       ethers.constants.AddressZero, // No initialization
       "0x"
     );
     await tx.wait();
     
     console.log("Facet added successfully");
   }
   ```

3. **Upgrading Diamond implementation**:

   ```javascript
   // scripts/upgrade-diamond.js
   async function main() {
     // Get diamond factory
     const diamondFactory = await ethers.getContractAt("DiamondFactory", FACTORY_ADDRESS);
     
     // Get existing diamond
     const diamond = await ethers.getContractAt("Diamond", DIAMOND_ADDRESS);
     
     // Deploy new facets
     const newFacets = await deployNewFacets();
     
     // Create facet cuts
     const facetCuts = createFacetCuts(newFacets);
     
     // Set new facet set on factory
     await diamondFactory.setFacets("newFacetSet", facetCuts);
     
     // Deploy new diamond initializer if needed
     const diamondInit = await deployDiamondInit();
     
     // Prepare initialization data
     const initData = diamondInit.interface.encodeFunctionData("init", [
       /* initialization parameters */
     ]);
     
     // Upgrade diamond
     const tx = await diamond.diamondCut(
       facetCuts,
       diamondInit.address,
       initData
     );
     await tx.wait();
     
     console.log("Diamond upgraded successfully");
   }
   ```

### Cloud Function Updates

1. **Updating cloud functions**:

   ```bash
   # Update cloud functions
   git pull origin main
   npm install
   npm run build
   
   # Restart Parse Server
   pm2 restart gemforce-server
   ```

2. **Testing cloud function updates**:

   ```javascript
   // test/cloud-functions/updated-function.test.js
   describe("Updated Cloud Function", function() {
     it("should handle new functionality", async function() {
       const result = await Parse.Cloud.run("updatedFunction", { param: "value" });
       expect(result).to.have.property("newProperty");
     });
   });
   ```

3. **Deploying specific cloud function changes**:

   ```bash
   # Deploy specific cloud function changes
   scp dist/src/cloud-functions/specific-function.js user@server:/var/www/gemforce/dist/src/cloud-functions/
   
   # Restart Parse Server
   ssh user@server "cd /var/www/gemforce && pm2 restart gemforce-server"
   ```

### Database Schema Migrations

1. **Creating a migration**:

   ```javascript
   // migrations/1625000000000_add_new_field.js
   exports.up = async (db) => {
     // Add new field to all documents in a collection
     await db.collection('User').updateMany(
       { newField: { $exists: false } }, 
       { $set: { newField: "" } }
     );
     
     // Create new index
     await db.collection('User').createIndex({ newField: 1 });
   };
   
   exports.down = async (db) => {
     // Remove the field
     await db.collection('User').updateMany(
       {}, 
       { $unset: { newField: "" } }
     );
     
     // Remove the index
     await db.collection('User').dropIndex({ newField: 1 });
   };
   ```

2. **Running migrations**:

   ```bash
   # Run migrations
   npx migrate-mongo up
   
   # Check migration status
   npx migrate-mongo status
   ```

3. **Rolling back migrations**:

   ```bash
   # Roll back last migration
   npx migrate-mongo down
   
   # Roll back to specific migration
   npx migrate-mongo down 1625000000000
   ```

### Backward Compatibility Considerations

1. **API versioning**:

   ```javascript
   // Example API versioning
   app.use('/api/v1', v1Routes);
   app.use('/api/v2', v2Routes);
   
   // Redirect old routes
   app.use('/api/legacy/:resource', (req, res) => {
     res.redirect(`/api/v1/${req.params.resource}`);
   });
   ```

2. **Smart contract compatibility**:

   - Never remove storage variables
   - Add new functions rather than changing existing ones
   - Use new facets for significant changes
   - Use feature flags to control access to new features

3. **Client compatibility**:

   ```javascript
   // Check client version and provide appropriate response
   Parse.Cloud.define("getFeatures", async (request) => {
     const clientVersion = request.params.clientVersion;
     
     if (semver.lt(clientVersion, "2.0.0")) {
       return legacyFeatureList;
     } else {
       return newFeatureList;
     }
   });
   ```

### Feature Flagging

1. **Implementing feature flags**:

   ```javascript
   // Feature flag configuration
   const featureFlags = {
     newMarketplace: process.env.ENABLE_NEW_MARKETPLACE === "true",
     carbonCredits: process.env.ENABLE_CARBON_CREDITS === "true",
     nftFractionalization: process.env.ENABLE_NFT_FRACTIONALIZATION === "true"
   };
   
   // Using feature flags
   Parse.Cloud.define("getMarketplace", async (request) => {
     if (featureFlags.newMarketplace) {
       return getNewMarketplace();
     } else {
       return getLegacyMarketplace();
     }
   });
   ```

2. **Controlling access to new features**:

   ```javascript
   // Progressive rollout
   Parse.Cloud.define("checkFeatureAccess", async (request) => {
     const { feature, userId } = request.params;
     
     // Check if feature is enabled globally
     if (!featureFlags[feature]) {
       return { hasAccess: false };
     }
     
     // Check if user is in beta group
     const user = await new Parse.Query(Parse.User)
       .get(userId, { useMasterKey: true });
     
     const isBetaTester = user.get("betaTester") === true;
     
     // Calculate percentage-based rollout
     const rolloutPercentage = 25; // 25% of users
     const userIdNumber = parseInt(userId.substring(0, 8), 16);
     const userPercentile = userIdNumber % 100;
     
     return {
       hasAccess: isBetaTester || userPercentile < rolloutPercentage
     };
   });
   ```

## Rollback Procedures

### Smart Contract Rollbacks

1. **Rollback strategy for facets**:

   ```javascript
   // scripts/rollback-facet.js
   async function main() {
     // Get diamond contract
     const diamond = await ethers.getContractAt("Diamond", DIAMOND_ADDRESS);
     
     // Get previous version of the facet
     const previousFacetAddress = PREVIOUS_FACET_ADDRESS;
     
     // Get selectors for the facet
     const selectors = SELECTORS;
     
     // Create facet cut for rollback
     const facetCut = {
       facetAddress: previousFacetAddress,
       action: FacetCutAction.Replace,
       functionSelectors: selectors
     };
     
     // Perform the rollback
     const tx = await diamond.diamondCut(
       [facetCut],
       ethers.constants.AddressZero,
       "0x"
     );
     await tx.wait();
     
     console.log("Facet rolled back successfully");
   }
   ```

2. **Diamond upgrade rollback**:

   ```javascript
   // scripts/rollback-diamond.js
   async function main() {
     // Get diamond factory
     const diamondFactory = await ethers.getContractAt("DiamondFactory", FACTORY_ADDRESS);
     
     // Get diamond to rollback
     const diamond = await ethers.getContractAt("Diamond", DIAMOND_ADDRESS);
     
     // Use previous facet set
     const previousFacetSet = "previousFacetSet";
     
     // Get facets from the set
     const facets = await diamondFactory.getFacets(previousFacetSet);
     
     // Rollback diamond
     const tx = await diamond.diamondCut(
       facets,
       ethers.constants.AddressZero,
       "0x"
     );
     await tx.wait();
     
     console.log("Diamond rolled back successfully");
   }
   ```

3. **Emergency pause**:

   ```javascript
   // scripts/emergency-pause.js
   async function main() {
     // Get contract
     const contract = await ethers.getContractAt("PausableFacet", DIAMOND_ADDRESS);
     
     // Pause the contract
     const tx = await contract.pause();
     await tx.wait();
     
     console.log("Contract paused successfully");
   }
   ```

### Cloud Function Rollbacks

1. **Rolling back with Git**:

   ```bash
   # Rollback to previous commit
   git reset --hard HEAD~1
   npm install
   npm run build
   
   # Restart server
   pm2 restart gemforce-server
   ```

2. **Using deployment tags**:

   ```bash
   # List tags
   git tag -l
   
   # Checkout specific version
   git checkout v1.2.3
   npm install
   npm run build
   
   # Restart server
   pm2 restart gemforce-server
   ```

3. **Specific file rollback**:

   ```bash
   # Revert specific file
   git checkout HEAD~1 -- src/cloud-functions/specific-file.ts
   npm run build
   
   # Restart server
   pm2 restart gemforce-server
   ```

### Database Rollbacks

1. **Restoring from backup**:

   ```bash
   # Restore MongoDB from backup
   mongorestore --uri="mongodb://username:password@host:port/database" --drop /backup/path/YYYY-MM-DD
   ```

2. **Rolling back a migration**:

   ```bash
   # Roll back the last migration
   npx migrate-mongo down
   ```

3. **Manual data correction script**:

   ```javascript
   // scripts/correct-data.js
   async function main() {
     const MongoClient = require('mongodb').MongoClient;
     const client = await MongoClient.connect(process.env.DATABASE_URI);
     const db = client.db();
     
     try {
       // Correct data issues
       await db.collection('User').updateMany(
         { incorrectField: { $exists: true } },
         { $rename: { "incorrectField": "correctField" } }
       );
       
       console.log("Data correction complete");
     } catch (error) {
       console.error("Error correcting data:", error);
     } finally {
       await client.close();
     }
   }
   
   main().catch(console.error);
   ```

### Emergency Procedures

1. **Complete service rollback**:

   ```bash
   # Emergency rollback script
   #!/bin/bash
   # scripts/emergency-rollback.sh
   
   # Stop the current service
   pm2 stop gemforce-server
   
   # Restore code from known good state
   git checkout v1.2.3
   npm install
   npm run build
   
   # Restore database
   mongorestore --uri="$DATABASE_URI" --drop /backups/latest
   
   # Restart the service
   pm2 start gemforce-server
   
   # Notify team
   curl -X POST -H "Content-Type: application/json" \
     -d '{"text":"Emergency rollback performed to v1.2.3"}' \
     $SLACK_WEBHOOK_URL
   ```

2. **Read-only mode**:

   ```javascript
   // Enable read-only mode
   Parse.Cloud.beforeSave("*", async () => {
     if (process.env.READONLY_MODE === "true") {
       throw new Parse.Error(
         Parse.Error.OPERATION_FORBIDDEN,
         "System is currently in read-only mode for maintenance."
       );
     }
   });
   
   Parse.Cloud.beforeDelete("*", async () => {
     if (process.env.READONLY_MODE === "true") {
       throw new Parse.Error(
         Parse.Error.OPERATION_FORBIDDEN,
         "System is currently in read-only mode for maintenance."
       );
     }
   });
   ```

### Data Integrity Verification

1. **Verifying contract state**:

   ```javascript
   // scripts/verify-contract-state.js
   async function main() {
     // Get contract
     const contract = await ethers.getContractAt("DiamondLoupe", DIAMOND_ADDRESS);
     
     // Get all facets
     const facets = await contract.facets();
     
     // Verify each facet
     for (const facet of facets) {
       console.log(`Verifying facet at ${facet.facetAddress}`);
       
       // Get facet code
       const code = await ethers.provider.getCode(facet.facetAddress);
       
       // Check code is not empty
       if (code === "0x" || code === "0x0") {
         throw new Error(`Facet at ${facet.facetAddress} has no code!`);
       }
       
       // Verify selectors
       for (const selector of facet.functionSelectors) {
         const result = await contract.facetAddress(selector);
         if (result !== facet.facetAddress) {
           throw new Error(`Selector ${selector} points to wrong facet!`);
         }
       }
     }
     
     console.log("Contract state verification successful");
   }
   
   main().catch(console.error);
   ```

2. **Database integrity check**:

   ```javascript
   // scripts/verify-database.js
   async function main() {
     const MongoClient = require('mongodb').MongoClient;
     const client = await MongoClient.connect(process.env.DATABASE_URI);
     const db = client.db();
     
     try {
       // Check User collection integrity
       const userCount = await db.collection('User').countDocuments();
       console.log(`User count: ${userCount}`);
       
       // Check for duplicate emails
       const duplicateEmails = await db.collection('User').aggregate([
         { $group: { _id: "$email", count: { $sum: 1 } } },
         { $match: { count: { $gt: 1 } } }
       ]).toArray();
       
       if (duplicateEmails.length > 0) {
         console.error(`Found ${duplicateEmails.length} duplicate emails!`);
         console.error(duplicateEmails);
       }
       
       // Verify indexes
       const indexes = await db.collection('User').indexes();
       console.log("Indexes:", indexes);
       
       // More integrity checks...
       
       console.log("Database integrity check complete");
     } finally {
       await client.close();
     }
   }
   
   main().catch(console.error);
   ```

## Testing and Verification

### Unit Testing

1. **Smart contract unit tests**:

   ```javascript
   // test/unit/GemforceMinter.test.ts
   describe("GemforceMinter", function() {
     let owner, user1, user2;
     let diamond, gemforceMinter;
     
     beforeEach(async function() {
       [owner, user1, user2] = await ethers.getSigners();
       
       // Deploy diamond with facets
       // ...deployment code
       
       gemforceMinter = await ethers.getContractAt("GemforceMinterFacet", diamond.address);
     });
     
     it("should allow owner to mint", async function() {
       const metadata = [
         { key: "name", attributeType: 0, value: "Carbon Credit" }
       ];
       
       const tx = await gemforceMinter.connect(owner).gemforceMint(metadata);
       const receipt = await tx.wait();
       
       // Check events
       const event = receipt.events.find(e => e.event === "GemforceMinted");
       expect(event).to.exist;
       expect(event.args.tokenId).to.equal(0);
       expect(event.args.minter).to.equal(owner.address);
     });
     
     it("should revert when non-owner tries to mint", async function() {
       const metadata = [
         { key: "name", attributeType: 0, value: "Carbon Credit" }
       ];
       
       await expect(
         gemforceMinter.connect(user1).gemforceMint(metadata)
       ).to.be.revertedWith("Only contract owner");
     });
   });
   ```

2. **Cloud function unit tests**:

   ```javascript
   // test/unit/cloud-functions.test.js
   describe("Cloud Functions Unit Tests", function() {
     before(function() {
       // Mock Parse
       this.originalParse = global.Parse;
       global.Parse = {
         Cloud: {
           define: (name, handler) => {
             this.cloudFunctions[name] = handler;
           }
         },
         Error: {
           INVALID_PARAMS: 141
         }
       };
       
       // Load cloud functions
       this.cloudFunctions = {};
       require("../../src/cloud-functions/contracts");
     });
     
     after(function() {
       global.Parse = this.originalParse;
     });
     
     it("should validate parameters in addDiamondFacet", async function() {
       const handler = this.cloudFunctions.addDiamondFacet;
       
       // Missing parameters
       try {
         await handler({ params: {} });
         assert.fail("Should have thrown an error");
       } catch (e) {
         expect(e.code).to.equal(Parse.Error.INVALID_PARAMS);
       }
       
       // With valid parameters (mocked)
       const mockResult = { success: true };
       this.addDiamondFacet = sinon.stub().resolves(mockResult);
       
       const result = await handler({
         params: {
           networkId: "1",
           diamondAddress: "0x123",
           facetName: "TestFacet"
         }
       });
       
       expect(result).to.deep.equal(mockResult);
     });
   });
   ```

### Integration Testing

1. **Smart contract integration tests**:

   ```javascript
   // test/integration/marketplace-flow.test.ts
   describe("Marketplace Integration", function() {
     let owner, seller, buyer;
     let diamond, marketplace, gem, treasury;
     
     before(async function() {
       [owner, seller, buyer] = await ethers.getSigners();
       
       // Deploy all contracts
       // ...deployment code
       
       // Get contract instances
       diamond = await ethers.getContractAt("Diamond", diamondAddress);
       marketplace = await ethers.getContractAt("MarketplaceFacet", diamondAddress);
       gem = await ethers.getContractAt("GemforceMinterFacet", diamondAddress);
       treasury = await ethers.getContractAt("Treasury", treasuryAddress);
     });
     
     it("should support full marketplace flow", async function() {
       // Mint token
       const metadata = [{ key: "test", attributeType: 0, value: "value" }];
       await gem.connect(owner).gemforceMint(metadata);
       
       // List token
       await marketplace.connect(owner).listItem(0, ethers.utils.parseEther("1.0"));
       
       // Check listing
       const listing = await marketplace.getListing(0);
       expect(listing.price).to.equal(ethers.utils.parseEther("1.0"));
       
       // Purchase token
       await marketplace.connect(buyer).purchaseItem(diamondAddress, 0, {
         value: ethers.utils.parseEther("1.0")
       });
       
       // Check ownership
       const newOwner = await diamond.ownerOf(0);
       expect(newOwner).to.equal(buyer.address);
       
       // Check treasury balance
       const balance = await treasury.getBalance();
       expect(balance).to.be.gt(0);
     });
   });
   ```

2. **API integration tests**:

   ```javascript
   // test/integration/api-flow.test.js
   describe("API Integration", function() {
     let user;
     
     before(async function() {
       // Initialize Parse
       Parse.initialize("test_app_id", "test_js_key", "test_master_key");
       Parse.serverURL = "http://localhost:1337/parse";
       
       // Create test user
       user = new Parse.User();
       user.set("username", "test@example.com");
       user.set("password", "password");
       user.set("email", "test@example.com");
       await user.signUp();
     });
     
     it("should handle user blockchain operations", async function() {
       // Get blockchains
       const blockchains = await Parse.Cloud.run("loadAllBlockchains");
       expect(blockchains).to.be.an("array");
       
       // Create DFNS wallet (mocked)
       const walletResult = await Parse.Cloud.run("registerInit", {
         username: user.get("email")
       });
       expect(walletResult).to.have.property("challenge");
       
       // Complete registration (mocked)
       const registrationResult = await Parse.Cloud.run("registerComplete", {
         signedChallenge: "mocked_challenge",
         temporaryAuthenticationToken: "mocked_token"
       });
       expect(registrationResult).to.have.property("token");
       
       // List wallets
       const wallets = await Parse.Cloud.run("listWallets", {
         authToken: registrationResult.token
       });
       expect(wallets).to.have.property("wallets");
     });
   });
   ```

### Contract Verification

1. **Verifying contracts on block explorer**:

   ```bash
   # Verify contract on Etherscan or Basescan
   npx hardhat verify --network baseSepolia <CONTRACT_ADDRESS> <CONSTRUCTOR_ARGS>
   
   # Verify contract with libraries
   npx hardhat verify --network baseSepolia <CONTRACT_ADDRESS> <CONSTRUCTOR_ARGS> \
     --libraries Library1=0x123... Library2=0x456...
   ```

2. **Automated verification script**:

   ```javascript
   // scripts/verify-contracts.js
   async function main() {
     // Get deployment data
     const deployments = require('../deployments.json');
     const network = process.env.NETWORK || 'baseSepolia';
     const networkId = hre.config.networks[network].chainId;
     
     // Verify Diamond contract
     const diamondAddress = deployments[networkId].diamondAddress;
     console.log(`Verifying Diamond at ${diamondAddress}`);
     
     try {
       await hre.run("verify:verify", {
         address: diamondAddress,
         constructorArguments: []
       });
     } catch (e) {
       console.log(`Error verifying Diamond: ${e.message}`);
     }
     
     // Verify Facets
     const facets = deployments[networkId].facets || [];
     for (const facet of facets) {
       console.log(`Verifying facet ${facet.name} at ${facet.address}`);
       
       try {
         await hre.run("verify:verify", {
           address: facet.address,
           constructorArguments: []
         });
       } catch (e) {
         console.log(`Error verifying ${facet.name}: ${e.message}`);
       }
     }
   }
   
   main().catch(console.error);
   ```

### Load Testing

1. **Smart contract load testing**:

   ```javascript
   // test/load/contract-load.test.js
   describe("Contract Load Test", function() {
     // Increase timeout for load tests
     this.timeout(300000);
     
     let diamond, minter;
     let signers;
     
     before(async function() {
       // Get signers
       signers = await ethers.getSigners();
       
       // Deploy contracts
       // ...deployment code
       
       diamond = await ethers.getContractAt("Diamond", diamondAddress);
       minter = await ethers.getContractAt("GemforceMinterFacet", diamondAddress);
     });
     
     it("should handle concurrent minting", async function() {
       const concurrentMints = 50;
       const mintPromises = [];
       
       // Create concurrent mint operations
       for (let i = 0; i < concurrentMints; i++) {
         const metadata = [
           { key: "name", attributeType: 0, value: `Token ${i}` },
           { key: "value", attributeType: 1, value: i.toString() }
         ];
         
         mintPromises.push(minter.connect(signers[0]).gemforceMint(metadata));
       }
       
       // Wait for all mints to complete
       const results = await Promise.allSettled(mintPromises);
       
       // Count successful mints
       const successfulMints = results.filter(r => r.status === "fulfilled").length;
       console.log(`Successfully minted ${successfulMints} of ${concurrentMints} tokens`);
       
       // Analyze failures
       const failures = results.filter(r => r.status === "rejected");
       for (const failure of failures) {
         console.log(`Failure reason: ${failure.reason}`);
       }
       
       expect(successfulMints).to.be.gt(0);
     });
   });
   ```

2. **API load testing with Artillery**:

   ```yaml
   # load-tests/api-load.yml
   config:
     target: "https://api.gemforce.com"
     phases:
       - duration: 60
         arrivalRate: 5
         rampTo: 50
         name: "Warm up phase"
       - duration: 120
         arrivalRate: 50
         name: "Sustained load"
       - duration: 60
         arrivalRate: 50
         rampTo: 100
         name: "Peak load"
     defaults:
       headers:
         X-Parse-Application-Id: "{{APP_ID}}"
         Content-Type: "application/json"
   
   scenarios:
     - name: "Load test API endpoints"
       flow:
         - post:
             url: "/parse/functions/loadAllBlockchains"
             json: {}
             expect:
               - statusCode: 200
         - post:
             url: "/parse/functions/loadProviderUrl"
             json:
               networkId: "84532"
             expect:
               - statusCode: 200
         - post:
             url: "/parse/functions/loadSmartContractsForNetwork"
             json:
               networkId: "84532"
             expect:
               - statusCode: 200
   ```

   Run with:

   ```bash
   npx artillery run load-tests/api-load.yml --environment production
   ```

### Security Testing

1. **Smart contract security testing**:

   ```bash
   # Run Slither security analyzer
   slither contracts/
   
   # Run Mythril
   myth analyze contracts/Diamond.sol
   ```

2. **Penetration testing for API**:

   ```bash
   # Run OWASP ZAP scan
   zap-cli quick-scan -s xss,sqli -r report.html https://api.gemforce.com
   ```

3. **Authentication security tests**:

   ```javascript
   // test/security/auth.test.js
   describe("Authentication Security", function() {
     it("should reject invalid authentication", async function() {
       // Try with invalid app ID
       try {
         Parse.initialize("invalid_app_id", "test_js_key");
         await Parse.Cloud.run("loadAllBlockchains");
         assert.fail("Should have thrown an error");
       } catch (e) {
         expect(e.code).to.equal(Parse.Error.INVALID_APP_ID);
       }
       
       // Try with invalid master key
       try {
         Parse.initialize("test_app_id", "test_js_key", "invalid_master_key");
         const user = new Parse.User();
         await user.fetch({ useMasterKey: true });
         assert.fail("Should have thrown an error");
       } catch (e) {
         expect(e.code).to.equal(Parse.Error.INVALID_MASTER_KEY);
       }
     });
   });
   ```

## Network Management

### Blockchain Node Management

1. **Connecting to blockchain nodes**:

   ```javascript
   // src/lib/blockchain.js
   const { ethers } = require("ethers");
   
   // Get provider for network
   function getProvider(networkId) {
     const networks = {
       "1": process.env.ETH_NODE_URI_MAINNET,
       "84532": process.env.ETH_NODE_URI_BASESEP
     };
     
     const nodeUrl = networks[networkId];
     if (!nodeUrl) {
       throw new Error(`No RPC endpoint configured for network ${networkId}`);
     }
     
     return new ethers.providers.JsonRpcProvider(nodeUrl);
   }
   
   // Get WebSocket provider for network
   function getWebSocketProvider(networkId) {
     const networks = {
       "1": process.env.ETH_WSS_URI_MAINNET,
       "84532": process.env.ETH_WSS_URI_BASESEP
     };
     
     const nodeUrl = networks[networkId];
     if (!nodeUrl) {
       throw new Error(`No WebSocket endpoint configured for network ${networkId}`);
     }
     
     return new ethers.providers.WebSocketProvider(nodeUrl);
   }
   
   module.exports = {
     getProvider,
     getWebSocketProvider
   };
   ```

2. **Monitoring node health**:

   ```javascript
   // src/lib/node-monitor.js
   const { getProvider } = require("./blockchain");
   
   async function checkNodeHealth(networkId) {
     try {
       const provider = getProvider(networkId);
       
       // Check if node is responding
       const blockNumber = await provider.getBlockNumber();
       
       // Check if node is synced
       const syncStatus = await provider.send("eth_syncing", []);
       const isSynced = syncStatus === false;
       
       // Check chain ID
       const chainId = await provider.getNetwork().then(net => net.chainId);
       const isCorrectChain = chainId.toString() === networkId;
       
       // Check peers
       const peerCount = await provider.send("net_peerCount", []);
       const peers = parseInt(peerCount, 16);
       const hasPeers = peers > 0;
       
       return {
         isHealthy: isSynced && isCorrectChain && hasPeers,
         blockNumber,
         isSynced,
         isCorrectChain,
         peers
       };
     } catch (error) {
       return {
         isHealthy: false,
         error: error.message
       };
     }
   }
   ```

3. **Node failover**:

   ```javascript
   // src/lib/node-failover.js
   const { ethers } = require("ethers");
   
   class FailoverProvider extends ethers.providers.StaticJsonRpcProvider {
     constructor(urls, network) {
       super(urls[0], network);
       this.urls = urls;
       this.currentUrlIndex = 0;
     }
     
     async send(method, params) {
       try {
         return await super.send(method, params);
       } catch (error) {
         // Try failover
         if (this.urls.length > 1) {
           this.currentUrlIndex = (this.currentUrlIndex + 1) % this.urls.length;
           this.connection.url = this.urls[this.currentUrlIndex];
           console.log(`Failing over to ${this.connection.url}`);
           return await super.send(method, params);
         }
         throw error;
       }
     }
   }
   
   function getFailoverProvider(networkId) {
     const networks = {
       "1": [
         process.env.ETH_NODE_URI_MAINNET_1,
         process.env.ETH_NODE_URI_MAINNET_2,
         process.env.ETH_NODE_URI_MAINNET_3
       ],
       "84532": [
         process.env.ETH_NODE_URI_BASESEP_1,
         process.env.ETH_NODE_URI_BASESEP_2
       ]
     };
     
     const urls = networks[networkId].filter(Boolean);
     if (urls.length === 0) {
       throw new Error(`No RPC endpoints configured for network ${networkId}`);
     }
     
     return new FailoverProvider(urls, parseInt(networkId));
   }
   ```

### RPC Endpoint Configuration

1. **Configuration file for RPC endpoints**:

   ```javascript
   // config/networks.js
   module.exports = {
     mainnet: {
       chainId: 1,
       name: "Ethereum Mainnet",
       rpcEndpoints: [
         {
           url: process.env.ETH_NODE_URI_MAINNET_1,
           provider: "Infura",
           weight: 10
         },
         {
           url: process.env.ETH_NODE_URI_MAINNET_2,
           provider: "Alchemy",
           weight: 5
         },
         {
           url: process.env.ETH_NODE_URI_MAINNET_3,
           provider: "Custom",
           weight: 1
         }
       ],
       wsEndpoints: [
         {
           url: process.env.ETH_WSS_URI_MAINNET,
           provider: "Infura"
         }
       ],
       explorerUrl: "https://etherscan.io"
     },
     baseSepolia: {
       chainId: 84532,
       name: "Base Sepolia",
       rpcEndpoints: [
         {
           url: process.env.ETH_NODE_URI_BASESEP_1,
           provider: "Base",
           weight: 10
         },
         {
           url: process.env.ETH_NODE_URI_BASESEP_2,
           provider: "Alchemy",
           weight: 5
         }
       ],
       wsEndpoints: [
         {
           url: process.env.ETH_WSS_URI_BASESEP,
           provider: "Base"
         }
       ],
       explorerUrl: "https://sepolia.basescan.org"
     }
   };
   ```

2. **RPC endpoint selection logic**:

   ```javascript
   // src/lib/rpc-manager.js
   const networks = require("../../config/networks");
   
   function selectRpcEndpoint(networkId) {
     const network = Object.values(networks).find(n => n.chainId === parseInt(networkId));
     if (!network) {
       throw new Error(`Network ${networkId} not found in configuration`);
     }
     
     // Use weighted selection
     const endpoints = network.rpcEndpoints.filter(e => e.url);
     if (endpoints.length === 0) {
       throw new Error(`No RPC endpoints configured for network ${networkId}`);
     }
     
     // Simple random selection for now
     // Could be expanded to use performance metrics or more complex selection
     const totalWeight = endpoints.reduce((sum, e) => sum + e.weight, 0);
     let random = Math.random() * totalWeight;
     
     for (const endpoint of endpoints) {
       random -= endpoint.weight;
       if (random <= 0) {
         return endpoint.url;
       }
     }
     
     // Fallback to first endpoint
     return endpoints[0].url;
   }
   ```

3. **Environment-specific configuration**:

   ```javascript
   // src/lib/network-config.js
   const networks = require("../../config/networks");
   
   function getNetworkConfig(networkId, environment = process.env.NODE_ENV) {
     const network = Object.values(networks).find(n => n.chainId === parseInt(networkId));
     if (!network) {
       throw new Error(`Network ${networkId} not found in configuration`);
     }
     
     // Apply environment-specific overrides
     const envOverrides = {
       development: {
         rpcEndpoints: [
           { url: "http://localhost:8545", provider: "Local", weight: 100 }
         ]
       },
       test: {
         // Test environment might use special endpoints
       },
       // production uses default configuration
     };
     
     if (envOverrides[environment]) {
       return {
         ...network,
         ...envOverrides[environment]
       };
     }
     
     return network;
   }
   ```

### Transaction Monitoring

1. **Monitoring transaction status**:

   ```javascript
   // src/lib/transaction-monitor.js
   const { getProvider } = require("./blockchain");
   const db = require("./db");
   
   async function monitorTransaction(txHash, networkId) {
     const provider = getProvider(networkId);
     
     // Store the transaction in the database
     await db.collection("Transaction").insertOne({
       hash: txHash,
       networkId,
       status: "pending",
       createdAt: new Date(),
       attempts: 0
     });
     
     // Get initial transaction receipt
     let receipt = await provider.getTransactionReceipt(txHash);
     
     // Transaction is still pending
     if (!receipt) {
       // Schedule check for later
       setTimeout(() => checkTransaction(txHash, networkId), 15000);
       return { status: "pending" };
     }
     
     // Transaction is mined
     updateTransactionStatus(txHash, receipt);
     return {
       status: receipt.status ? "confirmed" : "failed",
       receipt
     };
   }
   
   async function checkTransaction(txHash, networkId) {
     try {
       const provider = getProvider(networkId);
       const receipt = await provider.getTransactionReceipt(txHash);
       
       if (!receipt) {
         // Still pending, check if it's been too long
         const tx = await db.collection("Transaction").findOne({ hash: txHash });
         const age = Date.now() - tx.createdAt.getTime();
         const attempts = tx.attempts + 1;
         
         await db.collection("Transaction").updateOne(
           { hash: txHash },
           { $set: { attempts }, $currentDate: { lastChecked: true } }
         );
         
         if (age > 3600000) { // 1 hour
           // Transaction has been pending too long
           await db.collection("Transaction").updateOne(
             { hash: txHash },
             { $set: { status: "stalled" } }
           );
           // Could notify an admin here
         } else {
           // Check again later
           setTimeout(() => checkTransaction(txHash, networkId), 30000);
         }
         return;
       }
       
       // Transaction is mined, update status
       updateTransactionStatus(txHash, receipt);
     } catch (error) {
       console.error(`Error checking transaction ${txHash}:`, error);
     }
   }
   
   async function updateTransactionStatus(txHash, receipt) {
     const status = receipt.status ? "confirmed" : "failed";
     
     await db.collection("Transaction").updateOne(
       { hash: txHash },
       {
         $set: {
           status,
           blockNumber: receipt.blockNumber,
           gasUsed: receipt.gasUsed.toString(),
           effectiveGasPrice: receipt.effectiveGasPrice.toString()
         }
       }
     );
   }
   ```

2. **Transaction event listener**:

   ```javascript
   // src/lib/event-listener.js
   const { getWebSocketProvider } = require("./blockchain");
   const db = require("./db");
   
   class TransactionEventListener {
     constructor(networkId) {
       this.networkId = networkId;
       this.provider = getWebSocketProvider(networkId);
       this.contracts = new Map();
     }
     
     addContract(address, abi, name) {
       const contract = new ethers.Contract(address, abi, this.provider);
       this.contracts.set(address, { contract, name });
       
       console.log(`Added contract ${name} at ${address} for event monitoring`);
       return contract;
     }
     
     startListening() {
       // Listen for new blocks
       this.provider.on("block", this.handleNewBlock.bind(this));
       
       // Listen for specific events on each contract
       for (const [address, { contract, name }] of this.contracts.entries()) {
         contract.on("*", (...args) => {
           const event = args[args.length - 1];
           this.handleContractEvent(name, event);
         });
       }
     }
     
     async handleNewBlock(blockNumber) {
       console.log(`New block: ${blockNumber} on network ${this.networkId}`);
       
       // Check pending transactions
       const pendingTxs = await db.collection("Transaction").find({
         networkId: this.networkId,
         status: "pending"
       }).toArray();
       
       for (const tx of pendingTxs) {
         try {
           const receipt = await this.provider.getTransactionReceipt(tx.hash);
           if (receipt) {
             const status = receipt.status ? "confirmed" : "failed";
             
             await db.collection("Transaction").updateOne(
               { hash: tx.hash },
               {
                 $set: {
                   status,
                   blockNumber: receipt.blockNumber,
                   gasUsed: receipt.gasUsed.toString(),
                   effectiveGasPrice: receipt.effectiveGasPrice.toString()
                 }
               }
             );
             
             console.log(`Transaction ${tx.hash} ${status} in block ${receipt.blockNumber}`);
           }
         } catch (error) {
           console.error(`Error checking transaction ${tx.hash}:`, error);
         }
       }
     }
     
     async handleContractEvent(contractName, event) {
       console.log(`Event from ${contractName}: ${event.event}`);
       
       // Store event in database
       await db.collection("ContractEvent").insertOne({
         contractName,
         eventName: event.event,
         transactionHash: event.transactionHash,
         blockNumber: event.blockNumber,
         args: JSON.parse(JSON.stringify(event.args)), // Convert BigNumber to string
         timestamp: new Date()
       });
       
       // Emit event for WebSocket clients if needed
     }
     
     stop() {
       this.provider.removeAllListeners();
       
       for (const [address, { contract }] of this.contracts.entries()) {
         contract.removeAllListeners();
       }
       
       console.log(`Stopped event listener for network ${this.networkId}`);
     }
   }
   ```

### Gas Price Management

1. **Gas price estimation**:

   ```javascript
   // src/lib/gas-price.js
   const { ethers } = require("ethers");
   const { getProvider } = require("./blockchain");
   
   async function estimateGasPrice(networkId, priority = "medium") {
     try {
       const provider = getProvider(networkId);
       
       // Get current gas price from provider
       const gasPrice = await provider.getGasPrice();
       
       // Apply multipliers based on priority
       const multipliers = {
         low: 0.8,
         medium: 1.0,
         high: 1.5,
         urgent: 2.0
       };
       
       const multiplier = multipliers[priority] || 1.0;
       return gasPrice.mul(Math.floor(multiplier * 100)).div(100);
     } catch (error) {
       console.error(`Error estimating gas price:`, error);
       // Fallback gas prices (in gwei)
       const fallbackPrices = {
         1: { low: 20, medium: 30, high: 50, urgent: 80 }, // Mainnet
         84532: { low: 1, medium: 1.5, high: 2, urgent: 3 } // BaseSepolia
       };
       
       const gweiPrice = fallbackPrices[networkId]?.[priority] || 10;
       return ethers.utils.parseUnits(gweiPrice.toString(), "gwei");
     }
   }
   
   async function getGasSettings(networkId, priority = "medium") {
     const gasPrice = await estimateGasPrice(networkId, priority);
     
     // For EIP-1559 compatible networks, use maxFeePerGas and maxPriorityFeePerGas
     const provider = getProvider(networkId);
     const network = await provider.getNetwork();
     
     // Check if network supports EIP-1559
     const block = await provider.getBlock("latest");
     const eip1559Support = block && block.baseFeePerGas !== undefined;
     
     if (eip1559Support) {
       const baseFeePerGas = block.baseFeePerGas;
       
       // Priority fee multipliers
       const priorityFeeMultipliers = {
         low: 1.0,
         medium: 1.5,
         high: 2.0,
         urgent: 3.0
       };
       
       const multiplier = priorityFeeMultipliers[priority] || 1.5;
       const maxPriorityFeePerGas = ethers.utils.parseUnits("1", "gwei").mul(multiplier);
       
       // maxFeePerGas = (baseFeePerGas * 2) + maxPriorityFeePerGas
       const maxFeePerGas = baseFeePerGas.mul(2).add(maxPriorityFeePerGas);
       
       return { maxFeePerGas, maxPriorityFeePerGas };
     }
     
     // Fallback to regular gasPrice for non-EIP-1559 networks
     return { gasPrice };
   }
   ```

2. **Transaction submission with gas management**:

   ```javascript
   // src/lib/transaction.js
   const { getProvider } = require("./blockchain");
   const { getGasSettings } = require("./gas-price");
   
   async function sendTransaction(networkId, txData, priority = "medium") {
     const provider = getProvider(networkId);
     const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
     
     // Get gas settings based on network and priority
     const gasSettings = await getGasSettings(networkId, priority);
     
     // Estimate gas limit
     const gasLimit = await provider.estimateGas({
       from: signer.address,
       to: txData.to,
       data: txData.data,
       value: txData.value || 0
     });
     
     // Add buffer to gas limit
     const gasLimitWithBuffer = gasLimit.mul(120).div(100); // 20% buffer
     
     // Construct transaction
     const transaction = {
       from: signer.address,
       to: txData.to,
       data: txData.data,
       value: txData.value || 0,
       gasLimit: gasLimitWithBuffer,
       ...gasSettings
     };
     
     // Send transaction
     const tx = await signer.sendTransaction(transaction);
     console.log(`Transaction sent: ${tx.hash}`);
     
     return tx;
   }
   ```

### Network Upgrades Handling

1. **Network upgrade detector**:

   ```javascript
   // src/lib/network-upgrade.js
   const { getProvider } = require("./blockchain");
   
   async function checkNetworkUpgrade(networkId) {
     try {
       const provider = getProvider(networkId);
       
       // Get current block
       const block = await provider.getBlock("latest");
       
       // Check for fork identifier or other upgrade indicators
       const isEIP1559 = block && block.baseFeePerGas !== undefined;
       
       // Check for upcoming network upgrades
       const upgrades = {
         1: [ // Ethereum Mainnet
           { name: "Cancun", block: 19000000, active: block.number >= 19000000 }
         ],
         84532: [ // BaseSepolia
           { name: "Future Upgrade", block: 5000000, active: block.number >= 5000000 }
         ]
       };
       
       const networkUpgrades = upgrades[networkId] || [];
       const upcomingUpgrades = networkUpgrades.filter(u => !u.active);
       const activeUpgrades = networkUpgrades.filter(u => u.active);
       
       return {
         currentBlock: block.number,
         features: {
           eip1559: isEIP1559
         },
         activeUpgrades,
         upcomingUpgrades
       };
     } catch (error) {
       console.error(`Error checking network upgrade:`, error);
       return { error: error.message };
     }
   }
   ```

2. **Handling network forks**:

   ```javascript
   // src/lib/fork-handler.js
   const { getProvider } = require("./blockchain");
   
   class ForkHandler {
     constructor(networkId) {
       this.networkId = networkId;
       this.provider = getProvider(networkId);
       this.forkBlocks = {
         1: { // Ethereum Mainnet
           "Cancun": 19000000
         },
         84532: { // BaseSepolia
           "FutureUpgrade": 5000000
         }
       };
     }
     
     async detectFork() {
       const currentBlock = await this.provider.getBlockNumber();
       const networkForks = this.forkBlocks[this.networkId] || {};
       
       const activeForks = [];
       const pendingForks = [];
       
       for (const [name, blockNumber] of Object.entries(networkForks)) {
         if (currentBlock >= blockNumber) {
           activeForks.push({ name, blockNumber });
         } else {
           pendingForks.push({
             name,
             blockNumber,
             blocksRemaining: blockNumber - currentBlock
           });
         }
       }
       
       return { currentBlock, activeForks, pendingForks };
     }
     
     async adjustForFork(txParams) {
       const forkStatus = await this.detectFork();
       const newParams = { ...txParams };
       
       // Adjust transaction parameters based on active forks
       for (const fork of forkStatus.activeForks) {
         if (fork.name === "Cancun" || fork.name === "London") {
           // EIP-1559 transaction type
           delete newParams.gasPrice;
           
           if (!newParams.maxFeePerGas) {
             const block = await this.provider.getBlock("latest");
             const baseFeePerGas = block.baseFeePerGas;
             
             newParams.maxPriorityFeePerGas = ethers.utils.parseUnits("1.5", "gwei");
             newParams.maxFeePerGas = baseFeePerGas.mul(2).add(newParams.maxPriorityFeePerGas);
           }
           
           // Set type 2 transaction (EIP-1559)
           newParams.type = 2;
         }
       }
       
       return newParams;
     }
   }
   ```

## Version Control

### Repository Structure

1. **Recommended repository structure**:

   ```
   gemforce/
   ├── .github/                  # GitHub workflows and templates
   ├── contracts/                # Smart contracts
   │   ├── facets/               # Diamond facets
   │   ├── interfaces/           # Contract interfaces
   │   ├── libraries/            # Contract libraries
   │   ├── upgradeInitializers/  # Initializers for upgrades
   ├── deploy/                   # Deployment scripts
   ├── deployments/              # Deployment artifacts
   ├── docs/                     # Documentation
   ├── scripts/                  # Utility scripts
   ├── src/                      # Source code
   │   ├── cloud-functions/      # Parse cloud functions
   │   ├── lib/                  # Shared libraries
   │   ├── indexer/              # Blockchain indexer
   │   ├── triggers/             # Parse triggers
   ├── test/                     # Tests
   │   ├── unit/                 # Unit tests
   │   ├── integration/          # Integration tests
   │   ├── fixtures/             # Test fixtures
   ├── .env.example              # Example environment variables
   ├── hardhat.config.ts         # Hardhat configuration
   ├── package.json              # Project metadata and dependencies
   ├── tsconfig.json             # TypeScript configuration
   └── README.md                 # Project documentation
   ```

2. **Repository organization best practices**:

   - Separate smart contracts from off-chain code
   - Organize contracts by functionality
   - Use descriptive folder names
   - Keep tests close to the code they test
   - Include documentation for each component

### Branching Strategy

1. **Git Flow branching model**:

   ```
   main          ●───────●─────●──────●───── (production releases)
                 │       │     │      │
                 │       │     │      │
   staging       │       ●─────●──────●───── (pre-production testing)
                 │       │            │
                 │       │            │
   develop       ●───────●────────────●───── (integration branch)
                 │       │      │     │
                 │       │      │     │
   feature/xyz   ●───────●      │     │      (feature branches)
                         │      │     │
   feature/abc           ●──────●     │      (feature branches)
                                      │
   hotfix/123                         ●───── (hotfix branches)
   ```

   - **main**: Production code, tagged with version numbers
   - **staging**: Pre-production testing
   - **develop**: Integration branch for feature development
   - **feature/***:  Feature branches for new development
   - **hotfix/***:  Urgent fixes for production issues

2. **Branching guidelines**:

   ```bash
   # Create a new feature branch from develop
   git checkout develop
   git pull
   git checkout -b feature/new-feature
   
   # Work on the feature, committing changes
   git add .
   git commit -m "Implement new feature"
   
   # Push feature branch to remote
   git push -u origin feature/new-feature
   
   # When feature is complete, merge to develop
   git checkout develop
   git pull
   git merge --no-ff feature/new-feature
   git push origin develop
   
   # Create a hotfix branch from main
   git checkout main
   git pull
   git checkout -b hotfix/critical-fix
   
   # Fix the issue, commit changes
   git add .
   git commit -m "Fix critical issue"
   
   # Push hotfix branch
   git push -u origin hotfix/critical-fix
   
   # When hotfix is complete, merge to main and develop
   git checkout main
   git pull
   git merge --no-ff hotfix/critical-fix
   git push origin main
   
   git checkout develop
   git pull
   git merge --no-ff hotfix/critical-fix
   git push origin develop
   ```

### Release Management

1. **Semantic versioning**:

   ```
   MAJOR.MINOR.PATCH
   ```

   - **MAJOR**: Breaking changes
   - **MINOR**: New features, backwards compatible
   - **PATCH**: Bug fixes, backwards compatible

2. **Release process**:

   ```bash
   # Prepare release from develop
   git checkout develop
   git pull
   git checkout -b release/1.2.0
   
   # Update version numbers
   npm version minor --no-git-tag-version
   
   # Make final adjustments
   git add .
   git commit -m "Prepare release 1.2.0"
   
   # Merge to staging for testing
   git checkout staging
   git pull
   git merge --no-ff release/1.2.0
   git push origin staging
   
   # Deploy to staging environment
   npm run deploy:staging
   
   # After testing, merge to main
   git checkout main
   git pull
   git merge --no-ff release/1.2.0
   git tag -a v1.2.0 -m "Release 1.2.0"
   git push origin main --tags
   
   # Update develop with any changes
   git checkout develop
   git pull
   git merge --no-ff release/1.2.0
   git push origin develop
   
   # Delete release branch
   git branch -d release/1.2.0
   ```

3. **Changelog management**:

   ```markdown
   # Changelog
   
   All notable changes to this project will be documented in this file.
   
   ## [1.2.0] - 2025-02-25
   
   ### Added
   - New carbon credit retirement feature
   - Support for multiple wallet providers
   
   ### Changed
   - Improved marketplace performance
   - Updated identity verification flow
   
   ### Fixed
   - Fixed issue with token minting
   - Resolved WebSocket connection stability
   
   ## [1.1.0] - 2025-01-15
   
   ### Added
   - Identity management system
   - DFNS wallet integration
   
   ### Changed
   - Upgraded Parse Server to latest version
   - Improved error handling
   
   ### Deprecated
   - Legacy API endpoints (to be removed in 2.0)
   ```

### Tagging Conventions

1. **Version tags**:

   ```bash
   # Create an annotated version tag
   git tag -a v1.2.0 -m "Release 1.2.0"
   
   # Push tags to remote
   git push origin --tags
   ```

2. **Environment tags**:

   ```bash
   # Create a tag for production deployment
   git tag -a prod-2025-02-25 -m "Production deployment on Feb 25, 2025"
   
   # Create a tag for staging deployment
   git tag -a staging-2025-02-20 -m "Staging deployment on Feb 20, 2025"
   ```

3. **Contract deployment tags**:

   ```bash
   # Create a tag for contract deployment
   git tag -a deploy-mainnet-1.2.0 -m "Mainnet contract deployment v1.2.0"
   
   # Create a tag for facet upgrade
   git tag -a upgrade-marketplace-1.2.1 -m "Marketplace facet upgrade to v1.2.1"
   ```

### Documentation Versioning

1. **Documentation directory structure**:

   ```
   docs/
   ├── latest/                  # Latest documentation (symlink to current version)
   ├── v1.2.0/                  # Documentation for v1.2.0
   │   ├── admin-guide.md       # Administrator guide
   │   ├── api-reference.md     # API reference
   │   └── ...
   ├── v1.1.0/                  # Documentation for v1.1.0
   │   ├── admin-guide.md       # Administrator guide
   │   ├── api-reference.md     # API reference
   │   └── ...
   └── ...
   ```

2. **Version-specific documentation**:

   ```bash
   # Create a new documentation version
   mkdir -p docs/v1.2.0
   cp -r docs/v1.1.0/* docs/v1.2.0/
   
   # Update documentation for new version
   # Edit files in docs/v1.2.0/
   
   # Update the latest symlink
   rm docs/latest
   ln -s v1.2.0 docs/latest
   ```

3. **Documentation in the codebase**:

   ```solidity
   // contracts/facets/MarketplaceFacet.sol
   
   /**
    * @title MarketplaceFacet
    * @dev Implements marketplace functionality for the Diamond
    * @notice This facet handles listing, buying, and selling tokens
    * @version 1.2.0
    */
   contract MarketplaceFacet is Modifiers {
       // ...
   ```

   ```javascript
   // src/cloud-functions/marketplace.js
   
   /**
    * @api {post} /parse/functions/listItem List an item for sale
    * @apiVersion 1.2.0
    * @apiName ListItem
    * @apiGroup Marketplace
    * @apiDescription Lists a token for sale in the marketplace
    *
    * @apiParam {String} tokenId The ID of the token to list
    * @apiParam {String} price The price in ETH
    *
    * @apiSuccess {Object} result The result of the operation
    * @apiSuccess {Boolean} result.success Whether the operation was successful
    * @apiSuccess {String} result.transactionHash The transaction hash
    */
   Parse.Cloud.define("listItem", async (request) => {
       // ...
   });