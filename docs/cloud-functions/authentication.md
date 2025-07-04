# Authentication Cloud Functions

## Overview

The [`authFunctions.ts`](auth-functions.md) module provides comprehensive user authentication and account management functionality for the Gemforce platform. This module handles user registration, email verification, password management, and administrative functions through Parse Server cloud functions.

## Module Details

- **File**: `src/cloud-functions/authFunctions.ts`
- **Framework**: Parse Server Cloud Functions
- **Language**: TypeScript
- **Dependencies**: crypto, ejs, jsonwebtoken, @sendgrid/mail

## Key Features

### 🔹 User Registration & Verification
- Secure user registration with email verification
- Duplicate email prevention
- Role-based access control setup
- Automated verification email sending

### 🔹 Password Management
- Password reset request and processing
- Secure token-based password reset
- Email notifications for password changes
- Admin UI password setup

### 🔹 User Account Management
- User profile updates with field validation
- Onboarding status tracking
- Wallet and identity integration
- Administrative user management

### 🔹 Email Integration
- SendGrid email service integration
- EJS template-based email rendering
- Multiple email types (verification, reset, success)
- Professional email templates

## Cloud Functions

### User Registration Functions

#### `registerUser`
```typescript
Parse.Cloud.define("registerUser", async (request) => {
  const { username, password, email, company, firstName, lastName } = request.params;
  // Implementation details...
});
```

**Purpose**: Registers a new user account with email verification.

**Parameters**:
- `username` (string): Unique username for the account
- `password` (string): User's password
- `email` (string): User's email address
- `company` (string, optional): User's company name
- `firstName` (string, optional): User's first name
- `lastName` (string, optional): User's last name

**Returns**: Success message with verification instructions

**Process**:
1. Validates required fields (username, password, email)
2. Checks for existing users with the same email
3. Creates new Parse User with provided information
4. Sets up role-based ACL (user + admin access)
5. Generates secure verification token (24-hour expiry)
6. Sends verification email via SendGrid
7. Returns success response

**Access Control**: 
- User gets read/write access to their own data
- Admin role gets read/write access to user data

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("registerUser", {
  username: "john.doe",
  password: "SecurePassword123!",
  email: "john.doe@example.com",
  company: "Acme Corp",
  firstName: "John",
  lastName: "Doe"
});

console.log(result.message); // "User registered successfully. Please verify your email."
```

**Error Conditions**:
- Missing required fields
- Email already exists
- Admin role not found
- Email sending failure

---

#### `verifyEmail`
```typescript
Parse.Cloud.define("verifyEmail", async (request) => {
  const { token } = request.params;
  // Implementation details...
});
```

**Purpose**: Verifies user email address using verification token.

**Parameters**:
- `token` (string): Email verification token

**Returns**: Success message confirming email verification

**Process**:
1. Validates token parameter
2. Finds user associated with the token
3. Checks token expiration (24-hour limit)
4. Sets emailVerified flag to true
5. Removes verification token and expiration
6. Saves user with verified status

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("verifyEmail", {
  token: "abc123def456..."
});

console.log(result.message); // "Email verified successfully."
```

**Error Conditions**:
- Missing token
- Invalid or expired token
- Token expiration exceeded

---

#### `retrieveEmailFromToken`
```typescript
Parse.Cloud.define("retrieveEmailFromToken", async (request) => {
  const { token } = request.params;
  // Implementation details...
});
```

**Purpose**: Retrieves email address associated with a verification token.

**Parameters**:
- `token` (string): Verification token

**Returns**: Email address associated with the token

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("retrieveEmailFromToken", {
  token: "abc123def456..."
});

