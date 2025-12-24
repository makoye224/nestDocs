# Property Management System - Lower Level Design

## Overview
The Property Management System handles property listings, search, discovery, favorites, and media management for the Nest Tanzania Rental Platform.

## Architecture Components

### AWS Services
- **AWS AppSync**: GraphQL API with real-time subscriptions
- **AWS Lambda**: Business logic processing
- **DynamoDB**: Data persistence with event sourcing
- **DynamoDB Streams**: Real-time change notifications
- **S3 + CloudFront**: Media storage and delivery
- **OpenSearch**: Advanced search capabilities (future enhancement)

### Lambda Functions
- **PropertyFunction**: `nest/src/handlers/graphql-property.ts`
- **PropertyStreamFunction**: `nest/src/handlers/property-stream.ts`
- **PropertyJsonFunction**: `nest/src/handlers/property-json-generator.ts`

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
5. Trigger property JSON regeneration
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
5. Publish subscription events
6. Update search index (if applicable)

#### Update Property Status
```graphql
mutation UpdatePropertyStatus($propertyId: ID!, $landlordId: ID!, $status: PropertyStatus!) {
  updatePropertyStatus(propertyId: $propertyId, landlordId: $landlordId, status: $status) {
    propertyId
    status
    updatedAt
  }
}
```

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

#### Advanced Property Search
```graphql
query SearchProperties(
  $region: String
  $district: String
  $minPrice: Float
  $maxPrice: Float
  $propertyType: PropertyType
  $bedrooms: Int
  $limit: Int
  $from: Int
  $q: String
) {
  searchProperties(
    region: $region
    district: $district
    minPrice: $minPrice
    maxPrice: $maxPrice
    propertyType: $propertyType
    bedrooms: $bedrooms
    limit: $limit
    from: $from
    q: $q
  ) {
    properties {
      propertyId
      title
      monthlyRent
      propertyType
      bedrooms
      district
      region
      thumbnail
      available
    }
    count
    total
    from
    size
    nextToken
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

## Service Layer Implementation

### File Structure
```
nest/src/service/handlers/property/
├── CreatePropertyService.ts
├── UpdatePropertyService.ts
├── PropertySearchService.ts
├── PropertyViewService.ts
├── PropertyFavoritesService.ts
├── PropertyMediaService.ts
├── PropertyJsonService.ts
└── PropertySubscriptionService.ts
```

### Key Service Classes

#### CreatePropertyService
```typescript
class CreatePropertyService {
  constructor(
    private propertyDAO: PropertyDAO,
    private propertyEventsDAO: PropertyEventsDAO,
    private locationDAO: LocationDAO,
    private subscriptionService: PropertySubscriptionService
  ) {}

  async createProperty(landlordId: string, input: CreatePropertyInput): Promise<Property> {
    // 1. Validate landlord is verified
    // 2. Validate location exists
    // 3. Validate input data
    // 4. Generate propertyId
    // 5. Create PROPERTY_CREATED event
    // 6. Materialize property
    // 7. Publish subscription event
    // 8. Return property
  }

  private async validateLocation(address: AddressInput): Promise<boolean> {
    // Verify region, district, ward, street exist in location hierarchy
  }
}
```

#### PropertySearchService
```typescript
class PropertySearchService {
  constructor(
    private propertyDAO: PropertyDAO,
    private userActivityDAO: UserActivityDAO
  ) {}

  async searchProperties(filters: PropertySearchFilters): Promise<PropertySearchResponse> {
    // 1. Build DynamoDB query with filters
    // 2. Apply pagination
    // 3. Execute query
    // 4. Transform to PropertyCard format
    // 5. Return paginated results
  }

  async getPropertiesByLocation(region: string, district: string): Promise<Property[]> {
    // GSI query by location
  }

  async getNearbyProperties(lat: number, lng: number, radiusKm: number): Promise<Property[]> {
    // Geospatial query implementation
  }
}
```

#### PropertyViewService
```typescript
class PropertyViewService {
  constructor(
    private userActivityDAO: UserActivityDAO,
    private propertyDAO: PropertyDAO
  ) {}

