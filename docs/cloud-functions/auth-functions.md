# Authentication Functions

The Authentication Functions module provides comprehensive user authentication, registration, and session management capabilities for the Gemforce platform. It handles user lifecycle management, password security, and integration with external authentication providers.

## Overview

The Authentication Functions provide:

- **User Registration**: Create new user accounts with validation
- **Login/Logout**: Secure authentication and session management
- **Password Management**: Password reset and change functionality
- **Session Management**: JWT token generation and validation
- **Multi-Factor Authentication**: Optional MFA support
- **Social Authentication**: Integration with external providers

## Key Features

### User Management
- **Registration**: Email-based user registration with verification
- **Profile Management**: User profile creation and updates
- **Account Verification**: Email verification workflows
- **Account Recovery**: Password reset and account recovery

### Authentication Security
- **Password Hashing**: Secure password storage using bcrypt
- **JWT Tokens**: Stateless authentication with JSON Web Tokens
- **Session Management**: Secure session handling
- **Rate Limiting**: Protection against brute force attacks

### Integration Support
- **DFNS Integration**: Wallet creation during registration
- **Parse Server**: User management through Parse Server
- **External Providers**: Support for OAuth providers
- **Webhook Integration**: Authentication event notifications

## Core Functions

### registerUser()
Creates a new user account with email verification.

**Parameters:**
```typescript
interface RegisterUserRequest {
  email: string;
  password: string;
  username?: string;
  firstName?: string;
  lastName?: string;
  acceptTerms: boolean;
}
```

**Returns:**
```typescript
interface RegisterUserResponse {
  success: boolean;
  userId?: string;
  message: string;
  verificationRequired?: boolean;
}
```

**Usage:**
```typescript
const result = await Parse.Cloud.run('registerUser', {
  email: 'user@example.com',
  password: 'securePassword123',
  username: 'johndoe',
  firstName: 'John',
  lastName: 'Doe',
  acceptTerms: true
});
```

### loginUser()
Authenticates a user and returns session information.

**Parameters:**
```typescript
interface LoginUserRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}
```

**Returns:**
```typescript
interface LoginUserResponse {
  success: boolean;
  sessionToken?: string;
  user?: {
    id: string;
    email: string;
    username: string;
    walletAddress?: string;
  };
  message: string;
}
```

**Usage:**
```typescript
const result = await Parse.Cloud.run('loginUser', {
  email: 'user@example.com',
  password: 'securePassword123',
  rememberMe: true
});
```

### resetPassword()
Initiates password reset process via email.

**Parameters:**
```typescript
interface ResetPasswordRequest {
  email: string;
}
```

**Returns:**
```typescript
interface ResetPasswordResponse {
  success: boolean;
  message: string;
}
```

### changePassword()
Changes user password with current password verification.

**Parameters:**
```typescript
interface ChangePasswordRequest {
  currentPassword: string;
  newPassword: string;
}
```

**Returns:**
```typescript
interface ChangePasswordResponse {
  success: boolean;
  message: string;
}
```

### verifyEmail()
Verifies user email address using verification token.

**Parameters:**
```typescript
interface VerifyEmailRequest {
  token: string;
  email: string;
}
```

**Returns:**
```typescript
interface VerifyEmailResponse {
  success: boolean;
  message: string;
}
```

## Implementation Example

### User Registration with DFNS Integration

```typescript
Parse.Cloud.define('registerUser', async (request) => {
  const { email, password, username, firstName, lastName, acceptTerms } = request.params;
  
  try {
    // Validate input
    if (!email || !password || !acceptTerms) {
      throw new Error('Missing required fields');
    }
    
    // Check if user already exists
    const existingUser = await new Parse.Query(Parse.User)
      .equalTo('email', email.toLowerCase())
      .first({ useMasterKey: true });
    
    if (existingUser) {
      throw new Error('User already exists');
    }
    
    // Create new user
    const user = new Parse.User();
    user.set('email', email.toLowerCase());
    user.set('username', username || email.toLowerCase());
    user.set('password', password);
    user.set('firstName', firstName);
    user.set('lastName', lastName);
    user.set('emailVerified', false);
    user.set('acceptedTerms', acceptTerms);
    user.set('acceptedTermsDate', new Date());
    
    // Save user
    await user.signUp();
    
    // Create DFNS wallet
    try {
      const walletResult = await Parse.Cloud.run('createUserWallet', {
        userId: user.id,
        email: email.toLowerCase()
      });
      
      if (walletResult.success) {
        user.set('walletAddress', walletResult.walletAddress);
        user.set('dfnsWalletId', walletResult.walletId);
        await user.save(null, { useMasterKey: true });
      }
    } catch (walletError) {
      console.error('Wallet creation failed:', walletError);
      // Continue with registration even if wallet creation fails
    }
    
    // Send verification email
    await sendVerificationEmail(user);
    
    return {
      success: true,
      userId: user.id,
      message: 'Registration successful. Please check your email for verification.',
      verificationRequired: true
    };
    
  } catch (error) {
    console.error('Registration error:', error);
    return {
      success: false,
      message: error.message || 'Registration failed'
    };
  }
});
```

