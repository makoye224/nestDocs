# User Management System - Lower Level Design

## Overview
The User Management System handles authentication, authorization, and user profile management for Students, Landlords, and Admins in the Nest Tanzania Rental Platform.

## Architecture Components

### AWS Services
- **AWS Cognito User Pool**: Authentication and user management
- **AWS AppSync**: GraphQL API gateway
- **AWS Lambda**: Business logic processing
- **DynamoDB**: Data persistence with event sourcing
- **S3 + CloudFront**: Profile image storage

### Lambda Functions
- **AuthFunction**: `nest/src/handlers/graphql-auth.ts`
- **Handler**: `rental-auth-{stage}`
- **Timeout**: 30 seconds
- **Memory**: 128MB (default)

### Database Tables
- **users** (Materialized View): Current user state
- **user-events** (Event Store): Immutable user event history
- **application-events**: Landlord verification applications

## Data Models

### User Profile Types
```typescript
enum UserType {
  TENANT = 'TENANT'
  LANDLORD = 'LANDLORD'
  ADMIN = 'ADMIN'
}

enum AccountStatus {
  PENDING_VERIFICATION = 'PENDING_VERIFICATION'
  ACTIVE = 'ACTIVE'
  SUSPENDED = 'SUSPENDED'
  PENDING_LANDLORD_VERIFICATION = 'PENDING_LANDLORD_VERIFICATION'
}
```

### Base User Profile
```typescript
interface BaseUser {
  userId: ID
  email: string
  phoneNumber: string
  firstName: string
  lastName: string
  userType: UserType
  accountStatus: AccountStatus
  isEmailVerified: boolean
  profileImage?: string
  language: string
  currency: string
  emailNotifications: boolean
  smsNotifications: boolean
  pushNotifications: boolean
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}
```

### Student (Tenant) Profile
```typescript
interface Tenant extends BaseUser {
  userType: 'TENANT'
  preferences?: {
    maxBudget?: number
    preferredLocations?: string[]
    propertyTypes?: PropertyType[]
    amenities?: string[]
  }
  savedProperties?: string[]
  applicationHistory?: string[]
}
```

### Landlord Profile
```typescript
interface Landlord extends BaseUser {
  userType: 'LANDLORD'
  businessName?: string
  businessLicense?: string
  taxId?: string
  verificationDocuments?: string[]
  nationalId?: string
  birthDate?: string
  alternatePhone?: string
  address?: Address
  propertiesOwned?: string[]
  verificationStatus?: 'PENDING' | 'APPROVED' | 'REJECTED'
}
```

### Admin Profile
```typescript
interface Admin extends BaseUser {
  userType: 'ADMIN'
  permissions: string[]
  adminLevel?: 'SUPER_ADMIN' | 'MODERATOR' | 'SUPPORT'
}
```

## API Operations

### Authentication Operations

#### Student Registration
```graphql
mutation SignUp($input: SignUpInput!) {
  signUp(input: $input) {
    accessToken
    refreshToken
    user {
      userId
      firstName
      lastName
      email
      userType
      accountStatus
    }
  }
}
```

**Input:**
```typescript
interface SignUpInput {
  email: string
  password: string
  phoneNumber: string
  firstName: string
  lastName: string
}
```

**Implementation Flow:**
1. Validate input data
2. Create Cognito user account
3. Generate user event: `USER_CREATED`
4. Materialize user profile in users table
5. Send email verification
6. Return JWT tokens

#### Email Verification
```graphql
mutation VerifyEmail($email: String!, $code: String!) {
  verifyEmail(email: $email, code: $code) {
    success
    message
  }
}
```

**Implementation Flow:**
1. Verify code with Cognito
2. Generate user event: `EMAIL_VERIFIED`
3. Update user status to ACTIVE
4. Return success response

#### Sign In
```graphql
mutation SignIn($email: String!, $password: String!) {
  signIn(email: $email, password: $password) {
    accessToken
    refreshToken
    user {
      userId
      firstName
      lastName
      email
      userType
      accountStatus
    }
  }
}
```

**Implementation Flow:**
1. Authenticate with Cognito
2. Retrieve user profile from users table
3. Update last login timestamp
4. Return JWT tokens and user data

### Profile Management Operations

#### Update User Profile
```graphql
mutation UpdateUser($userId: ID!, $input: UpdateUserInput!) {
  updateUser(userId: $userId, input: $input) {
    ... on Tenant {
      userId
      firstName
      lastName
      preferences
    }
    ... on Landlord {
      userId
      firstName
      lastName
      businessName
    }
  }
}
```

