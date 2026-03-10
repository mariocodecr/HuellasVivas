# Requirements Document

## Introduction

This feature exposes a public API endpoint to retrieve the four animal categories (DOG, CAT, RABBIT, OTHER) that are seeded during database migrations. The endpoint enables the frontend to populate filter dropdowns and publication creation forms with the available category options. Since categories are static reference data that never change at runtime, this is a simple read-only endpoint with no authentication requirements.

## Glossary

- **Categories_API**: The REST API module that exposes animal category data
- **Category**: An animal classification entity with id, slug, and label fields
- **Categories_Repository**: The data access layer that queries the Supabase categories table
- **Categories_Service**: The business logic layer that orchestrates category retrieval
- **Categories_Controller**: The HTTP request handler that exposes the GET endpoint
- **Frontend**: The client application that consumes the categories endpoint
- **Supabase**: The PostgreSQL database where categories are stored

## Requirements

### Requirement 1: Expose Categories Endpoint

**User Story:** As a frontend developer, I want to retrieve all animal categories via a REST API, so that I can populate dropdown menus and form options.

#### Acceptance Criteria

1. THE Categories_Controller SHALL expose a GET endpoint at /api/v1/categories
2. WHEN a GET request is received at /api/v1/categories, THE Categories_API SHALL return HTTP status 200
3. WHEN a GET request is received at /api/v1/categories, THE Categories_API SHALL return all four categories (DOG, CAT, RABBIT, OTHER)
4. THE Categories_API SHALL wrap the response in a standard envelope with data and meta fields
5. THE Categories_API SHALL allow access to /api/v1/categories without authentication

### Requirement 2: Retrieve Categories from Database

**User Story:** As the Categories API, I want to query categories from the database, so that I can provide current category data to clients.

#### Acceptance Criteria

1. WHEN Categories_Service.findAll is invoked, THE Categories_Service SHALL delegate to Categories_Repository.findAll
2. WHEN Categories_Repository.findAll is invoked, THE Categories_Repository SHALL query the Supabase categories table
3. THE Categories_Repository SHALL select id, slug, and label fields from the categories table
4. THE Categories_Repository SHALL order results by slug in ascending order
5. THE Categories_Repository SHALL map database rows to Category entity objects

### Requirement 3: Return Structured Category Data

**User Story:** As a frontend developer, I want each category to include id, slug, and label, so that I can display user-friendly labels while submitting slug values.

#### Acceptance Criteria

1. FOR ALL categories returned, THE Categories_API SHALL include the id field as a UUID
2. FOR ALL categories returned, THE Categories_API SHALL include the slug field as a string
3. FOR ALL categories returned, THE Categories_API SHALL include the label field as a string
4. THE Categories_API SHALL return exactly four categories when the database contains the seeded data

### Requirement 4: Service Layer Testing

**User Story:** As a developer, I want comprehensive unit tests for the service layer, so that I can verify business logic correctness.

#### Acceptance Criteria

1. THE Categories_Service unit tests SHALL verify that findAll returns all categories from the repository
2. THE Categories_Service unit tests SHALL verify that findAll correctly maps repository results including slug, label, and id fields
3. THE Categories_Service unit tests SHALL use mocked repository dependencies
4. FOR ALL unit tests, THE test suite SHALL achieve 100% code coverage of the service layer

### Requirement 5: API Documentation

**User Story:** As an API consumer, I want the categories endpoint documented in Swagger, so that I can understand the endpoint contract.

#### Acceptance Criteria

1. THE Categories_Controller SHALL include @ApiTags('categories') decorator for Swagger grouping
2. THE Categories_Controller SHALL include @ApiOperation decorator with summary 'List all animal categories'
3. THE Swagger documentation SHALL indicate that /api/v1/categories is a public endpoint
4. THE Swagger documentation SHALL show the response schema with the standard data envelope