  async trackPropertyView(userId: string, propertyId: string): Promise<void> {
    // 1. Record view in user-activity table
    // 2. Increment property view count
    // 3. Update user's recently viewed list
  }

  async getRecentlyViewed(userId: string, limit: number = 10): Promise<PropertyCard[]> {
    // Get user's recent property views
  }
}
```

#### PropertyFavoritesService
```typescript
class PropertyFavoritesService {
  constructor(
    private userActivityDAO: UserActivityDAO,
    private propertyDAO: PropertyDAO
  ) {}

  async addToFavorites(userId: string, propertyId: string): Promise<void> {
    // 1. Add to user's favorites list
    // 2. Increment property favorite count
    // 3. Create activity record
  }

  async removeFromFavorites(userId: string, propertyId: string): Promise<void> {
    // 1. Remove from user's favorites
    // 2. Decrement property favorite count
  }

  async getFavoriteProperties(userId: string): Promise<PropertyCard[]> {
    // Get user's favorite properties
  }
}
```

#### PropertyMediaService
```typescript
class PropertyMediaService {
  constructor(
    private s3Service: S3Service,
    private mediaDAO: MediaDAO,
    private propertyDAO: PropertyDAO
  ) {}

  async getMediaUploadUrl(
    userId: string, 
    fileName: string, 
    contentType: string
  ): Promise<MediaUploadResponse> {
    // 1. Generate S3 presigned URL
    // 2. Create media record
    // 3. Return upload URL and final file URL
  }

  async associateMediaWithProperty(
    propertyId: string, 
    landlordId: string, 
    media: PropertyMediaInput
  ): Promise<Property> {
    // 1. Validate ownership
    // 2. Update property media
    // 3. Generate MEDIA_UPDATED event
    // 4. Return updated property
  }

  async generateThumbnail(imageUrl: string): Promise<string> {
    // Generate optimized thumbnail for property cards
  }
}
```

## Data Access Layer (DAO)

### PropertyDAO
```typescript
class PropertyDAO extends BaseDAO {
  async createProperty(property: Property): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.propertiesTableName,
      Item: marshall(property),
      ConditionExpression: 'attribute_not_exists(propertyId)'
    });
  }

  async getPropertyById(propertyId: string): Promise<Property | null> {
    const result = await this.dynamoClient.getItem({
      TableName: this.propertiesTableName,
      Key: marshall({ propertyId })
    });
    return result.Item ? unmarshall(result.Item) as Property : null;
  }

  async getPropertiesByLandlord(
    landlordId: string, 
    limit?: number, 
    nextToken?: string
  ): Promise<PropertyListResponse> {
    // GSI query by landlordId with pagination
  }

  async searchProperties(filters: PropertySearchFilters): Promise<PropertySearchResponse> {
    // Complex query with multiple filters
    // Consider using FilterExpression for non-key attributes
  }

  async getPropertiesByLocation(region: string, district: string): Promise<Property[]> {
    // GSI query by location
  }

  async updateProperty(propertyId: string, updates: Partial<Property>): Promise<void> {
    // Atomic update with version checking
  }

  async incrementViewCount(propertyId: string): Promise<number> {
    const result = await this.dynamoClient.updateItem({
      TableName: this.propertiesTableName,
      Key: marshall({ propertyId }),
      UpdateExpression: 'ADD viewCount :inc',
      ExpressionAttributeValues: marshall({ ':inc': 1 }),
      ReturnValues: 'UPDATED_NEW'
    });
    return unmarshall(result.Attributes!).viewCount;
  }
}
```

### PropertyEventsDAO
```typescript
class PropertyEventsDAO extends BaseDAO {
  async createPropertyEvent(event: PropertyEvent): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.propertyEventsTableName,
      Item: marshall({
        ...event,
        eventId: generateUUID(),
        timestamp: new Date().toISOString()
      })
    });
  }

  async getPropertyEvents(propertyId: string): Promise<PropertyEvent[]> {
    // Query all events for a property
  }

  async getPropertyEventsByType(
    propertyId: string, 
    eventType: PropertyEventType
  ): Promise<PropertyEvent[]> {
    // Query events by type
  }
}
```

### UserActivityDAO
```typescript
class UserActivityDAO extends BaseDAO {
  async recordPropertyView(userId: string, propertyId: string): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.userActivityTableName,
      Item: marshall({
        userId,
        activityType: 'PROPERTY_VIEW',
        propertyId,
        timestamp: new Date().toISOString(),
        ttl: Math.floor(Date.now() / 1000) + (90 * 24 * 60 * 60) // 90 days TTL
      })
    });
  }

  async addToFavorites(userId: string, propertyId: string): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.userActivityTableName,
      Item: marshall({
        userId,
        activityType: 'FAVORITE_ADDED',
        propertyId,
        timestamp: new Date().toISOString()
      }),
      ConditionExpression: 'attribute_not_exists(userId) AND attribute_not_exists(propertyId)'
    });
  }

  async getFavoriteProperties(userId: string): Promise<string[]> {
    // Query user's favorite properties
  }

  async getRecentlyViewed(userId: string, limit: number): Promise<string[]> {
    // Query recent property views with limit
  }
}
```

## Event Sourcing Implementation

### Property Events
```typescript
enum PropertyEventType {
  PROPERTY_CREATED = 'PROPERTY_CREATED',
  PROPERTY_UPDATED = 'PROPERTY_UPDATED',
  PRICE_CHANGED = 'PRICE_CHANGED',
  STATUS_CHANGED = 'STATUS_CHANGED',
  AVAILABILITY_CHANGED = 'AVAILABILITY_CHANGED',
  MEDIA_UPDATED = 'MEDIA_UPDATED',
  DESCRIPTION_UPDATED = 'DESCRIPTION_UPDATED',
  PROPERTY_DELETED = 'PROPERTY_DELETED'
}