### Secure Login with Rate Limiting

```typescript
Parse.Cloud.define('loginUser', async (request) => {
  const { email, password, rememberMe } = request.params;
  
  try {
    // Rate limiting check
    const rateLimitKey = `login_attempts_${email.toLowerCase()}`;
    const attempts = await getFromCache(rateLimitKey) || 0;
    
    if (attempts >= 5) {
      throw new Error('Too many login attempts. Please try again later.');
    }
    
    // Attempt login
    const user = await Parse.User.logIn(email.toLowerCase(), password);
    
    // Clear rate limiting on successful login
    await clearFromCache(rateLimitKey);
    
    // Check if email is verified
    if (!user.get('emailVerified')) {
      await Parse.User.logOut();
      throw new Error('Please verify your email before logging in');
    }
    
    // Update last login
    user.set('lastLogin', new Date());
    user.set('loginCount', (user.get('loginCount') || 0) + 1);
    await user.save(null, { useMasterKey: true });
    
    // Generate session token
    const sessionToken = user.getSessionToken();
    
    return {
      success: true,
      sessionToken,
      user: {
        id: user.id,
        email: user.get('email'),
        username: user.get('username'),
        walletAddress: user.get('walletAddress')
      },
      message: 'Login successful'
    };
    
  } catch (error) {
    // Increment rate limiting counter
    const rateLimitKey = `login_attempts_${email.toLowerCase()}`;
    const attempts = await getFromCache(rateLimitKey) || 0;
    await setInCache(rateLimitKey, attempts + 1, 900); // 15 minutes
    
    console.error('Login error:', error);
    return {
      success: false,
      message: error.message || 'Login failed'
    };
  }
});
```

### Password Reset with Email

```typescript
Parse.Cloud.define('resetPassword', async (request) => {
  const { email } = request.params;
  
  try {
    // Find user
    const user = await new Parse.Query(Parse.User)
      .equalTo('email', email.toLowerCase())
      .first({ useMasterKey: true });
    
    if (!user) {
      // Don't reveal if user exists or not
      return {
        success: true,
        message: 'If an account with that email exists, a reset link has been sent.'
      };
    }
    
    // Generate reset token
    const resetToken = generateSecureToken();
    const resetExpiry = new Date(Date.now() + 3600000); // 1 hour
    
    user.set('passwordResetToken', resetToken);
    user.set('passwordResetExpiry', resetExpiry);
    await user.save(null, { useMasterKey: true });
    
    // Send reset email
    await sendPasswordResetEmail(user, resetToken);
    
    return {
      success: true,
      message: 'If an account with that email exists, a reset link has been sent.'
    };
    
  } catch (error) {
    console.error('Password reset error:', error);
    return {
      success: false,
      message: 'Password reset failed'
    };
  }
});
```

## Security Features

### Password Security

```typescript
// Password validation
function validatePassword(password: string): { valid: boolean; message?: string } {
  if (password.length < 8) {
    return { valid: false, message: 'Password must be at least 8 characters long' };
  }
  
  if (!/(?=.*[a-z])/.test(password)) {
    return { valid: false, message: 'Password must contain at least one lowercase letter' };
  }
  
  if (!/(?=.*[A-Z])/.test(password)) {
    return { valid: false, message: 'Password must contain at least one uppercase letter' };
  }
  
  if (!/(?=.*\d)/.test(password)) {
    return { valid: false, message: 'Password must contain at least one number' };
  }
  
  if (!/(?=.*[@$!%*?&])/.test(password)) {
    return { valid: false, message: 'Password must contain at least one special character' };
  }
  
  return { valid: true };
}
```

### Session Management

```typescript
Parse.Cloud.define('validateSession', async (request) => {
  try {
    const user = request.user;
    
    if (!user) {
      throw new Error('Invalid session');
    }
    
    // Check if session is still valid
    const sessionQuery = new Parse.Query(Parse.Session);
    sessionQuery.equalTo('user', user);
    sessionQuery.equalTo('sessionToken', request.headers['x-parse-session-token']);
    
    const session = await sessionQuery.first({ useMasterKey: true });
    
    if (!session) {
      throw new Error('Session not found');
    }
    
    // Check session expiry
    const expiresAt = session.get('expiresAt');
    if (expiresAt && new Date() > expiresAt) {
      await session.destroy({ useMasterKey: true });
      throw new Error('Session expired');
    }
    
    return {
      success: true,
      user: {
        id: user.id,
        email: user.get('email'),
        username: user.get('username')
      }
    };
    
  } catch (error) {
    return {
      success: false,
      message: error.message
    };
  }
});
```

### Multi-Factor Authentication

```typescript
Parse.Cloud.define('enableMFA', async (request) => {
  const user = request.user;
  
  if (!user) {
    throw new Error('Authentication required');
  }
  
  try {
    // Generate TOTP secret
    const secret = generateTOTPSecret();
    
    // Store secret (encrypted)
    user.set('mfaSecret', encryptSecret(secret));
    user.set('mfaEnabled', false); // Will be enabled after verification
    await user.save(null, { useMasterKey: true });
    
    // Generate QR code for authenticator app
    const qrCode = generateQRCode(user.get('email'), secret);
    
    return {
      success: true,
      secret,
      qrCode,
      message: 'Scan the QR code with your authenticator app'
    };
    
  } catch (error) {
    console.error('MFA setup error:', error);
    throw new Error('Failed to setup MFA');
  }
});
```

