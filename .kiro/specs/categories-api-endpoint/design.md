# Design Document: Categories API Endpoint

## Overview

The Categories API endpoint is a simple read-only REST endpoint that exposes the four static animal categories (DOG, CAT, RABBIT, OTHER) seeded in the database. This endpoint follows NestJS module patterns with a three-layer architecture: Controller → Service → Repository.

The endpoint is public (no authentication required) since categories are static reference data that never change at runtime. The response follows the standard envelope format enforced by the global ResponseInterceptor.

**Key Design Decisions:**
- No caching layer needed - categories are static and the query is lightweight
- No pagination needed - only four categories exist
- No filtering or search - clients receive all categories
- Repository handles snake_case to camelCase mapping
- Service layer transforms entities to DTOs using class-transformer

## Architecture

The feature follows the standard NestJS module architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                     CategoriesModule                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────┐                                        │
│  │   Controller     │  HTTP Layer                            │
│  │  GET /categories │  - Route handling                      │
│  │  @Public()       │  - Swagger docs                        │
│  └────────┬─────────┘                                        │
│           │                                                   │
│           ▼                                                   │
│  ┌──────────────────┐                                        │
│  │    Service       │  Business Logic                        │
│  │  findAll()       │  - DTO transformation                  │
│  └────────┬─────────┘  - Orchestration                       │
│           │                                                   │
│           ▼                                                   │
│  ┌──────────────────┐                                        │
│  │   Repository     │  Data Access                           │
│  │  findAll()       │  - Supabase queries                    │
│  └────────┬─────────┘  - Entity mapping                      │
│           │                                                   │
└───────────┼─────────────────────────────────────────────────┘
            │
            ▼
    ┌───────────────┐
    │   Supabase    │
    │  categories   │
    │     table     │
    └───────────────┘
```

**Module Dependencies:**
- `DatabaseModule` - provides SupabaseService for database access
- `ConfigModule` - global configuration (already imported in AppModule)

**Data Flow:**
1. Client sends GET request to `/api/v1/categories`
2. Controller receives request (no auth guard)
3. Controller calls `categoriesService.findAll()`
4. Service calls `categoriesRepository.findAll()`
5. Repository queries Supabase `categories` table
6. Repository maps database rows to Category entities
7. Service transforms entities to CategoryResponseDto
8. Controller returns DTOs
9. ResponseInterceptor wraps response in standard envelope

## Components and Interfaces

### 1. CategoriesModule

**File:** `src/modules/categories/categories.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { CategoriesController } from './categories.controller';
import { CategoriesService } from './categories.service';
import { CategoriesRepository } from './categories.repository';
import { DatabaseModule } from '../../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [CategoriesController],
  providers: [CategoriesService, CategoriesRepository],
  exports: [CategoriesService],
})
export class CategoriesModule {}
```

**Responsibilities:**
- Register controller, service, and repository
- Import DatabaseModule for Supabase access
- Export service for potential use by other modules

### 2. CategoriesController

**File:** `src/modules/categories/categories.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { CategoriesService } from './categories.service';
import { CategoryResponseDto } from './dto/category-response.dto';
import { Public } from '../../common/decorators/public.decorator';

@ApiTags('categories')
@Controller('categories')
export class CategoriesController {
  constructor(private readonly categoriesService: CategoriesService) {}

  @Get()
  @Public()
  @ApiOperation({ summary: 'List all animal categories' })
  @ApiResponse({
    status: 200,
    description: 'Returns all animal categories',
    type: [CategoryResponseDto],
  })
  async findAll(): Promise<CategoryResponseDto[]> {
    return this.categoriesService.findAll();
  }
}
```

**Responsibilities:**
- Expose GET endpoint at `/api/v1/categories`
- Apply `@Public()` decorator to bypass authentication
- Document endpoint with Swagger decorators
- Delegate to service layer

**Decorators:**
- `@ApiTags('categories')` - groups endpoint in Swagger UI
- `@Public()` - marks endpoint as public (no JWT required)
- `@ApiOperation()` - provides endpoint description
- `@ApiResponse()` - documents response schema

**Note:** The `@Public()` decorator must be created as a custom decorator that sets metadata to bypass the JWT auth guard.

### 3. CategoriesService

**File:** `src/modules/categories/categories.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { plainToInstance } from 'class-transformer';
import { CategoriesRepository } from './categories.repository';
import { CategoryResponseDto } from './dto/category-response.dto';