interface PropertyEvent {
  eventId: string
  propertyId: string
  landlordId: string
  eventType: PropertyEventType
  eventData: any
  changes?: PropertyChange[]
  timestamp: string
  version: number
}

interface PropertyChange {
  field: string
  oldValue?: string
  newValue: string
}
```

### Property Stream Handler
```typescript
// nest/src/handlers/property-stream.ts
export const propertyStreamHandler = async (event: DynamoDBStreamEvent): Promise<void> => {
  for (const record of event.Records) {
    if (record.eventName === 'MODIFY') {
      const newImage = record.dynamodb?.NewImage;
      const oldImage = record.dynamodb?.OldImage;
      
      if (newImage && oldImage) {
        const changes = detectChanges(oldImage, newImage);
        await publishPropertyUpdateEvent(newImage.propertyId.S!, changes);
      }
    } else if (record.eventName === 'INSERT') {
      const newImage = record.dynamodb?.NewImage;
      if (newImage) {
        await publishNewPropertyEvent(newImage);
      }
    }
  }
};
```

## Real-time Subscriptions Implementation

### Subscription Publishing
```typescript
class PropertySubscriptionService {
  constructor(private appSyncClient: AppSyncClient) {}

  async publishNewPropertyEvent(propertyId: string, region: string): Promise<void> {
    const mutation = `
      mutation PublishNewPropertyEvent($propertyId: ID!, $region: String!) {
        publishNewPropertyEvent(propertyId: $propertyId, region: $region) {
          success
          message
        }
      }
    `;

    await this.appSyncClient.graphql({
      query: mutation,
      variables: { propertyId, region }
    });
  }

