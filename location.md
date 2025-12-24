# Location Management System - Lower Level Design

## Overview
The Location Management System provides hierarchical location data (Region → District → Ward → Street) for property addressing and search functionality in the Nest Tanzania Rental Platform.

## Architecture Components

### AWS Services
- **AWS AppSync**: GraphQL API for location queries
- **AWS Lambda**: Location data processing and management
- **DynamoDB**: Hierarchical location storage with event sourcing
- **S3 + CloudFront**: Location JSON cache for fast client access
- **EventBridge**: Batch processing triggers

### Lambda Functions
- **LocationFunction**: `nest/src/handlers/graphql-location.ts`
- **BatchJobFunction**: `nest/src/handlers/batch-job.ts` (CSV imports)

### Database Tables
- **locations** (Materialized View): Current location hierarchy
- **location-events** (Event Store): Immutable location change history

## Data Models

### Location Hierarchy Types
```typescript
enum LocationType {
  REGION = 'REGION'
  DISTRICT = 'DISTRICT'
  WARD = 'WARD'
  STREET = 'STREET'
}
```

### Location Entities
```typescript
interface Region {
  id: ID
  name: string
  code?: string
  population?: number
  area?: number
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface District {
  id: ID
  name: string
  regionId: ID
  code?: string
  population?: number
  area?: number
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface Ward {
  id: ID
  name: string
  districtId: ID
  code?: string
  population?: number
  area?: number
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface Street {
  id: ID
  name: string
  wardId: ID
  postalCode?: string
  coordinates?: Coordinates
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface Coordinates {
  latitude: number
  longitude: number
}
```

### Unified Location Model
```typescript
interface Location {
  locationId: ID
  type: LocationType
  name: string
  parentId?: ID
  level: number // 1=Region, 2=District, 3=Ward, 4=Street
  hierarchy: string // "region/district/ward/street"
  code?: string
  metadata?: LocationMetadata
  coordinates?: Coordinates
  createdAt: AWSDateTime
  updatedAt: AWSDateTime
}

interface LocationMetadata {
  population?: number
  area?: number
  postalCode?: string
  economicZone?: string
  administrativeCode?: string
}
```

### Location Tree Structure
```typescript
interface LocationTree {
  regions: Region[]
  districts: { [regionId: string]: District[] }
  wards: { [districtId: string]: Ward[] }
  streets: { [wardId: string]: Street[] }
}
```

## API Operations

### Location Hierarchy Queries

#### Get All Regions
```graphql
query GetRegions {
  getRegions {
    id
    name
    code
    population
  }
}
```

**Implementation:**
- Query locations table where type = 'REGION'
- Cache results for 1 hour (regions rarely change)
- Return sorted by name

#### Get Districts by Region
```graphql
query GetDistricts($regionId: ID!) {
  getDistricts(regionId: $regionId) {
    id
    name
    regionId
    code
    population
  }
}
```

**Implementation:**
- Query locations table where type = 'DISTRICT' AND parentId = regionId
- Use GSI: parentId-name-index
- Cache results for 30 minutes

#### Get Wards by District
```graphql
query GetWards($districtId: ID!) {
  getWards(districtId: $districtId) {
    id
    name
    districtId
    code
  }
}
```

#### Get Streets by Ward
```graphql
query GetStreets($wardId: ID!) {
  getStreets(wardId: $wardId) {
    id
    name
    wardId
    postalCode
    coordinates {
      latitude
      longitude
    }
  }
}
```

### Location Management Operations

#### Create Location
```graphql
mutation CreateLocation($input: CreateLocationInput!) {
  createLocation(input: $input) {
    success
    location {
      ... on Region {
        id
        name
      }
      ... on District {
        id
        name
        regionId
      }
      ... on Ward {
        id
        name
        districtId
      }
      ... on Street {
        id
        name
        wardId
      }
    }
    message
  }
}
```

**Input:**
```typescript
interface CreateLocationInput {
  type: LocationType
  name: string
  parent?: string // parentId for hierarchical locations
  code?: string
  coordinates?: CoordinatesInput
  metadata?: LocationMetadataInput
}
```

**Implementation Flow:**
1. Validate location type and parent relationship
2. Check for duplicate names within same parent
3. Generate location event: `LOCATION_CREATED`
4. Materialize location in locations table
5. Update location JSON cache
6. Return created location

#### Update Location
```graphql
mutation UpdateLocation($locationId: ID!, $name: String!) {
  updateLocation(locationId: $locationId, name: $name) {
    success
    message
    location {
      id
      name
    }
  }
}
```