## Integration Patterns

### Frontend Integration

```typescript
class AuthService {
  async register(userData: RegisterUserRequest): Promise<RegisterUserResponse> {
    return await Parse.Cloud.run('registerUser', userData);
  }
  
  async login(credentials: LoginUserRequest): Promise<LoginUserResponse> {
    return await Parse.Cloud.run('loginUser', credentials);
  }
  
  async logout(): Promise<void> {
    await Parse.User.logOut();
  }
  
  async resetPassword(email: string): Promise<ResetPasswordResponse> {
    return await Parse.Cloud.run('resetPassword', { email });
  }
  
  async changePassword(passwords: ChangePasswordRequest): Promise<ChangePasswordResponse> {
    return await Parse.Cloud.run('changePassword', passwords);
  }
  
  getCurrentUser(): Parse.User | null {
    return Parse.User.current();
  }
  
  isAuthenticated(): boolean {
    return Parse.User.current() !== null;
  }
}
```

### React Hook Integration

```typescript
import React, { useState, useEffect, useContext, createContext } from 'react';
import Parse from 'parse';

interface AuthContextType {
  currentUser: Parse.User | null;
  isAuthenticated: boolean;
  login: (credentials: LoginUserRequest) => Promise<LoginUserResponse>;
  register: (userData: RegisterUserRequest) => Promise<RegisterUserResponse>;
  logout: () => Promise<void>;
  resetPassword: (email: string) => Promise<ResetPasswordResponse>;
  changePassword: (passwords: ChangePasswordRequest) => Promise<ChangePasswordResponse>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [currentUser, setCurrentUser] = useState<Parse.User | null>(Parse.User.current());
  const isAuthenticated = !!currentUser;

  useEffect(() => {
    const handleAuthChange = () => {
      setCurrentUser(Parse.User.current());
    };

    // Subscribe to Parse user changes (e.g., login, logout)
    Parse.User.on('logIn', handleAuthChange);
    Parse.User.on('logOut', handleAuthChange);

    return () => {
      // Clean up subscriptions
      Parse.User.off('logIn', handleAuthChange);
      Parse.User.off('logOut', handleAuthChange);
    };
  }, []);

  const login = async (credentials: LoginUserRequest) => {
    const response = await Parse.Cloud.run('loginUser', credentials);
    if (response.success) {
      setCurrentUser(Parse.User.current());
    }
    return response;
  };

  const register = async (userData: RegisterUserRequest) => {
    const response = await Parse.Cloud.run('registerUser', userData);
    if (response.success) {
      setCurrentUser(Parse.User.current());
    }
    return response;
  };

  const logout = async () => {
    await Parse.User.logOut();
    setCurrentUser(null);
  };

  const resetPassword = async (email: string) => {
    return await Parse.Cloud.run('resetPassword', { email });
  };

  const changePassword = async (passwords: ChangePasswordRequest) => {
    return await Parse.Cloud.run('changePassword', passwords);
  };

  const value = {
    currentUser,
    isAuthenticated,
    login,
    register,
    logout,
    resetPassword,
    changePassword,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
```

## Error Handling

### Common Errors
- `"User already exists"`: Email already registered
- `"Invalid credentials"`: Wrong email/password combination
- `"Email not verified"`: Account requires email verification
- `"Too many attempts"`: Rate limiting triggered
- `"Session expired"`: Authentication session has expired

### Error Response Format
```typescript
interface ErrorResponse {
  success: false;
  message: string;
  code?: string;
  details?: any;
}
```

## Best Practices

### Security Guidelines
1. **Password Strength**: Enforce strong password requirements
2. **Rate Limiting**: Implement login attempt rate limiting
3. **Session Management**: Use secure session handling
4. **Email Verification**: Require email verification for new accounts

### Development Guidelines
1. **Input Validation**: Validate all input parameters
2. **Error Handling**: Provide meaningful error messages
3. **Logging**: Log authentication events for security monitoring
4. **Testing**: Comprehensive testing of authentication flows

## Related Documentation

- [Integrator's Guide: Error Handling](../integrator-guide/error-handling.md)
- [Integrator's Guide: Authentication](../integrator-guide/authentication.md)
- [DFNS Integration](../integrator-guide/dfns.md)
- [Gemforce Administrator Guide](../gemforce-administrator-guide.md)
- [Security: Overview](../security/overview.md)
- [Parse Server Integration](../integrator-guide/parse-server.md)

## Security Considerations

- **Password Storage**: Passwords are hashed using bcrypt
- **Session Security**: JWT tokens with expiration
- **Rate Limiting**: Protection against brute force attacks
- **Email Verification**: Required for account activation
- **MFA Support**: Optional multi-factor authentication
- **Audit Logging**: All authentication events are logged