**Input:**
```typescript
interface UpdateUserInput {
  firstName?: string
  lastName?: string
  phoneNumber?: string
  profileImage?: string
  preferences?: AWSJSON
}
```

#### Get User Profile
```graphql
query GetUser($userId: ID!) {
  getUser(userId: $userId) {
    ... on Tenant {
      userId
      firstName
      lastName
      email
      preferences
      savedProperties
    }
    ... on Landlord {
      userId
      firstName
      lastName
      businessName
      verificationStatus
    }
  }
}
```

### Landlord Verification Operations

#### Become Landlord Application
```graphql
mutation BecomeLandlord($userId: ID!, $input: BecomeLandlordInput!) {
  becomeLandlord(userId: $userId, input: $input) {
    success
    message
    applicationId
    status
  }
}
```

**Input:**
```typescript
interface BecomeLandlordInput {
  businessName: string
  businessLicense: string
  taxId: string
  verificationDocuments: string[]
}
```

#### Submit Detailed Landlord Application
```graphql
mutation SubmitLandlordApplication($input: LandlordApplicationInput!) {
  submitLandlordApplication(input: $input) {
    success
    message
    applicationId
    status
    submittedAt
  }
}
```

**Input:**
```typescript
interface LandlordApplicationInput {
  userId: ID
  nationalId: string
  birthDate: string
  phoneNumber: string
  alternatePhone?: string
  address: AddressInput
}
```

## Service Layer Implementation

### File Structure
```
nest/src/service/handlers/auth/
├── sign-up.ts
├── sign-in.ts
├── verify-email.ts
├── update-profile.ts
├── become-landlord.ts
├── submit-landlord-application.ts
└── admin-user-management.ts
```

### Key Service Classes

#### AuthService
```typescript
class AuthService {
  constructor(
    private cognitoClient: CognitoIdentityProviderClient,
    private userDAO: UserDAO,
    private userEventsDAO: UserEventsDAO
  ) {}

  async signUp(input: SignUpInput): Promise<AuthResponse> {
    // 1. Validate input
    // 2. Create Cognito user
    // 3. Generate USER_CREATED event
    // 4. Materialize user profile
    // 5. Return tokens
  }

  async verifyEmail(email: string, code: string): Promise<SuccessResponse> {
    // 1. Verify with Cognito
    // 2. Generate EMAIL_VERIFIED event
    // 3. Update user status
  }

  async signIn(email: string, password: string): Promise<AuthResponse> {
    // 1. Authenticate with Cognito
    // 2. Get user profile
    // 3. Update last login
    // 4. Return tokens and profile
  }
}
```

#### UserProfileService
```typescript
class UserProfileService {
  constructor(
    private userDAO: UserDAO,
    private userEventsDAO: UserEventsDAO,
    private s3Service: S3Service
  ) {}

  async updateProfile(userId: string, input: UpdateUserInput): Promise<UserProfile> {
    // 1. Validate permissions
    // 2. Generate PROFILE_UPDATED event
    // 3. Materialize changes
    // 4. Return updated profile
  }

  async uploadProfileImage(userId: string, fileName: string): Promise<MediaUploadResponse> {
    // 1. Generate presigned S3 URL
    // 2. Return upload URL and final image URL
  }
}
```

#### LandlordVerificationService
```typescript
class LandlordVerificationService {
  constructor(
    private applicationDAO: ApplicationDAO,
    private userDAO: UserDAO,
    private notificationService: NotificationService
  ) {}

  async submitLandlordApplication(input: LandlordApplicationInput): Promise<ApplicationResponse> {
    // 1. Validate user exists and is TENANT
    // 2. Create application event
    // 3. Update user status to PENDING_LANDLORD_VERIFICATION
    // 4. Notify admins
    // 5. Return application details
  }

  async reviewLandlordApplication(
    applicationId: string, 
    status: 'APPROVED' | 'REJECTED',
    adminNotes?: string
  ): Promise<SuccessResponse> {
    // 1. Validate admin permissions
    // 2. Update application status
    // 3. If approved, upgrade user to LANDLORD
    // 4. Send notification to applicant
  }
}
```

## Data Access Layer (DAO)