#### Bulk Import from CSV
```graphql
mutation ImportLocationsFromCSV($csvData: String!) {
  importLocationsFromCSV(csvData: $csvData) {
    success
    imported
    skipped
    errors
    message
  }
}
```

### Location JSON Cache Management

#### Regenerate Location JSON
```graphql
mutation RegenerateLocationJson {
  regenerateLocationJson {
    success
    cloudfrontUrl
    message
  }
}
```

**Implementation:**
- Generate complete location hierarchy JSON
- Upload to S3 with public read access
- Invalidate CloudFront cache
- Return CDN URL for client consumption

## Service Layer Implementation

### File Structure
```
nest/src/service/handlers/location/
├── LocationQueryService.ts
├── LocationManagementService.ts
├── LocationImportService.ts
├── LocationJsonService.ts
└── LocationValidationService.ts
```

### Key Service Classes

#### LocationQueryService
```typescript
class LocationQueryService {
  constructor(
    private locationDAO: LocationDAO,
    private cacheService: CacheService
  ) {}

  async getRegions(): Promise<Region[]> {
    const cacheKey = 'locations:regions';
    let regions = await this.cacheService.get(cacheKey);
    
    if (!regions) {
      regions = await this.locationDAO.getLocationsByType('REGION');
      await this.cacheService.set(cacheKey, regions, 3600); // 1 hour
    }
    
    return regions.sort((a, b) => a.name.localeCompare(b.name));
  }

  async getDistricts(regionId: string): Promise<District[]> {
    const cacheKey = `locations:districts:${regionId}`;
    let districts = await this.cacheService.get(cacheKey);
    
    if (!districts) {
      districts = await this.locationDAO.getLocationsByParent(regionId);
      await this.cacheService.set(cacheKey, districts, 1800); // 30 minutes
    }
    
    return districts.sort((a, b) => a.name.localeCompare(b.name));
  }

  async getWards(districtId: string): Promise<Ward[]> {
    return this.locationDAO.getLocationsByParent(districtId);
  }

  async getStreets(wardId: string): Promise<Street[]> {
    return this.locationDAO.getLocationsByParent(wardId);
  }

  async getLocationHierarchy(locationId: string): Promise<string[]> {
    // Return full hierarchy path: [regionId, districtId, wardId, streetId]
    const location = await this.locationDAO.getLocationById(locationId);
    if (!location) return [];

    const hierarchy = [locationId];
    let currentLocation = location;

    while (currentLocation.parentId) {
      hierarchy.unshift(currentLocation.parentId);
      currentLocation = await this.locationDAO.getLocationById(currentLocation.parentId);
    }

    return hierarchy;
  }
}
```

#### LocationManagementService
```typescript
class LocationManagementService {
  constructor(
    private locationDAO: LocationDAO,
    private locationEventsDAO: LocationEventsDAO,
    private validationService: LocationValidationService,
    private jsonService: LocationJsonService
  ) {}

  async createLocation(input: CreateLocationInput): Promise<LocationCreateResponse> {
    // 1. Validate input and hierarchy
    await this.validationService.validateCreateInput(input);

    // 2. Check for duplicates
    const existing = await this.locationDAO.findByNameAndParent(input.name, input.parent);
    if (existing) {
      throw new Error(`Location '${input.name}' already exists in this area`);
    }

    // 3. Generate location ID and hierarchy path
    const locationId = generateLocationId(input.type, input.name);
    const hierarchy = await this.buildHierarchyPath(input.parent, input.name);

    // 4. Create location event
    const event: LocationEvent = {
      eventId: generateUUID(),
      locationId,
      eventType: LocationEventType.LOCATION_CREATED,
      eventData: {
        ...input,
        locationId,
        hierarchy,
        level: this.getLocationLevel(input.type)
      },
      timestamp: new Date().toISOString(),
      version: 1
    };

    await this.locationEventsDAO.createLocationEvent(event);

    // 5. Materialize location
    const location = await this.materializeLocation(event);
    await this.locationDAO.createLocation(location);

    // 6. Update JSON cache
    await this.jsonService.regenerateLocationJson();

    return {
      success: true,
      location: this.transformToUnion(location),
      message: `${input.type} '${input.name}' created successfully`
    };
  }

  async updateLocation(locationId: string, name: string): Promise<LocationUpdateResponse> {
    // 1. Get current location
    const currentLocation = await this.locationDAO.getLocationById(locationId);
    if (!currentLocation) {
      throw new Error('Location not found');
    }

    // 2. Validate new name
    await this.validationService.validateLocationName(name, currentLocation.type);

    // 3. Create update event
    const event: LocationEvent = {
      eventId: generateUUID(),
      locationId,
      eventType: LocationEventType.LOCATION_UPDATED,
      eventData: {
        oldName: currentLocation.name,
        newName: name
      },
      timestamp: new Date().toISOString(),
      version: currentLocation.version + 1
    };

    await this.locationEventsDAO.createLocationEvent(event);

    // 4. Update materialized view
    await this.locationDAO.updateLocation(locationId, { 
      name, 
      updatedAt: new Date().toISOString(),
      version: event.version
    });

    // 5. Update JSON cache
    await this.jsonService.regenerateLocationJson();

    return {
      success: true,
      message: `Location updated to '${name}'`,
      location: await this.locationDAO.getLocationById(locationId) as Region
    };
  }

  private getLocationLevel(type: LocationType): number {
    const levels = {
      [LocationType.REGION]: 1,
      [LocationType.DISTRICT]: 2,
      [LocationType.WARD]: 3,
      [LocationType.STREET]: 4
    };
    return levels[type];
  }

  private async buildHierarchyPath(parentId?: string, currentName?: string): Promise<string> {
    if (!parentId) return currentName || '';
    
    const parent = await this.locationDAO.getLocationById(parentId);
    return parent ? `${parent.hierarchy}/${currentName}` : currentName || '';
  }
}
```