console.log(result.email); // "john.doe@example.com"
```

### Password Management Functions

#### `requestPasswordReset`
```typescript
Parse.Cloud.define("requestPasswordReset", async (request) => {
  const { email } = request.params;
  // Implementation details...
});
```

**Purpose**: Initiates password reset process by sending reset email.

**Parameters**:
- `email` (string): Email address of the account to reset

**Returns**: Success message confirming reset email sent

**Process**:
1. Validates email parameter
2. Finds user by email address
3. Generates secure reset token (1-hour expiry)
4. Saves token and expiration to user account
5. Sends password reset email with reset link
6. Returns success confirmation

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("requestPasswordReset", {
  email: "john.doe@example.com"
});

console.log(result.message); // "Password reset email sent successfully."
```

**Error Conditions**:
- Missing email parameter
- No user found with email
- Email sending failure

---

#### `resetPassword`
```typescript
Parse.Cloud.define("resetPassword", async (request) => {
  const { token, newPassword, skipEmail = false } = request.params;
  // Implementation details...
});
```

**Purpose**: Completes password reset using reset token and new password.

**Parameters**:
- `token` (string): Password reset token
- `newPassword` (string): New password for the account
- `skipEmail` (boolean, optional): Skip sending success email

**Returns**: Success message confirming password update

**Process**:
1. Validates token and new password
2. Finds user with valid, non-expired token
3. Updates user password
4. Clears reset token and expiration
5. Optionally sends success confirmation email
6. Returns success response

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("resetPassword", {
  token: "reset123token456...",
  newPassword: "NewSecurePassword123!",
  skipEmail: false
});

console.log(result.message); // "Password updated successfully. A confirmation email has been sent."
```

**Error Conditions**:
- Missing token or new password
- Invalid or expired token
- Password update failure

### User Management Functions

#### `updateUserByEmail`
```typescript
Parse.Cloud.define("updateUserByEmail", async (request) => {
  const { email, updates, secretKey } = request.params;
  // Implementation details...
});
```

**Purpose**: Updates user account information using email and secret key authentication.

**Parameters**:
- `email` (string): Email address of user to update
- `updates` (object): Object containing fields to update
- `secretKey` (string): Pre-shared secret key for authorization

**Allowed Update Fields**:
- `termsAccepted` (Date): Terms acceptance timestamp
- `email` (string): Email address
- `walletAddress` (string): Blockchain wallet address
- `walletId` (string): Wallet identifier
- `walletPreference` (string): Preferred wallet type
- `personaReferenceId` (string): Persona verification ID
- `username` (string): Username

**Returns**: Success message with updated user data

**Security**: Requires valid secret key matching `AUTH_SECRET_KEY` environment variable

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("updateUserByEmail", {
  email: "john.doe@example.com",
  updates: {
    walletAddress: "0x1234567890123456789012345678901234567890",
    termsAccepted: new Date().toISOString()
  },
  secretKey: process.env.AUTH_SECRET_KEY
});

console.log(result.message); // "User updated successfully."
```

**Error Conditions**:
- Unauthorized secret key
- Missing email or updates
- User not found
- Invalid field updates
- Invalid date format for termsAccepted

---

#### `isUserOnboarded`
```typescript
Parse.Cloud.define("isUserOnboarded", async (request) => {
  const { email } = request.params;
  // Implementation details...
});
```

**Purpose**: Checks if user has completed onboarding process.

**Parameters**:
- `email` (string): Email address to check

**Returns**: Boolean indicating onboarding completion status

**Onboarding Criteria**: User is considered onboarded if they have both:
- `walletAddress` set (not null/undefined)
- `personaReferenceId` set (not null/undefined)

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("isUserOnboarded", {
  email: "john.doe@example.com"
});

console.log(result.result); // true if onboarded, false otherwise
```

---

#### `getUsersWithIdentityWallets`
```typescript
Parse.Cloud.define("getUsersWithIdentityWallets", async (request) => {
  // Implementation details...
});
```

**Purpose**: Retrieves users who have wallet addresses matching Identity table records.

**Returns**: Array of user objects with wallet addresses and emails

**Process**:
1. Queries Identity table for all wallet addresses
2. Queries User table for users with matching wallet addresses
3. Returns filtered list with wallet address and email

**Example Usage**:
```javascript
const users = await Parse.Cloud.run("getUsersWithIdentityWallets");