@Injectable()
export class CategoriesService {
  private readonly logger = new Logger(CategoriesService.name);

  constructor(private readonly categoriesRepository: CategoriesRepository) {}

  async findAll(): Promise<CategoryResponseDto[]> {
    this.logger.log('Fetching all categories');
    const categories = await this.categoriesRepository.findAll();
    return plainToInstance(CategoryResponseDto, categories, {
      excludeExtraneousValues: true,
    });
  }
}
```

**Responsibilities:**
- Orchestrate category retrieval
- Transform entities to DTOs using class-transformer
- Log operations for observability
- No business logic needed (simple pass-through)

**Error Handling:**
- No custom error handling needed
- Repository errors propagate to global exception filter
- Empty result is valid (returns empty array)

### 4. CategoriesRepository

**File:** `src/modules/categories/categories.repository.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { SupabaseService } from '../../database/supabase.service';
import { Category } from './entities/category.entity';

@Injectable()
export class CategoriesRepository {
  private readonly logger = new Logger(CategoriesRepository.name);

  constructor(private readonly supabase: SupabaseService) {}

  async findAll(): Promise<Category[]> {
    const { data, error } = await this.supabase.client
      .from('categories')
      .select('id, slug, label')
      .order('slug', { ascending: true });

    if (error) {
      this.logger.error('Failed to fetch categories', error);
      throw error;
    }

    return (data || []).map(this.mapRow);
  }

  private mapRow(row: Record<string, unknown>): Category {
    return {
      id: row['id'] as string,
      slug: row['slug'] as string,
      label: row['label'] as string,
    };
  }
}
```

**Responsibilities:**
- Query Supabase `categories` table
- Select only required fields (id, slug, label)
- Order results by slug ascending
- Map database rows to Category entities
- Handle database errors

**Query Specification:**
- Table: `categories`
- Fields: `id`, `slug`, `label`
- Order: `slug ASC`
- No filtering (return all rows)

### 5. Public Decorator

**File:** `src/common/decorators/public.decorator.ts`

```typescript
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

**Responsibilities:**
- Mark endpoints as public (no authentication required)
- Set metadata that JWT auth guard checks

**Note:** The JWT auth guard must be updated to check for this metadata and skip authentication when present.

## Data Models

### Category Entity

**File:** `src/modules/categories/entities/category.entity.ts`

```typescript
export interface Category {
  id: string;
  slug: string;
  label: string;
}
```

**Field Specifications:**
- `id`: UUID string (primary key from database)
- `slug`: Uppercase string enum value (DOG, CAT, RABBIT, OTHER)
- `label`: Human-readable Spanish label (Perro, Gato, Conejo, Otros)

**Database Mapping:**
- Maps directly from `categories` table columns
- No snake_case conversion needed (all columns are lowercase)

### CategoryResponseDto

**File:** `src/modules/categories/dto/category-response.dto.ts`

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { Expose } from 'class-transformer';

export class CategoryResponseDto {
  @ApiProperty({
    description: 'Unique category identifier',
    example: '550e8400-e29b-41d4-a716-446655440000',
  })
  @Expose()
  id: string;

  @ApiProperty({
    description: 'Category slug (uppercase)',
    example: 'DOG',
    enum: ['DOG', 'CAT', 'RABBIT', 'OTHER'],
  })
  @Expose()
  slug: string;