#### LocationImportService
```typescript
class LocationImportService {
  constructor(
    private locationManagementService: LocationManagementService,
    private csvExtractor: CSVExtractor
  ) {}

  async importFromCSV(csvData: string): Promise<LocationImportResponse> {
    const results = {
      success: true,
      imported: 0,
      skipped: 0,
      errors: [] as string[]
    };

    try {
      const records = await this.csvExtractor.extractLocations(csvData);
      
      for (const record of records) {
        try {
          await this.importLocationRecord(record);
          results.imported++;
        } catch (error) {
          results.skipped++;
          results.errors.push(`Row ${record.rowNumber}: ${error.message}`);
        }
      }

      return {
        ...results,
        message: `Import completed: ${results.imported} imported, ${results.skipped} skipped`
      };
    } catch (error) {
      return {
        success: false,
        imported: 0,
        skipped: 0,
        errors: [error.message],
        message: 'Import failed'
      };
    }
  }

  private async importLocationRecord(record: LocationCSVRecord): Promise<void> {
    // Handle hierarchical import: Region → District → Ward → Street
    let parentId: string | undefined;

    // Create/find region
    if (record.region) {
      parentId = await this.ensureLocationExists({
        type: LocationType.REGION,
        name: record.region
      });
    }

    // Create/find district
    if (record.district && parentId) {
      parentId = await this.ensureLocationExists({
        type: LocationType.DISTRICT,
        name: record.district,
        parent: parentId
      });
    }

    // Create/find ward
    if (record.ward && parentId) {
      parentId = await this.ensureLocationExists({
        type: LocationType.WARD,
        name: record.ward,
        parent: parentId
      });
    }

    // Create street
    if (record.street && parentId) {
      await this.ensureLocationExists({
        type: LocationType.STREET,
        name: record.street,
        parent: parentId,
        coordinates: record.coordinates
      });
    }
  }

  private async ensureLocationExists(input: CreateLocationInput): Promise<string> {
    // Check if location already exists
    const existing = await this.locationDAO.findByNameAndParent(input.name, input.parent);
    if (existing) {
      return existing.locationId;
    }

    // Create new location
    const result = await this.locationManagementService.createLocation(input);
    return result.location.id;
  }
}
```