users.forEach(user => {
  console.log(`User: ${user.email}, Wallet: ${user.walletAddress}`);
});
```

### Administrative Functions

#### `setupAdminUIPassword`
```typescript
Parse.Cloud.define("setupAdminUIPassword", async (request) => {
  const { email } = request.params;
  // Implementation details...
});
```

**Purpose**: Initiates admin UI password setup for existing users.

**Parameters**:
- `email` (string): Email address of admin user

**Returns**: Success message confirming setup email sent

**Process**:
1. Finds user by email address
2. Generates secure setup token (1-hour expiry)
3. Saves token to user account
4. Sends admin UI password setup email
5. Returns success confirmation

**Example Usage**:
```javascript
const result = await Parse.Cloud.run("setupAdminUIPassword", {
  email: "admin@example.com"
});

console.log(result.message); // "Setup Admin UI password email sent successfully."
```

---

#### `generateOnboardingLink`
```typescript
Parse.Cloud.define("generateOnboardingLink", async (request) => {
  const { jwtToken } = request.params;
  // Implementation details...
});
```

**Purpose**: Creates user account and generates onboarding link from JWT token.

**Parameters**:
- `jwtToken` (string): JWT token containing user information

**JWT Payload Structure**:
```typescript
{
  email: string;
  firstName: string;
  lastName: string;
}
```

**Returns**: Onboarding link for email verification

**Process**:
1. Verifies JWT token with secret key
2. Extracts user information from token
3. Checks for existing user with same email
4. Creates new user account with default password
5. Sets up ACL permissions
6. Generates verification token
7. Returns onboarding link

**Example Usage**:
```javascript
const payload = {
  email: "newuser@example.com",
  firstName: "New",
  lastName: "User"
};
const jwtToken = jwt.sign(payload, secretKey, { expiresIn: '1h' });

const result = await Parse.Cloud.run("generateOnboardingLink", {
  jwtToken: jwtToken
});

console.log(result.result); // "https://app.example.com/verify-email/abc123..."
```

## Email Templates and Integration

### SendGrid Configuration
The module uses SendGrid for email delivery with the following configuration:
- **Sender**: `NomyxID.Registrations@nomyx.io`
- **Templates**: EJS-based HTML templates
- **Template Location**: `src/utils/emailTemplate/`

### Email Types

#### Verification Email
- **Template**: `verifyEmailTemplate.ejs`
- **Purpose**: Email address verification
- **Link**: `${PROJECT_WIZARD_URL}/verify-email/${token}`

#### Password Reset Email
- **Template**: `resetPasswordTemplate.ejs`
- **Purpose**: Password reset request
- **Link**: `${PROJECT_WIZARD_URL}/reset-password/${token}`

#### Password Reset Success Email
- **Template**: `resetPasswordSuccessTemplate.ejs`
- **Purpose**: Password reset confirmation
- **Content**: Success notification

#### Admin UI Password Setup Email
- **Template**: `createPasswordTemplate.ejs`
- **Purpose**: Admin UI password setup
- **Link**: `${ADMIN_UI_URL}/create-password/${token}`

## Integration Examples

### User Registration Flow
```javascript
// Complete user registration and verification flow
class UserRegistrationService {
  async registerNewUser(userData) {
    try {
      // Step 1: Register user
      const registrationResult = await Parse.Cloud.run("registerUser", {
        username: userData.username,
        password: userData.password,
        email: userData.email,
        company: userData.company,
        firstName: userData.firstName,
        lastName: userData.lastName
      });
      
      console.log("Registration successful:", registrationResult.message);
      
      // Step 2: User receives email and clicks verification link
      // This would be handled by frontend calling verifyEmail
      
      return {
        success: true,
        message: "Registration initiated. Please check email for verification."
      };
      
    } catch (error) {
      console.error("Registration failed:", error.message);
      throw error;
    }
  }
  
