# Property Management System - Lower Level Design

## Overview
The Property Management System handles property listings, search, discovery, favorites, and media management for the Nest Tanzania Rental Platform.

## System Architecture Diagram

```mermaid
graph TB
    CLIENT[Client Apps<br/>Web & Mobile] --> APPSYNC[AppSync<br/>GraphQL API + Subscriptions]
    APPSYNC --> PROPERTY_LAMBDA[Property Lambda<br/>All Property Operations]
    APPSYNC --> STREAM_LAMBDA[Stream Lambda<br/>Real-time Updates]
    
    PROPERTY_LAMBDA --> DB[(DynamoDB<br/>Properties & Events)]
    STREAM_LAMBDA --> DB
    
    PROPERTY_LAMBDA --> REDIS[(Redis Cache<br/>Property Data)]
    STREAM_LAMBDA --> REDIS
    
    PROPERTY_LAMBDA --> S3[S3<br/>Property Media]
    
    DB --> STREAMS[DynamoDB Streams]
    STREAMS --> STREAM_LAMBDA
    STREAM_LAMBDA --> APPSYNC
```

## Architecture Components

### AWS Services
- **AWS AppSync**: GraphQL API with real-time subscriptions
- **AWS Lambda**: Business logic processing
- **DynamoDB**: Data persistence with event sourcing
- **DynamoDB Streams**: Real-time change notifications
- **Redis Cache**: High-performance property data caching
- **S3 + CloudFront**: Media storage and delivery
- **OpenSearch**: Advanced search capabilities (future enhancement)

### Lambda Functions
- **PropertyFunction**: `nest/src/handlers/graphql-property.ts`
  - Handles: CRUD operations, search, favorites, views, Redis caching
- **PropertyStreamFunction**: `nest/src/handlers/property-stream.ts`
  - Handles: Real-time subscription events from DynamoDB Streams, cache invalidation

### Database Tables
- **properties** (Materialized View): Current property state
- **property-events** (Event Store): Immutable property event history
- **user-activity**: Property views, favorites, and interactions
- **media-library**: Property media files and metadata

## Data Models

### Property Core Types
```typescript
enum PropertyType {
  APARTMENT = 'APARTMENT'
  HOUSE = 'HOUSE'
  STUDIO = 'STUDIO'
  ROOM = 'ROOM'
  COMMERCIAL = 'COMMERCIAL'
  LAND = 'LAND'
}

enum PropertyStatus {
  DRAFT = 'DRAFT'
  AVAILABLE = 'AVAILABLE'
  RENTED = 'RENTED'
  MAINTENANCE = 'MAINTENANCE'
  DELETED = 'DELETED'
}
```

### Property Data Model
```typescript
interface Property {
  propertyId: ID
  landlordId: ID
  managerId?: ID
  title: string
  description: string
  address: Address
  propertyType: PropertyType
  specifications: PropertySpecifications
  pricing: PropertyPricing
  amenities: string[]
  media?: PropertyMedia
  availability: PropertyAvailability
  status: PropertyStatus
  version: number
  viewCount?: number
  favoriteCount?: number
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface Address {
  street: string
  ward: string
  district: string
  region: string
  postalCode?: string
  coordinates?: Coordinates
}

interface Coordinates {
  latitude: number
  longitude: number
}

interface PropertySpecifications {
  squareMeters: number
  bedrooms?: number
  bathrooms?: number
  floors?: number
  parkingSpaces?: number
  furnished: boolean
}

interface PropertyPricing {
  monthlyRent: number
  deposit: number
  currency: string
  utilitiesIncluded: boolean
  serviceCharge?: number
}

interface PropertyMedia {
  images?: string[]
  videos?: string[]
  virtualTour?: string
  floorPlan?: string
}

interface PropertyAvailability {
  available: boolean
  availableFrom?: AWSDateTime
  minimumLeaseTerm?: number
  maximumLeaseTerm?: number
}
```

### Property Card (Optimized for Lists)
```typescript
interface PropertyCard {
  propertyId: ID
  title: string
  monthlyRent: number
  currency: string
  propertyType: PropertyType
  bedrooms?: number
  district: string
  region: string
  thumbnail?: string
  available: boolean
  viewCount?: number
  favoriteCount?: number
}
```

## API Operations

### Property Listing Management

#### Create Property
```graphql
mutation CreateProperty($landlordId: ID!, $input: CreatePropertyInput!) {
  createProperty(landlordId: $landlordId, input: $input) {
    propertyId
    title
    status
    createdAt
  }
}
```

**Input:**
```typescript
interface CreatePropertyInput {
  title: string
  description: string
  address: AddressInput
  propertyType: PropertyType
  specifications: PropertySpecificationsInput
  pricing: PropertyPricingInput
  amenities: string[]
  media?: PropertyMediaInput
  availability: PropertyAvailabilityInput
}
```

**Implementation Flow:**
1. Validate landlord permissions
2. Validate input data and location
3. Generate property event: `PROPERTY_CREATED`
4. Materialize property in properties table
5. Cache property data in Redis
6. Publish subscription event
7. Return property details