#### LocationJsonService
```typescript
class LocationJsonService {
  constructor(
    private locationDAO: LocationDAO,
    private s3Service: S3Service,
    private cloudfrontService: CloudFrontService
  ) {}

  async regenerateLocationJson(): Promise<LocationJsonResponse> {
    try {
      // 1. Build complete location tree
      const locationTree = await this.buildLocationTree();

      // 2. Generate optimized JSON structure
      const locationJson = {
        version: Date.now(),
        lastUpdated: new Date().toISOString(),
        data: locationTree,
        metadata: {
          totalRegions: locationTree.regions.length,
          totalDistricts: Object.values(locationTree.districts).flat().length,
          totalWards: Object.values(locationTree.wards).flat().length,
          totalStreets: Object.values(locationTree.streets).flat().length
        }
      };

      // 3. Upload to S3
      const s3Key = 'public/locations.json';
      await this.s3Service.uploadJson(s3Key, locationJson);

      // 4. Invalidate CloudFront cache
      await this.cloudfrontService.invalidate(['/locations.json']);

      // 5. Return CDN URL
      const cloudfrontUrl = `${process.env.CLOUDFRONT_DOMAIN}/locations.json`;

      return {
        success: true,
        cloudfrontUrl,
        message: 'Location JSON regenerated successfully'
      };
    } catch (error) {
      return {
        success: false,
        cloudfrontUrl: '',
        message: `Failed to regenerate location JSON: ${error.message}`
      };
    }
  }

  private async buildLocationTree(): Promise<LocationTree> {
    // Get all locations
    const allLocations = await this.locationDAO.getAllLocations();

    // Group by type and parent
    const regions = allLocations.filter(l => l.type === LocationType.REGION) as Region[];
    const districts: { [regionId: string]: District[] } = {};
    const wards: { [districtId: string]: Ward[] } = {};
    const streets: { [wardId: string]: Street[] } = {};

    // Group districts by region
    allLocations
      .filter(l => l.type === LocationType.DISTRICT)
      .forEach(district => {
        const regionId = district.parentId!;
        if (!districts[regionId]) districts[regionId] = [];
        districts[regionId].push(district as District);
      });

    // Group wards by district
    allLocations
      .filter(l => l.type === LocationType.WARD)
      .forEach(ward => {
        const districtId = ward.parentId!;
        if (!wards[districtId]) wards[districtId] = [];
        wards[districtId].push(ward as Ward);
      });

    // Group streets by ward
    allLocations
      .filter(l => l.type === LocationType.STREET)
      .forEach(street => {
        const wardId = street.parentId!;
        if (!streets[wardId]) streets[wardId] = [];
        streets[wardId].push(street as Street);
      });

    return { regions, districts, wards, streets };
  }
}
```

## Data Access Layer (DAO)

### LocationDAO
```typescript
class LocationDAO extends BaseDAO {
  async createLocation(location: Location): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.locationsTableName,
      Item: marshall(location),
      ConditionExpression: 'attribute_not_exists(locationId)'
    });
  }

  async getLocationById(locationId: string): Promise<Location | null> {
    const result = await this.dynamoClient.getItem({
      TableName: this.locationsTableName,
      Key: marshall({ locationId })
    });
    return result.Item ? unmarshall(result.Item) as Location : null;
  }

  async getLocationsByType(type: LocationType): Promise<Location[]> {
    const result = await this.dynamoClient.query({
      TableName: this.locationsTableName,
      IndexName: 'type-name-index',
      KeyConditionExpression: '#type = :type',
      ExpressionAttributeNames: { '#type': 'type' },
      ExpressionAttributeValues: marshall({ ':type': type })
    });

    return result.Items?.map(item => unmarshall(item) as Location) || [];
  }

  async getLocationsByParent(parentId: string): Promise<Location[]> {
    const result = await this.dynamoClient.query({
      TableName: this.locationsTableName,
      IndexName: 'parentId-name-index',
      KeyConditionExpression: 'parentId = :parentId',
      ExpressionAttributeValues: marshall({ ':parentId': parentId })
    });

    return result.Items?.map(item => unmarshall(item) as Location) || [];
  }

  async findByNameAndParent(name: string, parentId?: string): Promise<Location | null> {
    const result = await this.dynamoClient.query({
      TableName: this.locationsTableName,
      IndexName: 'name-parentId-index',
      KeyConditionExpression: '#name = :name',
      FilterExpression: parentId ? 'parentId = :parentId' : 'attribute_not_exists(parentId)',
      ExpressionAttributeNames: { '#name': 'name' },
      ExpressionAttributeValues: marshall(
        parentId ? { ':name': name, ':parentId': parentId } : { ':name': name }
      )
    });

    return result.Items?.[0] ? unmarshall(result.Items[0]) as Location : null;
  }

  async getAllLocations(): Promise<Location[]> {
    const result = await this.dynamoClient.scan({
      TableName: this.locationsTableName
    });

    return result.Items?.map(item => unmarshall(item) as Location) || [];
  }

  async updateLocation(locationId: string, updates: Partial<Location>): Promise<void> {
    const updateExpression = Object.keys(updates)
      .map(key => `#${key} = :${key}`)
      .join(', ');

    const expressionAttributeNames = Object.keys(updates)
      .reduce((acc, key) => ({ ...acc, [`#${key}`]: key }), {});

    const expressionAttributeValues = marshall(
      Object.entries(updates).reduce((acc, [key, value]) => ({ ...acc, [`:${key}`]: value }), {})
    );

    await this.dynamoClient.updateItem({
      TableName: this.locationsTableName,
      Key: marshall({ locationId }),
      UpdateExpression: `SET ${updateExpression}`,
      ExpressionAttributeNames: expressionAttributeNames,
      ExpressionAttributeValues: expressionAttributeValues
    });
  }
}
```

### LocationEventsDAO
```typescript
class LocationEventsDAO extends BaseDAO {
  async createLocationEvent(event: LocationEvent): Promise<void> {
    await this.dynamoClient.putItem({
      TableName: this.locationEventsTableName,
      Item: marshall(event)
    });
  }