### UserDAO
```typescript
class UserDAO extends BaseDAO {
  async createUser(user: BaseUser): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.usersTableName,
      Item: marshall(user)
    });
  }

  async getUserById(userId: string): Promise<UserProfile | null> {
    const result = await this.dynamoClient.getItem({
      TableName: this.usersTableName,
      Key: marshall({ userId })
    });
    return result.Item ? unmarshall(result.Item) as UserProfile : null;
  }

  async updateUser(userId: string, updates: Partial<BaseUser>): Promise<void> {
    // Implementation for atomic updates
  }

  async getUserByEmail(email: string): Promise<UserProfile | null> {
    // GSI query by email
  }
}
```

### UserEventsDAO
```typescript
class UserEventsDAO extends BaseDAO {
  async createUserEvent(event: UserEvent): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.userEventsTableName,
      Item: marshall({
        ...event,
        eventId: generateUUID(),
        timestamp: new Date().toISOString()
      })
    });
  }

  async getUserEvents(userId: string, limit?: number): Promise<UserEvent[]> {
    // Query user events by userId
  }
}
```

## Event Sourcing Implementation

### User Events
```typescript
enum UserEventType {
  USER_CREATED = 'USER_CREATED',
  EMAIL_VERIFIED = 'EMAIL_VERIFIED',
  PROFILE_UPDATED = 'PROFILE_UPDATED',
  PASSWORD_CHANGED = 'PASSWORD_CHANGED',
  ACCOUNT_SUSPENDED = 'ACCOUNT_SUSPENDED',
  ACCOUNT_ACTIVATED = 'ACCOUNT_ACTIVATED',
  LANDLORD_APPLICATION_SUBMITTED = 'LANDLORD_APPLICATION_SUBMITTED',
  LANDLORD_VERIFIED = 'LANDLORD_VERIFIED',
  PREFERENCES_UPDATED = 'PREFERENCES_UPDATED'
}

interface UserEvent {
  eventId: string
  userId: string
  eventType: UserEventType
  eventData: any
  timestamp: string
  version: number
}
```

### Materialization Service
```typescript
class UserMaterializationService {
  async materializeUserFromEvents(userId: string): Promise<UserProfile> {
    const events = await this.userEventsDAO.getUserEvents(userId);
    let user: Partial<UserProfile> = {};

    for (const event of events) {
      user = this.applyEvent(user, event);
    }

    return user as UserProfile;
  }

  private applyEvent(user: Partial<UserProfile>, event: UserEvent): Partial<UserProfile> {
    switch (event.eventType) {
      case UserEventType.USER_CREATED:
        return { ...event.eventData };
      case UserEventType.PROFILE_UPDATED:
        return { ...user, ...event.eventData, updatedAt: event.timestamp };
      case UserEventType.EMAIL_VERIFIED:
        return { ...user, isEmailVerified: true, accountStatus: 'ACTIVE' };
      // ... other event handlers
    }
  }
}
```

## Security & Validation

### Input Validation
```typescript
const signUpSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/),
  phoneNumber: z.string().regex(/^\+255[67]\d{8}$/), // Tanzania format
  firstName: z.string().min(2).max(50),
  lastName: z.string().min(2).max(50)
});
```

### Authorization Rules
- Students can only update their own profiles
- Landlords can update their profiles and manage their properties
- Admins can manage all users and moderate content
- Only verified landlords can create property listings

## Error Handling

### Common Error Types
```typescript
enum UserErrorType {
  USER_NOT_FOUND = 'USER_NOT_FOUND',
  EMAIL_ALREADY_EXISTS = 'EMAIL_ALREADY_EXISTS',
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  EMAIL_NOT_VERIFIED = 'EMAIL_NOT_VERIFIED',
  ACCOUNT_SUSPENDED = 'ACCOUNT_SUSPENDED',
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  LANDLORD_VERIFICATION_PENDING = 'LANDLORD_VERIFICATION_PENDING'
}
```

## Performance Considerations

### Caching Strategy
- User profiles cached in memory for 15 minutes
- Cognito tokens cached until expiry
- Profile images served via CloudFront CDN

### Database Optimization
- GSI on email for login lookups
- GSI on userType for admin queries
- Pagination for user lists
- Batch operations for bulk user management

## Monitoring & Logging

### Key Metrics
- User registration rate
- Email verification completion rate
- Login success/failure rates
- Landlord application approval rate
- Profile update frequency

### CloudWatch Alarms
- High authentication failure rate
- Slow response times (>2s)
- Error rate >1%
- Cognito user pool limits approaching