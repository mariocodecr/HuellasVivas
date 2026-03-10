# Implementation Plan: Categories API Endpoint

## Overview

This implementation plan breaks down the Categories API endpoint feature into discrete coding tasks. The endpoint exposes a public REST API to retrieve the four static animal categories (DOG, CAT, RABBIT, OTHER) from the database. The implementation follows NestJS module patterns with a three-layer architecture: Controller → Service → Repository.

## Tasks

- [x] 1. Create module structure and core entities
  - Create `src/modules/categories` directory
  - Create `entities/category.entity.ts` with Category interface (id, slug, label fields)
  - Create `dto/category-response.dto.ts` with CategoryResponseDto class using @ApiProperty and @Expose decorators
  - _Requirements: 3.1, 3.2, 3.3_

- [x] 2. Implement @Public() decorator
  - Create `src/common/decorators/public.decorator.ts` file
  - Implement Public decorator using SetMetadata with IS_PUBLIC_KEY constant
  - Export both IS_PUBLIC_KEY and Public decorator
  - _Requirements: 1.5_

- [x] 3. Implement CategoriesRepository
  - [x] 3.1 Create categories.repository.ts with database query logic
    - Inject SupabaseService in constructor
    - Implement findAll() method that queries categories table
    - Select id, slug, label fields and order by slug ascending
    - Implement mapRow() private method to map database rows to Category entities
    - Add error logging for database failures
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_
  
  - [ ]* 3.2 Write unit tests for CategoriesRepository
    - Test that findAll queries correct table with correct fields
    - Test that results are ordered by slug ascending
    - Test that database rows are mapped to entities correctly
    - Test that database errors are logged and thrown
    - Test that empty results return empty array
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_
  
  - [ ]* 3.3 Write property test for categories ordering
    - **Property 1: Categories ordering**
    - **Validates: Requirements 2.4**
    - Generate random category arrays with different slugs
    - Verify returned categories are always ordered by slug ascending
    - Use @fast-check/jest with 100 iterations
    - _Requirements: 2.4_

- [x] 4. Implement CategoriesService
  - [x] 4.1 Create categories.service.ts with business logic
    - Inject CategoriesRepository in constructor
    - Implement findAll() method that delegates to repository
    - Transform Category entities to CategoryResponseDto using plainToInstance
    - Add logging for operations
    - Use excludeExtraneousValues option in transformation
    - _Requirements: 2.1, 4.1, 4.2_
  
  - [ ]* 4.2 Write unit tests for CategoriesService
    - Test that findAll returns all categories from repository
    - Test that entities are transformed to DTOs correctly with all fields (id, slug, label)
    - Test that empty repository results return empty array
    - Test that repository errors propagate through service
    - Mock repository dependencies
    - Achieve 100% code coverage
    - _Requirements: 4.1, 4.2, 4.3, 4.4_
  
  - [ ]* 4.3 Write property test for category structure completeness
    - **Property 2: Category structure completeness**
    - **Validates: Requirements 3.1, 3.2, 3.3**
    - Generate random category arrays with various field values
    - Verify all returned DTOs have id, slug, and label fields
    - Verify all fields are non-empty strings
    - Use @fast-check/jest with 100 iterations
    - _Requirements: 3.1, 3.2, 3.3_

- [x] 5. Implement CategoriesController
  - [x] 5.1 Create categories.controller.ts with HTTP endpoint
    - Inject CategoriesService in constructor
    - Implement GET findAll() method that delegates to service
    - Apply @Public() decorator to bypass authentication
    - Add @ApiTags('categories') for Swagger grouping
    - Add @ApiOperation with summary 'List all animal categories'
    - Add @ApiResponse decorator documenting 200 response with CategoryResponseDto array
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 5.1, 5.2, 5.3_
  
  - [ ]* 5.2 Write unit tests for CategoriesController
    - Test that findAll calls service.findAll()
    - Test that controller returns service result
    - Mock service dependencies
    - _Requirements: 1.1, 1.2_

- [x] 6. Create and register CategoriesModule
  - Create categories.module.ts with @Module decorator
  - Import DatabaseModule for Supabase access
  - Register CategoriesController in controllers array
  - Register CategoriesService and CategoriesRepository in providers array
  - Export CategoriesService for potential use by other modules
  - _Requirements: 1.1, 2.1_

- [x] 7. Register CategoriesModule in AppModule
  - Add CategoriesModule to imports array in app.module.ts
  - Verify module is properly wired into application
  - _Requirements: 1.1_

- [x] 8. Checkpoint - Ensure all tests pass
  - Run unit tests with `npm run test`
  - Run test coverage with `npm run test:cov`
  - Verify 100% coverage for service and repository layers
  - Ensure all property-based tests pass (100 iterations each)
  - Ask the user if questions arise

- [ ]* 9. Write E2E tests for categories endpoint
  - [ ]* 9.1 Create test/categories.e2e-spec.ts
    - Test GET /api/v1/categories returns 200 without authentication
    - Test response has standard envelope structure with data and meta fields
    - Test data array contains four categories
    - Test each category has id, slug, and label fields
    - Test categories are ordered by slug: CAT, DOG, OTHER, RABBIT
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 2.4, 3.1, 3.2, 3.3, 3.4_
  
  - [ ]* 9.2 Run E2E tests
    - Execute `npm run test:e2e`
    - Verify all E2E tests pass
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ]* 10. Verify Swagger documentation
  - Start application and access Swagger UI at /api/docs
  - Verify categories endpoint appears under 'categories' tag
  - Verify endpoint shows summary 'List all animal categories'
  - Verify endpoint is marked as public (no lock icon)
  - Verify response schema shows CategoryResponseDto structure with id, slug, label fields
  - Verify slug field shows enum values: DOG, CAT, RABBIT, OTHER
  - _Requirements: 5.1, 5.2, 5.3, 5.4_

- [x] 11. Final checkpoint - Verify complete functionality
  - Ensure all unit tests pass
  - Ensure all property-based tests pass
  - Verify endpoint is accessible without authentication
  - Verify response follows standard envelope format
  - Ask the user if questions arise

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties using @fast-check/jest
- Unit tests validate specific examples and edge cases
- E2E tests validate the complete request-response flow
- The @Public() decorator must be implemented before the controller to avoid compilation errors
- The ResponseInterceptor (already implemented globally) will wrap responses in the standard envelope