  async verifyUserEmail(token) {
    try {
      const verificationResult = await Parse.Cloud.run("verifyEmail", { token });
      console.log("Email verification successful:", verificationResult.message);
      return verificationResult;
    } catch (error) {
      console.error("Email verification failed:", error.message);
      throw error;
    }
  }
}
```

### Password Reset Flow
```javascript
// Complete password reset flow
class PasswordResetService {
  async requestReset(email) {
    try {
      const result = await Parse.Cloud.run("requestPasswordReset", { email });
      console.log("Password reset request:", result.message);
      return result;
    } catch (error) {
      console.error("Password reset request failed:", error.message);
      throw error;
    }
  }

  async resetPassword(token, newPassword) {
    try {
      const result = await Parse.Cloud.run("resetPassword", { token, newPassword });
      console.log("Password reset:", result.message);
      return result;
    } catch (error) {
      console.error("Password reset failed:", error.message);
      throw error;
    }
  }
}
```

### User Onboarding Integration
```javascript
// Example of integrating user onboarding status check
class OnboardingService {
  async checkOnboardingStatus(email) {
    try {
      const result = await Parse.Cloud.run("isUserOnboarded", { email });
      console.log(`User ${email} onboarding status: ${result.result}`);
      return result.result;
    } catch (error) {
      console.error("Failed to check onboarding status:", error.message);
      throw error;
    }
  }
}
```

### Admin User Management
```javascript
// Example of administrative user management
class AdminUserService {
  async setupAdminPassword(email) {
    try {
      const result = await Parse.Cloud.run("setupAdminUIPassword", { email });
      console.log("Admin password setup initiated:", result.message);
      return result;
    } catch (error) {
      console.error("Admin password setup failed:", error.message);
      throw error;
    }
  }

  async generateOnboardingLink(jwtToken) {
    try {
      const result = await Parse.Cloud.run("generateOnboardingLink", { jwtToken });
      console.log("Onboarding link generated:", result.result);
      return result;
    } catch (error) {
      console.error("Failed to generate onboarding link:", error.message);
      throw error;
    }
  }
}
```

## Security Considerations

### Token Security
- JWT tokens are signed with a strong secret key.
- Tokens have a short expiration time (e.g., 1 hour).
- Refresh tokens can be implemented for longer sessions.

### Access Control
- ACLs (Access Control Lists) are used to restrict data access.
- Master Key is required for privileged operations.
- Role-based access ensures proper permissions.

### Data Protection
- Sensitive user data (e.g., passwords) is hashed.
- Email addresses are stored in lowercase for consistency.
- Input validation prevents injection attacks.

### Email Security
- SendGrid API key is securely stored.
- Email templates are sanitized to prevent XSS.
- SPF/DKIM records are configured for email authenticity.

## Error Handling

### Validation Errors
- Missing or invalid parameters.
- Incorrect data types.
- Constraints violations (e.g., unique email).

### User Errors
- Invalid credentials.
- Account not found.
- Email not verified.
- Too many login attempts.

### System Errors
- Database connection issues.
- External API failures (e.g., SendGrid).
- Unexpected server errors.

## Testing

### Unit Tests
- Individual cloud functions are tested in isolation.
- Mock Parse SDK and external dependencies.
- Cover all success and error paths.

### Integration Tests
- Test end-to-end flows (e.g., registration to login).
- Verify interactions with Parse Server and external services.
- Use a dedicated test environment.

### Security Tests
- Penetration testing for common vulnerabilities.
- Brute force attack simulations.
- Session hijacking attempts.

## Related Documentation

- [Cloud Functions: Auth Functions](auth-functions.md)
- [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)
- [Integrator's Guide: Authentication](../integrator-guide/authentication.md)
- [Gemforce Administrator Guide](../gemforce-administrator-guide.md)
- [Security: Overview](../security/overview.md)
- [Parse Server Integration](../integrator-guide/parse-server.md)