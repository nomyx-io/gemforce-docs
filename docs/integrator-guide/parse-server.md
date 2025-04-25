# Parse Server Integration Guide

## Connecting to Parse Server

### Server Configuration (src/lib/parse-server.ts)
```javascript
const config = {
  server: {
    appId: 'YOUR_APP_ID',
    masterKey: 'YOUR_MASTER_KEY',
    serverURL: 'http://localhost:1337/parse',
    port: 1337
  }
};

const parseServer = await parseServerLoad(config);
```

### Client Initialization (src/lib/parse.ts)
```javascript
import { parseClientConnect } from '../lib/parse';

const gemforceConfig = {
  server: {
    appId: 'YOUR_APP_ID',
    javascriptKey: 'YOUR_JS_KEY',
    masterKey: 'YOUR_MASTER_KEY',
    serverURL: 'http://localhost:1337/parse'
  }
};

await parseClientConnect(gemforceConfig);
```

## CRUD Operations

### Create Record
```javascript
import { createRecord } from '../lib/parse';

const newUser = await createRecord('User', ['email'], ['user@example.com'], {
  username: 'jdoe',
  email: 'user@example.com',
  password: 'securepassword'
});
```

### Query Records
```javascript
import { getRecords } from '../lib/parse';

// Get all users created after 2024
const users = await getRecords(
  'User',
  ['createdAt'],
  [{ $gt: new Date('2024-01-01') }],
  [],
  100,
  0,
  'createdAt',
  'desc'
);
```

### Update Record
```javascript
import { updateExistingRecord } from '../lib/parse';

await updateExistingRecord(
  'User',
  ['objectId'],
  ['xK9sL3mZ'],
  { lastLogin: new Date() }
);
```

## LiveQuery Setup for User Insertions

### Server Configuration
Enable LiveQuery in parse-server.ts:
```javascript
config.server.liveQuery = {
  classNames: ['User'] // Enable for User class
};
```

### Client Subscription
```javascript
const query = new Parse.Query('User');
const subscription = await query.subscribe();

subscription.on('create', (user) => {
  console.log('New user created:', user.get('email'));
  
  // Init identity workflow
  Parse.Cloud.run('dfnsCreateIdentityInit', {
    userId: user.id,
    walletAddress: user.get('walletAddress')
  });
});
```

## Identity Management Integration

### User Identity Workflow
1. User registration triggers identity creation:
```javascript
Parse.Cloud.afterSave('User', async (request) => {
  const user = request.object;
  if (!user.existed()) {
    await Parse.Cloud.run('dfnsCreateIdentityComplete', {
      userId: user.id,
      publicKey: user.get('publicKey')
    });
  }
});
```

2. Adding claims to identity:
```javascript
await Parse.Cloud.run('dfnsSetClaimsComplete', {
  userId: 'xK9sL3mZ',
  claims: [{
    topic: 'KYC_VERIFICATION',
    value: 'VERIFIED',
    issuer: 'TRUSTED_KYC_PROVIDER'
  }]
});
```

## Best Practices

1. Always use master key for backend operations:
```javascript
Parse.Cloud.useMasterKey();
```

2. Handle errors in LiveQuery subscriptions:
```javascript
subscription.on('error', (error) => {
  console.error('LiveQuery Error:', error);
});
```

3. Use include for related objects:
```javascript
const query = new Parse.Query('Identity');
query.include('user');