  async publishPropertyUpdateEvent(input: PropertyUpdateEventInput): Promise<void> {
    const mutation = `
      mutation PublishPropertyUpdateEvent($input: PropertyUpdateEventInput!) {
        publishPropertyUpdateEvent(input: $input) {
          propertyId
          eventType
          changes {
            field
            oldValue
            newValue
          }
          timestamp
        }
      }
    `;

    await this.appSyncClient.graphql({
      query: mutation,
      variables: { input }
    });
  }
}
```

## Performance Optimizations

### Database Indexes
```typescript
// Global Secondary Indexes
const propertyIndexes = {
  // For landlord property listings
  'landlordId-createdAt-index': {
    partitionKey: 'landlordId',
    sortKey: 'createdAt'
  },
  
  // For location-based searches
  'region-district-index': {
    partitionKey: 'region',
    sortKey: 'district'
  },
  
  // For price-based searches
  'propertyType-monthlyRent-index': {
    partitionKey: 'propertyType',
    sortKey: 'monthlyRent'
  },
  
  // For status filtering
  'status-updatedAt-index': {
    partitionKey: 'status',
    sortKey: 'updatedAt'
  }
};
```

### Caching Strategy
- Property cards cached for 5 minutes
- Property details cached for 15 minutes
- Search results cached for 2 minutes
- Media URLs cached via CloudFront CDN

### Property JSON Generation
```typescript
// Daily batch job to generate optimized property JSON
class PropertyJsonService {
  async generatePropertyJson(): Promise<void> {
    // 1. Query all active properties
    // 2. Transform to optimized format
    // 3. Upload to S3
    // 4. Invalidate CloudFront cache
  }
}
```

## Search Implementation

### Basic Search (DynamoDB)
```typescript
class BasicPropertySearch {
  async searchByFilters(filters: PropertySearchFilters): Promise<Property[]> {
    let query = this.dynamoClient.scan({
      TableName: this.propertiesTableName,
      FilterExpression: 'attribute_exists(propertyId)'
    });

    // Add filters
    if (filters.region) {
      query.FilterExpression += ' AND #region = :region';
      query.ExpressionAttributeNames = { '#region': 'region' };
      query.ExpressionAttributeValues = { ':region': { S: filters.region } };
    }

    // Execute query with pagination
    return this.executePaginatedQuery(query, filters.limit, filters.nextToken);
  }
}
```

### Advanced Search (Future: OpenSearch)
```typescript
class AdvancedPropertySearch {
  async searchProperties(query: string, filters: PropertySearchFilters): Promise<Property[]> {
    const searchQuery = {
      index: 'properties',
      body: {
        query: {
          bool: {
            must: [
              {
                multi_match: {
                  query: query,
                  fields: ['title^2', 'description', 'amenities']
                }
              }
            ],
            filter: this.buildFilters(filters)
          }
        },
        sort: [
          { _score: { order: 'desc' } },
          { monthlyRent: { order: 'asc' } }
        ]
      }
    };

    return this.openSearchClient.search(searchQuery);
  }
}
```

## Error Handling & Validation

### Input Validation
```typescript
const createPropertySchema = z.object({
  title: z.string().min(10).max(200),
  description: z.string().min(50).max(2000),
  address: z.object({
    street: z.string().min(5),
    ward: z.string().min(2),
    district: z.string().min(2),
    region: z.string().min(2)
  }),
  pricing: z.object({
    monthlyRent: z.number().positive(),
    deposit: z.number().nonnegative(),
    currency: z.enum(['TZS', 'USD'])
  }),
  specifications: z.object({
    squareMeters: z.number().positive(),
    bedrooms: z.number().int().nonnegative().optional(),
    bathrooms: z.number().int().nonnegative().optional()
  })
});
```

### Error Types
```typescript
enum PropertyErrorType {
  PROPERTY_NOT_FOUND = 'PROPERTY_NOT_FOUND',
  UNAUTHORIZED_ACCESS = 'UNAUTHORIZED_ACCESS',
  INVALID_LOCATION = 'INVALID_LOCATION',
  PROPERTY_ALREADY_RENTED = 'PROPERTY_ALREADY_RENTED',
  MEDIA_UPLOAD_FAILED = 'MEDIA_UPLOAD_FAILED',
  SEARCH_LIMIT_EXCEEDED = 'SEARCH_LIMIT_EXCEEDED'
}
```

## Monitoring & Analytics

### Key Metrics
- Property creation rate
- Search query performance
- Property view counts
- Favorite conversion rates
- Media upload success rates
- Subscription event delivery rates

### CloudWatch Dashboards
- Property listing trends
- Search performance metrics
- Real-time subscription activity
- Media storage usage