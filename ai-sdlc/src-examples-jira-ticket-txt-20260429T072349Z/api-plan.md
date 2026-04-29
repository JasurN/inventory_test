# Node API Implementation Plan Template

## API Overview

The Notification Template Management API enables admin users to manage notification templates used for customer notifications across multiple channels (email, SMS, push). Admins can create, view, update, activate, and deactivate templates. Only active templates are eligible for sending notifications. The API enforces role-based access control, input validation, and business rules to maintain data integrity and security.

## REST Endpoints

| Method | Path                               | Purpose                          | Request Body                          | Response Body                          | Status Codes                   |
|--------|----------------------------------|---------------------------------|-------------------------------------|--------------------------------------|-------------------------------|
| GET    | /notification-templates          | List all notification templates | None                                | Array of NotificationTemplateDTO     | 200 OK, 401 Unauthorized      |
| POST   | /notification-templates          | Create a new notification template | CreateNotificationTemplateDTO       | NotificationTemplateDTO               | 201 Created, 400 Bad Request, 401 Unauthorized, 409 Conflict |
| PUT    | /notification-templates/{id}     | Update an existing template      | UpdateNotificationTemplateDTO       | NotificationTemplateDTO               | 200 OK, 400 Bad Request, 401 Unauthorized, 404 Not Found, 409 Conflict |
| PATCH  | /notification-templates/{id}/deactivate | Deactivate a template            | None                                | NotificationTemplateDTO               | 200 OK, 401 Unauthorized, 404 Not Found |

## DTOs

```typescript
// Request DTOs
type Channel = 'email' | 'sms' | 'push';
type Status = 'active' | 'inactive';

interface CreateNotificationTemplateDTO {
  name: string;
  channel: Channel;
  subject?: string; // Required if channel === 'email'
  body: string;
  status: Status;
}

interface UpdateNotificationTemplateDTO {
  name?: string;
  channel?: Channel;
  subject?: string; // Required if channel === 'email'
  body?: string;
  status?: Status;
}

// Response DTO
interface NotificationTemplateDTO {
  id: string;
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
  createdAt: string; // ISO date string
  updatedAt: string; // ISO date string
}
```

## Validation Rules

| Validation Rule                                     | Request Validation                          | Service Layer Enforcement                     |
|---------------------------------------------------|---------------------------------------------|-----------------------------------------------|
| `name`, `channel`, `body`, and `status` required | Validate presence and non-empty strings     | Confirm required fields before DB operations  |
| `subject` required if `channel` is `email`        | Conditional validation on `subject` field   | Enforce subject presence for email channel    |
| `status` must be `active` or `inactive`            | Enum validation                             | Confirm status value before update/create     |
| Only active templates can be used for sending      | N/A (outside this API scope)                 | Enforced in notification sending service      |

## Database Model

**NotificationTemplate Entity**

| Field     | Type       | Constraints                          |
|-----------|------------|------------------------------------|
| id        | UUID       | PK, generated                      |
| name      | string     | Required, indexed, unique per channel (optional) |
| channel   | string     | Required, enum: email, sms, push   |
| subject   | string     | Nullable, required if channel=email|
| body      | string     | Required                          |
| status    | string     | Required, enum: active, inactive  |
| createdAt | timestamp  | Auto-generated                    |
| updatedAt | timestamp  | Auto-updated                     |

**Indexes & Constraints**

- Unique index on (name, channel) to prevent duplicate template names per channel (optional, depending on business needs).
- Index on status for efficient filtering.

## Service Responsibilities

- **Controller**: Handle HTTP requests, parse and validate input DTOs, enforce authentication and authorization, call service methods, and return appropriate HTTP responses.
- **Service**: Implement business logic, validate business rules, coordinate repository calls, handle idempotency and concurrency concerns.
- **Repository**: Abstract database access, perform CRUD operations, manage transactions.
- **Mapper**: Convert between database entities and DTOs.

## Error Handling

| Error Type          | Cause                                    | Response Status | Response Body Example                          |
|---------------------|------------------------------------------|-----------------|-----------------------------------------------|
| Validation Error    | Missing or invalid fields                 | 400 Bad Request | `{ "error": "Validation failed", "details": [...] }` |
| Not Found           | Template ID does not exist                 | 404 Not Found   | `{ "error": "Notification template not found" }` |
| Conflict            | Duplicate template name/channel on create or update | 409 Conflict   | `{ "error": "Template with this name and channel already exists" }` |
| Unauthorized        | User not authenticated or not admin       | 401 Unauthorized| `{ "error": "Unauthorized" }`                  |
| Unexpected Error    | Unhandled exceptions                       | 500 Internal Server Error | `{ "error": "Internal server error" }`         |

## Security Considerations

- **Authentication & Authorization**: Only authenticated admin users can access these endpoints. Use JWT or session-based auth with role checks.
- **Input Validation**: Strict validation to prevent injection attacks.
- **Audit Logging**: Log create, update, and deactivate actions with user info and timestamps (consider for compliance).
- **Sensitive Data**: Template bodies may contain sensitive info; ensure secure storage and access control.
- **Idempotency**: POST and PUT operations should be idempotent where possible; consider idempotency keys if needed.

## Example Backend Stub Code