  async getLocationEvents(locationId: string): Promise<LocationEvent[]> {
    const result = await this.dynamoClient.query({
      TableName: this.locationEventsTableName,
      KeyConditionExpression: 'locationId = :locationId',
      ExpressionAttributeValues: marshall({ ':locationId': locationId }),
      ScanIndexForward: true // Chronological order
    });

    return result.Items?.map(item => unmarshall(item) as LocationEvent) || [];
  }
}
```

## Event Sourcing Implementation

### Location Events
```typescript
enum LocationEventType {
  LOCATION_CREATED = 'LOCATION_CREATED',
  LOCATION_UPDATED = 'LOCATION_UPDATED',
  LOCATION_DELETED = 'LOCATION_DELETED',
  HIERARCHY_CHANGED = 'HIERARCHY_CHANGED',
  COORDINATES_UPDATED = 'COORDINATES_UPDATED',
  METADATA_UPDATED = 'METADATA_UPDATED'
}

interface LocationEvent {
  eventId: string
  locationId: string
  eventType: LocationEventType
  eventData: any
  timestamp: string
  version: number
  userId?: string // Who made the change
}
```

### Location Materialization
```typescript
class LocationMaterializationService {
  async materializeLocationFromEvents(locationId: string): Promise<Location> {
    const events = await this.locationEventsDAO.getLocationEvents(locationId);
    let location: Partial<Location> = {};

    for (const event of events) {
      location = this.applyEvent(location, event);
    }

    return location as Location;
  }

  private applyEvent(location: Partial<Location>, event: LocationEvent): Partial<Location> {
    switch (event.eventType) {
      case LocationEventType.LOCATION_CREATED:
        return {
          ...event.eventData,
          createdAt: event.timestamp,
          updatedAt: event.timestamp,
          version: event.version
        };
      
      case LocationEventType.LOCATION_UPDATED:
        return {
          ...location,
          ...event.eventData,
          updatedAt: event.timestamp,
          version: event.version
        };
      
      case LocationEventType.COORDINATES_UPDATED:
        return {
          ...location,
          coordinates: event.eventData.coordinates,
          updatedAt: event.timestamp,
          version: event.version
        };
      
      default:
        return location;
    }
  }
}
```

## Batch Processing Implementation

### CSV Import Structure
```typescript
interface LocationCSVRecord {
  rowNumber: number
  region: string
  district?: string
  ward?: string
  street?: string
  postalCode?: string
  latitude?: number
  longitude?: number
  population?: number
  area?: number
}
```

### Batch Job Handler
```typescript
// nest/src/handlers/batch-job.ts
export const batchJobHandler = async (event: S3Event): Promise<void> => {
  for (const record of event.Records) {
    if (record.s3.object.key.startsWith('locations/')) {
      await processLocationCSV(record.s3.bucket.name, record.s3.object.key);
    }
  }
};