#### Update Property
```graphql
mutation UpdateProperty($propertyId: ID!, $landlordId: ID!, $input: UpdatePropertyInput!) {
  updateProperty(propertyId: $propertyId, landlordId: $landlordId, input: $input) {
    propertyId
    title
    updatedAt
    version
  }
}
```

**Implementation Flow:**
1. Validate ownership permissions
2. Compare changes with current state
3. Generate property events for each change
4. Materialize updates
5. Update Redis cache
6. Publish subscription events
7. Update search index (if applicable)


### Property Discovery & Search

#### Get Property Cards (Browse)
```graphql
query GetPropertyCards($limit: Int, $nextToken: String) {
  getPropertyCards(limit: $limit, nextToken: $nextToken) {
    properties {
      propertyId
      title
      monthlyRent
      currency
      propertyType
      bedrooms
      district
      region
      thumbnail
      available
    }
    nextToken
    count
  }
}
```

#### Get Property Details
```graphql
query GetProperty($propertyId: ID!) {
  getProperty(propertyId: $propertyId) {
    propertyId
    title
    description
    address {
      street
      ward
      district
      region
      coordinates {
        latitude
        longitude
      }
    }
    specifications {
      squareMeters
      bedrooms
      bathrooms
      furnished
    }
    pricing {
      monthlyRent
      deposit
      currency
      utilitiesIncluded
    }
    amenities
    media {
      images
      videos
      virtualTour
      floorPlan
    }
    availability {
      available
      availableFrom
      minimumLeaseTerm
    }
    status
    viewCount
  }
}
```

#### Location-Based Queries
```graphql
query GetPropertiesByLocation($region: String!, $district: String!) {
  getPropertiesByLocation(region: $region, district: $district) {
    propertyId
    title
    monthlyRent
    propertyType
    thumbnail
  }
}

query GetNearbyProperties($lat: Float!, $lng: Float!, $radiusKm: Float) {
  getNearbyProperties(lat: $lat, lng: $lng, radiusKm: $radiusKm) {
    propertyId
    title
    monthlyRent
    address {
      district
      coordinates {
        latitude
        longitude
      }
    }
  }
}
```

### Property Favorites & Activity

#### Add to Favorites
```graphql
mutation AddToFavorites($userId: ID!, $propertyId: ID!) {
  addToFavorites(userId: $userId, propertyId: $propertyId) {
    success
    message
  }
}
```

#### Track Property View
```graphql
mutation TrackPropertyView($userId: ID!, $propertyId: ID!) {
  trackPropertyView(userId: $userId, propertyId: $propertyId) {
    success
    viewCount
  }
}
```

### Real-time Subscriptions

#### Property Updates
```graphql
subscription OnPropertyUpdated($propertyId: ID!) {
  onPropertyUpdated(propertyId: $propertyId) {
    propertyId
    eventType
    property {
      title
      pricing {
        monthlyRent
      }
      status
    }
    changes {
      field
      oldValue
      newValue
    }
    timestamp
  }
}
```

#### New Properties in Region
```graphql
subscription OnNewPropertyInRegion($region: String!) {
  onNewPropertyInRegion(region: $region) {
    propertyId
    title
    monthlyRent
    district
    thumbnail
  }
}
```

## Property Flow Diagrams

### Property Creation Flow

```mermaid
sequenceDiagram
    participant L as Landlord
    participant W as Web App
    participant A as AppSync
    participant PL as Property Lambda
    participant D as DynamoDB
    participant R as Redis Cache
    participant S3 as S3 Storage

    L->>W: Create property form
    W->>S3: Upload property images
    S3-->>W: Image URLs
    W->>A: createProperty mutation
    A->>PL: Process creation
    PL->>D: Store PROPERTY_CREATED event
    PL->>D: Materialize property
    PL->>R: Cache property data
    PL-->>A: Property created
    A-->>W: Success response
    W-->>L: Show property listing
```

### Property Search Flow

```mermaid
sequenceDiagram
    participant T as Tenant
    participant W as Web App
    participant A as AppSync
    participant PL as Property Lambda
    participant R as Redis Cache
    participant D as DynamoDB

    T->>W: Search properties
    W->>A: searchProperties query
    A->>PL: Process search
    alt Fast loading from cache
        PL->>R: Get cached results
        R-->>PL: Property data
    else Cache miss
        PL->>D: Query properties table
        D-->>PL: Property data
        PL->>R: Cache results
    end
    PL-->>A: Search results
    A-->>W: Property cards
    W-->>T: Display results
```

### Real-time Property Updates Flow

```mermaid
sequenceDiagram
    participant L as Landlord
    participant W1 as Landlord App
    participant A as AppSync
    participant PL as Property Lambda
    participant D as DynamoDB
    participant DS as DynamoDB Streams
    participant SL as Stream Lambda
    participant R as Redis Cache
    participant W2 as Tenant App
    participant T as Tenant

    L->>W1: Update property
    W1->>A: updateProperty mutation
    A->>PL: Process update
    PL->>D: Store PROPERTY_UPDATED event
    PL->>R: Update cache
    D->>DS: Stream change event
    DS->>SL: Trigger stream handler
    SL->>R: Invalidate related cache
    SL->>A: Publish subscription
    A->>W2: Real-time update
    W2->>T: Show updated property
```