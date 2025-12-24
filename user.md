# User Management System - Lower Level Design

## Overview
The User Management System handles authentication, authorization, and user profile management for Tenants, Landlords, and Admins in the Nest Tanzania Rental Platform.

## System Architecture Diagram

```mermaid
graph TB
    CLIENT[Client Apps<br/>Web & Mobile] --> APPSYNC[AppSync<br/>GraphQL API]
    APPSYNC --> LAMBDA[User Lambda<br/>All User Operations]
    LAMBDA --> COGNITO[Cognito<br/>Authentication]
    LAMBDA --> DB[(DynamoDB<br/>Users & Events)]
    LAMBDA --> S3[S3<br/>Profile Images]
    LAMBDA --> EMAIL[Email/SMS<br/>Notifications]
```

## Architecture Components

### AWS Services
- **AWS Cognito User Pool**: Authentication and user management
- **AWS AppSync**: GraphQL API gateway
- **AWS Lambda**: Business logic processing
- **DynamoDB**: Data persistence with event sourcing
- **S3 + CloudFront**: Profile image storage

### Lambda Functions
- **UserFunction**: `nest/src/handlers/graphql-auth.ts`
- **Handler**: `rental-auth-{stage}`
- **Handles**: Authentication, profile management, and landlord verification
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

### Tenant Profile
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
  verificationDocuments?: string[]
  nationalId?: string
  birthDate?: string
  alternatePhone?: string
  address?: Address
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



## User Flow Diagrams

### Tenant Registration Flow

```mermaid
sequenceDiagram
    participant T as Tenant
    participant W as Web App
    participant A as AppSync
    participant L as User Lambda
    participant C as Cognito
    participant D as DynamoDB

    T->>W: Fill registration form
    W->>A: signUp mutation
    A->>L: Process registration
    L->>C: Create user account
    L->>D: Store USER_CREATED event
    L-->>A: Return tokens & user data
    A-->>W: Registration response
    W-->>T: Show verification prompt
    
    T->>W: Enter verification code
    W->>A: verifyEmail mutation
    A->>L: Verify code
    L->>C: Confirm verification
    L->>D: Store EMAIL_VERIFIED event
    L-->>A: Verification success
    A-->>W: Success response
    W-->>T: Redirect to dashboard
```

### Landlord Verification Flow

```mermaid
sequenceDiagram
    participant L as Landlord
    participant W as Web App
    participant A as AppSync
    participant UL as User Lambda
    participant D as DynamoDB
    participant AD as Admin

    L->>W: Submit landlord application
    W->>A: submitLandlordApplication mutation
    A->>UL: Process application
    UL->>D: Store application events
    UL-->>A: Application submitted
    A-->>W: Success response
    W-->>L: Show pending status

    AD->>W: Review application
    W->>A: approveLandlordApplication mutation
    A->>UL: Process approval
    UL->>D: Store LANDLORD_VERIFIED event
    UL-->>A: Approval processed
    A-->>W: Success response
```