async function processLocationCSV(bucketName: string, objectKey: string): Promise<void> {
  // 1. Download CSV from S3
  const csvContent = await s3Service.getObject(bucketName, objectKey);
  
  // 2. Process with LocationImportService
  const importService = new LocationImportService();
  const result = await importService.importFromCSV(csvContent);
  
  // 3. Log results
  console.log('Location import completed:', result);
  
  // 4. Notify admins of results
  await notificationService.notifyAdmins('Location Import Completed', result);
}
```

## Performance Optimizations

### Database Indexes
```typescript
const locationIndexes = {
  // Primary access patterns
  'type-name-index': {
    partitionKey: 'type',
    sortKey: 'name'
  },
  
  // Hierarchical queries
  'parentId-name-index': {
    partitionKey: 'parentId',
    sortKey: 'name'
  },
  
  // Name-based lookups
  'name-parentId-index': {
    partitionKey: 'name',
    sortKey: 'parentId'
  },
  
  // Level-based queries
  'level-hierarchy-index': {
    partitionKey: 'level',
    sortKey: 'hierarchy'
  }
};
```

### Caching Strategy
```typescript
class LocationCacheService {
  private cache = new Map<string, any>();
  private readonly TTL = {
    regions: 3600,      // 1 hour
    districts: 1800,    // 30 minutes
    wards: 900,         // 15 minutes
    streets: 600        // 10 minutes
  };

  async get(key: string): Promise<any> {
    const cached = this.cache.get(key);
    if (cached && cached.expiry > Date.now()) {
      return cached.data;
    }
    return null;
  }

  async set(key: string, data: any, ttl: number): Promise<void> {
    this.cache.set(key, {
      data,
      expiry: Date.now() + (ttl * 1000)
    });
  }

  invalidateLocationCache(locationId: string, type: LocationType): void {
    // Invalidate related cache entries
    const patterns = [
      `locations:${type.toLowerCase()}s`,
      `locations:${type.toLowerCase()}:${locationId}`,
      'locations:tree'
    ];
    
    patterns.forEach(pattern => {
      for (const key of this.cache.keys()) {
        if (key.includes(pattern)) {
          this.cache.delete(key);
        }
      }
    });
  }
}
```

### Location JSON Optimization
```typescript
// Optimized JSON structure for client consumption
interface OptimizedLocationTree {
  version: number
  lastUpdated: string
  regions: {
    [regionId: string]: {
      name: string
      districts: {
        [districtId: string]: {
          name: string
          wards: {
            [wardId: string]: {
              name: string
              streets: {
                [streetId: string]: {
                  name: string
                  coordinates?: Coordinates
                }
              }
            }
          }
        }
      }
    }
  }
}
```

## Validation & Error Handling

### Location Validation
```typescript
class LocationValidationService {
  async validateCreateInput(input: CreateLocationInput): Promise<void> {
    // Validate location type hierarchy
    if (input.type !== LocationType.REGION && !input.parent) {
      throw new Error(`${input.type} requires a parent location`);
    }

    // Validate parent relationship
    if (input.parent) {
      const parent = await this.locationDAO.getLocationById(input.parent);
      if (!parent) {
        throw new Error('Parent location not found');
      }

      const validParentTypes = this.getValidParentTypes(input.type);
      if (!validParentTypes.includes(parent.type)) {
        throw new Error(`Invalid parent type for ${input.type}`);
      }
    }

    // Validate location name
    await this.validateLocationName(input.name, input.type);
  }

  private getValidParentTypes(type: LocationType): LocationType[] {
    const hierarchy = {
      [LocationType.DISTRICT]: [LocationType.REGION],
      [LocationType.WARD]: [LocationType.DISTRICT],
      [LocationType.STREET]: [LocationType.WARD]
    };
    return hierarchy[type] || [];
  }

  async validateLocationName(name: string, type: LocationType): Promise<void> {
    if (!name || name.trim().length < 2) {
      throw new Error('Location name must be at least 2 characters');
    }

    if (name.length > 100) {
      throw new Error('Location name must be less than 100 characters');
    }

    // Check for invalid characters
    const invalidChars = /[<>\"'&]/;
    if (invalidChars.test(name)) {
      throw new Error('Location name contains invalid characters');
    }
  }
}
```

### Error Types
```typescript
enum LocationErrorType {
  LOCATION_NOT_FOUND = 'LOCATION_NOT_FOUND',
  DUPLICATE_LOCATION = 'DUPLICATE_LOCATION',
  INVALID_HIERARCHY = 'INVALID_HIERARCHY',
  INVALID_PARENT = 'INVALID_PARENT',
  CSV_PARSE_ERROR = 'CSV_PARSE_ERROR',
  IMPORT_VALIDATION_ERROR = 'IMPORT_VALIDATION_ERROR'
}
```

## Monitoring & Analytics

### Key Metrics
- Location query performance
- Cache hit rates
- CSV import success rates
- Location JSON generation time
- Hierarchy depth distribution

### CloudWatch Alarms
- Location query latency >500ms
- Cache miss rate >20%
- Import failure rate >5%
- JSON generation failures