  @ApiProperty({
    description: 'Human-readable category label',
    example: 'Perro',
  })
  @Expose()
  label: string;
}
```

**Responsibilities:**
- Define response structure for API clients
- Document fields in Swagger
- Control serialization with `@Expose()` decorator

**Swagger Documentation:**
- Each field has description and example
- Slug field documents valid enum values
- Used in controller `@ApiResponse()` decorator

### Response Envelope

The global `ResponseInterceptor` wraps the DTO array in a standard envelope:

```typescript
{
  "data": [
    {
      "id": "uuid-1",
      "slug": "CAT",
      "label": "Gato"
    },
    {
      "id": "uuid-2",
      "slug": "DOG",
      "label": "Perro"
    },
    {
      "id": "uuid-3",
      "slug": "OTHER",
      "label": "Otros"
    },
    {
      "id": "uuid-4",
      "slug": "RABBIT",
      "label": "Conejo"
    }
  ],
  "meta": {
    "timestamp": "2025-01-15T10:30:00.000Z"
  }
}
```

**Note:** The ordering shown above reflects the `slug ASC` ordering (alphabetical).

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Categories Ordering

*For any* set of categories returned by the repository, the results should be ordered by slug in ascending alphabetical order.

**Validates: Requirements 2.4**

**Rationale:** Consistent ordering ensures predictable UI rendering and makes the API response deterministic. Alphabetical ordering by slug (CAT, DOG, OTHER, RABBIT) provides a natural sort order.

**Test Strategy:** Generate random category data with different slugs, call the repository method, and verify the returned array is sorted by slug ascending.

### Property 2: Category Structure Completeness

*For all* categories returned by the API, each category must include an `id` field (UUID string), a `slug` field (non-empty string), and a `label` field (non-empty string).

**Validates: Requirements 3.1, 3.2, 3.3**

**Rationale:** Clients depend on all three fields being present. The `id` uniquely identifies the category, the `slug` is used for filtering publications, and the `label` is displayed in the UI.

**Test Strategy:** Call the service method and verify each returned DTO has all three fields with correct types and non-empty values.

## Error Handling

### Repository Layer Errors

**Supabase Query Failure:**
- **Cause:** Database connection error, network timeout, or query syntax error
- **Handling:** Repository logs error and throws it
- **Response:** Global exception filter catches and returns 500 Internal Server Error

```typescript
if (error) {
  this.logger.error('Failed to fetch categories', error);
  throw error;
}
```

**Empty Result:**
- **Cause:** Categories table is empty (should never happen with migrations)
- **Handling:** Repository returns empty array
- **Response:** Service returns empty array, wrapped in standard envelope

### Service Layer Errors

No custom error handling needed. The service is a simple pass-through that transforms entities to DTOs. Any repository errors propagate to the global exception filter.

### Controller Layer Errors

No custom error handling needed. The controller delegates to the service and returns the result. The global exception filter handles all errors.

### Error Response Format

All errors follow the standard envelope format:

```json
{
  "error": {
    "statusCode": 500,
    "code": "INTERNAL_SERVER_ERROR",
    "message": "Failed to fetch categories",
    "details": null,
    "timestamp": "2025-01-15T10:30:00.000Z",
    "path": "/api/v1/categories"
  }
}
```

## Testing Strategy

### Unit Tests

**CategoriesService Tests:**

File: `src/modules/categories/categories.service.spec.ts`

Test cases:
1. **Should return all categories from repository**
   - Mock repository to return array of Category entities
   - Call `service.findAll()`
   - Verify service returns CategoryResponseDto array
   - Verify all fields are mapped correctly (id, slug, label)

2. **Should transform entities to DTOs**
   - Mock repository with specific category data
   - Call `service.findAll()`
   - Verify returned DTOs only contain exposed fields
   - Verify class-transformer excludes extraneous values

3. **Should handle empty repository result**
   - Mock repository to return empty array
   - Call `service.findAll()`
   - Verify service returns empty array (not null or undefined)

4. **Should propagate repository errors**
   - Mock repository to throw error
   - Call `service.findAll()`
   - Verify error propagates (not caught by service)

**CategoriesRepository Tests:**

File: `src/modules/categories/categories.repository.spec.ts`

Test cases:
1. **Should query categories table with correct fields**
   - Mock Supabase client
   - Call `repository.findAll()`
   - Verify query selects id, slug, label
   - Verify query orders by slug ascending

2. **Should map database rows to entities**
   - Mock Supabase to return raw database rows
   - Call `repository.findAll()`
   - Verify returned entities have correct structure
   - Verify all fields are mapped correctly

3. **Should handle database errors**
   - Mock Supabase to return error
   - Call `repository.findAll()`
   - Verify error is logged
   - Verify error is thrown

4. **Should handle empty result**
   - Mock Supabase to return empty data array
   - Call `repository.findAll()`
   - Verify repository returns empty array

**CategoriesController Tests:**

File: `src/modules/categories/categories.controller.spec.ts`

Test cases:
1. **Should call service.findAll()**
   - Mock service
   - Call `controller.findAll()`
   - Verify service method is called

2. **Should return service result**
   - Mock service to return DTO array
   - Call `controller.findAll()`
   - Verify controller returns same array

**Coverage Target:** 100% code coverage for service and repository layers

### Property-Based Tests

**Property Test Configuration:**
- Library: `@fast-check/jest` (property-based testing for TypeScript)
- Iterations: 100 per property test
- Tag format: `Feature: categories-api-endpoint, Property {number}: {property_text}`

**Property Test 1: Categories Ordering**

File: `src/modules/categories/categories.repository.spec.ts`

```typescript
// Feature: categories-api-endpoint, Property 1: Categories ordering
it('should always return categories ordered by slug ascending', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.array(fc.record({
        id: fc.uuid(),
        slug: fc.constantFrom('DOG', 'CAT', 'RABBIT', 'OTHER'),
        label: fc.string({ minLength: 1 }),
      }), { minLength: 1, maxLength: 10 }),
      async (categories) => {
        // Mock Supabase to return unordered categories
        mockSupabaseClient.from.mockReturnValue({
          select: jest.fn().mockReturnValue({
            order: jest.fn().mockResolvedValue({
              data: categories,
              error: null,
            }),
          }),
        });

        const result = await repository.findAll();
        
        // Verify ordering
        for (let i = 1; i < result.length; i++) {
          expect(result[i - 1].slug <= result[i].slug).toBe(true);
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

**Property Test 2: Category Structure Completeness**

File: `src/modules/categories/categories.service.spec.ts`

```typescript
// Feature: categories-api-endpoint, Property 2: Category structure completeness
it('should always return categories with id, slug, and label fields', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.array(fc.record({
        id: fc.uuid(),
        slug: fc.constantFrom('DOG', 'CAT', 'RABBIT', 'OTHER'),
        label: fc.string({ minLength: 1 }),
      }), { minLength: 1, maxLength: 10 }),
      async (categories) => {
        // Mock repository
        mockRepository.findAll.mockResolvedValue(categories);

        const result = await service.findAll();
        
        // Verify all categories have required fields
        result.forEach(category => {
          expect(category.id).toBeDefined();
          expect(typeof category.id).toBe('string');
          expect(category.id.length).toBeGreaterThan(0);
          
          expect(category.slug).toBeDefined();
          expect(typeof category.slug).toBe('string');
          expect(category.slug.length).toBeGreaterThan(0);
          
          expect(category.label).toBeDefined();
          expect(typeof category.label).toBe('string');
          expect(category.label.length).toBeGreaterThan(0);
        });
      }
    ),
    { numRuns: 100 }
  );
});
```

### Integration Tests

**E2E Test:**

File: `test/categories.e2e-spec.ts`

Test cases:
1. **GET /api/v1/categories returns 200 and all categories**
   - Send GET request without auth token
   - Verify status 200
   - Verify response has standard envelope structure
   - Verify data array contains four categories
   - Verify each category has id, slug, label

2. **GET /api/v1/categories returns categories in correct order**
   - Send GET request
   - Verify categories are ordered: CAT, DOG, OTHER, RABBIT

3. **GET /api/v1/categories is accessible without authentication**
   - Send GET request without Authorization header
   - Verify status 200 (not 401)

4. **Swagger documentation includes categories endpoint**
   - Fetch Swagger JSON from /api/docs-json
   - Verify /api/v1/categories path exists
   - Verify endpoint is marked as public
   - Verify response schema matches CategoryResponseDto

### Test Execution

**Unit tests:**
```bash
npm run test
```

**E2E tests:**
```bash
npm run test:e2e
```

**Coverage report:**
```bash
npm run test:cov
```

**Property-based tests:**
- Run automatically with unit tests
- Each property test runs 100 iterations
- Failures include counterexample for debugging
