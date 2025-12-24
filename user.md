# User Management System - Lower Level Design

## Overview
The User Management System handles authentication, authorization, and user profile management for Students, Landlords, and Admins in the Nest Tanzania Rental Platform.

## System Architecture Diagram

```mermaid
graph TB
    %% Client Layer
    subgraph "Client Applications"
        WEB[Web App<br/>React/Next.js]
        MOBILE[Mobile App<br/>React Native]
    end

    %% API Gateway Layer
    subgraph "API Gateway"
        APPSYNC[AWS AppSync<br/>GraphQL API]
    end

    %% Authentication Layer
    subgraph "Authentication"
        COGNITO[AWS Cognito<br/>User Pool]
        JWT[JWT Tokens]
    end

    %% Lambda Functions Layer
    subgraph "Lambda Functions"
        AUTH_LAMBDA[Auth Handler<br/>graphql-auth.ts]
        USER_LAMBDA[User Handler<br/>user-operations.ts]
        LANDLORD_LAMBDA[Landlord Handler<br/>landlord-verification.ts]
    end

    %% Database Layer
    subgraph "Database Layer"
        USERS_TABLE[(users<br/>Materialized View)]
        EVENTS_TABLE[(user-events<br/>Event Store)]
        APP_EVENTS[(application-events<br/>Landlord Applications)]
    end

    %% Storage Layer
    subgraph "Storage"
        S3[S3 Bucket<br/>Profile Images]
        CLOUDFRONT[CloudFront<br/>CDN]
    end

    %% External Services
    subgraph "External Services"
        EMAIL[Email Service<br/>SES/SNS]
        SMS[SMS Service<br/>SNS]
    end

    %% Connections
    WEB --> APPSYNC
    MOBILE --> APPSYNC
    
    APPSYNC --> AUTH_LAMBDA
    APPSYNC --> USER_LAMBDA
    APPSYNC --> LANDLORD_LAMBDA
    
    AUTH_LAMBDA --> COGNITO
    AUTH_LAMBDA --> USERS_TABLE
    AUTH_LAMBDA --> EVENTS_TABLE
    
    USER_LAMBDA --> USERS_TABLE
    USER_LAMBDA --> EVENTS_TABLE
    USER_LAMBDA --> S3
    
    LANDLORD_LAMBDA --> APP_EVENTS
    LANDLORD_LAMBDA --> EVENTS_TABLE
    LANDLORD_LAMBDA --> USERS_TABLE
    
    COGNITO --> JWT
    JWT --> APPSYNC
    
    S3 --> CLOUDFRONT
    CLOUDFRONT --> WEB
    CLOUDFRONT --> MOBILE
    
    AUTH_LAMBDA --> EMAIL
    LANDLORD_LAMBDA --> EMAIL
    AUTH_LAMBDA --> SMS

    %% Styling
    classDef client fill:#e1f5fe
    classDef api fill:#f3e5f5
    classDef auth fill:#fff3e0
    classDef lambda fill:#e8f5e8
    classDef database fill:#fce4ec
    classDef storage fill:#f1f8e9
    classDef external fill:#fff8e1

    class WEB,MOBILE client
    class APPSYNC api
    class COGNITO,JWT auth
    class AUTH_LAMBDA,USER_LAMBDA,LANDLORD_LAMBDA lambda
    class USERS_TABLE,EVENTS_TABLE,APP_EVENTS database
    class S3,CLOUDFRONT storage
    class EMAIL,SMS external
```

## User Flow Diagrams

### Student Registration Flow

```mermaid
sequenceDiagram
    participant U as Student
    participant W as Web App
    participant A as AppSync
    participant L as Auth Lambda
    participant C as Cognito
    participant D as DynamoDB
    participant E as Email Service

    U->>W: Fill registration form
    W->>A: signUp mutation
    A->>L: Process registration
    L->>C: Create user account
    C-->>L: User created
    L->>D: Store USER_CREATED event
    L->>D: Materialize user profile
    L->>E: Send verification email
    L-->>A: Return tokens & user data
    A-->>W: Registration response
    W-->>U: Show verification prompt
    
    Note over U,E: Email verification flow
    U->>W: Enter verification code
    W->>A: verifyEmail mutation
    A->>L: Verify code
    L->>C: Confirm verification
    L->>D: Store EMAIL_VERIFIED event
    L->>D: Update user status to ACTIVE
    L-->>A: Verification success
    A-->>W: Success response
    W-->>U: Redirect to dashboard
```

### Landlord Verification Flow

```mermaid
sequenceDiagram
    participant L as Landlord
    participant W as Web App
    participant A as AppSync
    participant LH as Landlord Handler
    participant D as DynamoDB
    participant AD as Admin
    participant E as Email Service

    L->>W: Submit landlord application
    W->>A: submitLandlordApplication mutation
    A->>LH: Process application
    LH->>D: Store application in application-events
    LH->>D: Store LANDLORD_APPLICATION_SUBMITTED event
    LH->>D: Update user status to PENDING_LANDLORD_VERIFICATION
    LH->>E: Notify admin team
    LH-->>A: Application submitted
    A-->>W: Success response
    W-->>L: Show pending status

    Note over AD: Admin reviews application
    AD->>W: Review application
    W->>A: approveLandlordApplication mutation
    A->>LH: Process approval
    LH->>D: Store LANDLORD_VERIFIED event
    LH->>D: Update user type to LANDLORD
    LH->>D: Update status to ACTIVE
    LH->>E: Send approval email
    LH-->>A: Approval processed
    A-->>W: Success response
    W-->>L: Email notification
```

### User Profile Update Flow

```mermaid
sequenceDiagram
    participant U as User
    participant W as Web App
    participant A as AppSync
    participant UH as User Handler
    participant D as DynamoDB
    participant S3 as S3 Storage

    U->>W: Update profile form
    alt Profile image upload
        W->>S3: Upload image
        S3-->>W: Image URL
    end
    W->>A: updateUser mutation
    A->>UH: Process update
    UH->>D: Store USER_UPDATED event
    UH->>D: Update materialized view
    UH-->>A: Update success
    A-->>W: Updated user data
    W-->>U: Show success message
```

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

#### Submit Landlord Application
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