```typescript
// Controller (Express example)
import { Request, Response } from 'express';
import { NotificationTemplateService } from '../services/NotificationTemplateService';
import { CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO } from '../dtos';

export class NotificationTemplateController {
  constructor(private service: NotificationTemplateService) {}

  async list(req: Request, res: Response) {
    // Authorization check omitted for brevity
    const templates = await this.service.listAll();
    res.status(200).json(templates);
  }

  async create(req: Request, res: Response) {
    const dto: CreateNotificationTemplateDTO = req.body;
    // Validate dto here or via middleware
    try {
      const created = await this.service.create(dto);
      res.status(201).json(created);
    } catch (err) {
      // Handle validation, conflict, etc.
      res.status(400).json({ error: err.message });
    }
  }

  async update(req: Request, res: Response) {
    const id = req.params.id;
    const dto: UpdateNotificationTemplateDTO = req.body;
    try {
      const updated = await this.service.update(id, dto);
      res.status(200).json(updated);
    } catch (err) {
      if (err.message === 'NotFound') {
        res.status(404).json({ error: 'Notification template not found' });
      } else {
        res.status(400).json({ error: err.message });
      }
    }
  }

  async deactivate(req: Request, res: Response) {
    const id = req.params.id;
    try {
      const deactivated = await this.service.deactivate(id);
      res.status(200).json(deactivated);
    } catch (err) {
      if (err.message === 'NotFound') {
        res.status(404).json({ error: 'Notification template not found' });
      } else {
        res.status(400).json({ error: err.message });
      }
    }
  }
}

// Service
export class NotificationTemplateService {
  constructor(private repository: NotificationTemplateRepository) {}

  async listAll() {
    return this.repository.findAll();
  }

  async create(dto: CreateNotificationTemplateDTO) {
    this.validateDTO(dto);
    // Check for duplicates
    const exists = await this.repository.findByNameAndChannel(dto.name, dto.channel);
    if (exists) throw new Error('Template with this name and channel already exists');
    const entity = this.repository.toEntity(dto);
    return this.repository.create(entity);
  }

  async update(id: string, dto: UpdateNotificationTemplateDTO) {
    const existing = await this.repository.findById(id);
    if (!existing) throw new Error('NotFound');
    const updatedEntity = this.repository.merge(existing, dto);
    this.validateDTO(updatedEntity);
    return this.repository.update(updatedEntity);
  }

  async deactivate(id: string) {
    const existing = await this.repository.findById(id);
    if (!existing) throw new Error('NotFound');
    existing.status = 'inactive';
    return this.repository.update(existing);
  }

  private validateDTO(dto: CreateNotificationTemplateDTO | NotificationTemplateDTO) {
    if (!dto.name || !dto.channel || !dto.body || !dto.status) {
      throw new Error('Missing required fields');
    }
    if (!['email', 'sms', 'push'].includes(dto.channel)) {
      throw new Error('Invalid channel');
    }
    if (!['active', 'inactive'].includes(dto.status)) {
      throw new Error('Invalid status');
    }
    if (dto.channel === 'email' && (!dto.subject || dto.subject.trim() === '')) {
      throw new Error('Subject is required for email channel');
    }
  }
}

// Repository (stub)
export class NotificationTemplateRepository {
  async findAll() { /* DB call */ }
  async findById(id: string) { /* DB call */ }
  async findByNameAndChannel(name: string, channel: string) { /* DB call */ }
  async create(entity: NotificationTemplateDTO) { /* DB call */ }
  async update(entity: NotificationTemplateDTO) { /* DB call */ }
  toEntity(dto: CreateNotificationTemplateDTO): NotificationTemplateDTO { /* mapping */ }
  merge(existing: NotificationTemplateDTO, dto: UpdateNotificationTemplateDTO): NotificationTemplateDTO { /* merge */ }
}
```

---

# .NET Web API Implementation Note Template

## .NET Mapping

- Use ASP.NET Core MVC controllers or minimal APIs to expose the endpoints.
- Controllers will receive DTOs decorated with validation attributes.
- Use dependency injection for services and repositories.
- Use IActionResult or typed ActionResult<T> for responses.

## Suggested Structure

- **DTOs**: CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO, NotificationTemplateDTO.
- **Validators**: Use FluentValidation or DataAnnotations for input validation.
- **Services**: NotificationTemplateService implementing business logic.
- **Repositories**: NotificationTemplateRepository using EF Core DbContext.
- **Entities/Models**: NotificationTemplate EF Core entity class.

## Validation

- Use DataAnnotations (e.g., [Required], [StringLength], [EnumDataType]) or FluentValidation for complex rules.
- Conditional validation (subject required if channel is email) implemented via FluentValidation or custom validation attributes.
- Business rules enforced in service layer.

## Persistence

- EF Core entity with properties matching the database model.
- Configure unique indexes (e.g., on name + channel) via Fluent API.
- Use migrations to create/update schema.
- Consider concurrency tokens (e.g., rowversion) for update conflicts.

## Testing

- Unit tests with xUnit or NUnit for services and validators.
- Integration tests using WebApplicationFactory to test controllers and middleware.
- API contract tests to verify request/response schemas and status codes.

---

This plan provides a clean architecture approach to managing notification templates with clear separation of concerns, validation, and security considerations. The .NET note outlines how to translate this design into an ASP.NET Core Web API implementation.