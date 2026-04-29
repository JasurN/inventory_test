# Node API Implementation Plan Template

## API Overview

The Notification Template Management API enables admin users to manage notification templates used for customer notifications across multiple channels (email, SMS, push). Admins can create, edit, view, and deactivate templates. Templates have required fields including name, channel, body, and status, with additional conditional requirements (e.g., subject required for email channel). Only active templates are valid for sending notifications. The lifecycle includes creation, update, activation (implicitly via status), deactivation, and retrieval. The API enforces admin-only access and validation rules to maintain data integrity and business constraints.

## REST Endpoints

| Method | Path                                 | Purpose                                  | Request Body                                     | Response Body                                  | Status Codes                          |
|--------|-------------------------------------|------------------------------------------|-------------------------------------------------|-----------------------------------------------|-------------------------------------|
| GET    | /notification-templates              | List all notification templates          | None                                            | Array of NotificationTemplateResponseDTO      | 200 OK                              |
| POST   | /notification-templates              | Create a new notification template       | NotificationTemplateCreateDTO                    | NotificationTemplateResponseDTO                | 201 Created, 400 Bad Request, 401 Unauthorized, 403 Forbidden |
| PUT    | /notification-templates/{id}         | Update an existing notification template | NotificationTemplateUpdateDTO                    | NotificationTemplateResponseDTO                | 200 OK, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| PATCH  | /notification-templates/{id}/deactivate | Deactivate a notification template       | None                                            | NotificationTemplateResponseDTO                | 200 OK, 401 Unauthorized, 403 Forbidden, 404 Not Found |

## DTOs

```typescript
// Request DTOs
export type Channel = "email" | "sms" | "push";
export type Status = "active" | "inactive";

export interface NotificationTemplateCreateDTO {
  name: string;
  channel: Channel;
  subject?: string; // Required if channel === 'email'
  body: string;
  status: Status;
}

export interface NotificationTemplateUpdateDTO {
  name?: string;
  channel?: Channel;
  subject?: string; // Required if channel === 'email'
  body?: string;
  status?: Status;
}

// Response DTO
export interface NotificationTemplateResponseDTO {
  id: string;
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
  createdAt: string; // ISO timestamp
  updatedAt: string; // ISO timestamp
}
```

## Validation Rules

| Validation Rule                                           | Request Validation                          | Service Layer Enforcement                      |
|-----------------------------------------------------------|---------------------------------------------|------------------------------------------------|
| `name` is required                                        | Validate presence and non-empty string      | Enforce uniqueness if required (optional)      |
| `channel` is required and must be one of "email", "sms", "push" | Validate enum membership                      | Confirm channel validity                        |
| `body` is required                                        | Validate presence and non-empty string      | Confirm non-empty body                          |
| `status` is required and must be "active" or "inactive" | Validate enum membership                      | Confirm valid status                            |
| `subject` is required if `channel` is "email"            | Conditional validation: if channel === "email", subject must be present and non-empty | Enforce subject presence for email channel     |
| Only active templates can be used for sending notifications | N/A (enforced in notification sending service) | Enforce status check before sending notifications |
| Only admin users can manage templates                     | Authorization middleware checks admin role  | Double-check in service layer                   |

## Database Model

**Entity:** NotificationTemplate

| Field     | Type      | Constraints                                  |
|-----------|-----------|----------------------------------------------|
| id        | UUID      | Primary key, generated                       |
| name      | string    | Required, indexed for search                  |
| channel   | string    | Required, enum: "email", "sms", "push"       |
| subject   | string    | Nullable, required if channel = "email"      |
| body      | string    | Required                                      |
| status    | string    | Required, enum: "active", "inactive"          |
| createdAt | timestamp | Auto-set on insert                            |
| updatedAt | timestamp | Auto-set on update                            |

**Indexes & Constraints:**

- Primary key on `id`
- Index on `name` for efficient listing/search
- Optional unique constraint on `name` + `channel` if desired (not specified)
- Timestamps for audit and sorting

## Service Responsibilities

- **Controller:** Handle HTTP requests/responses, parse and validate input DTOs, call service methods, handle HTTP status codes.
- **Service:** Enforce business rules (e.g., subject required for email, status validation), coordinate repository calls, handle domain logic, throw domain-specific errors.
- **Repository:** Abstract database access, perform CRUD operations, handle transactions if needed.
- **Mapper:** Convert between database entities and DTOs.

## Error Handling

| Error Type           | Description                                      | Response Status | Response Body Example                      |
|----------------------|-------------------------------------------------|-----------------|-------------------------------------------|
| Validation Error     | Missing or invalid fields                        | 400 Bad Request | `{ error: "Validation failed: subject is required for email channel" }` |
| Not Found            | Template with given ID does not exist            | 404 Not Found   | `{ error: "Notification template not found" }` |
| Conflict             | (Optional) Duplicate name/channel if uniqueness enforced | 409 Conflict    | `{ error: "Template with this name and channel already exists" }` |
| Authorization Error  | User is not an admin                              | 403 Forbidden   | `{ error: "Access denied" }`               |
| Authentication Error | User not authenticated                            | 401 Unauthorized| `{ error: "Authentication required" }`    |
| Unexpected Error     | Server or DB failure                              | 500 Internal Server Error | `{ error: "Internal server error" }` |

## Security Considerations

- **Authentication:** Require valid user authentication token (e.g., JWT).
- **Authorization:** Verify user role is admin before allowing any template management operation.
- **Input Validation:** Strict validation on all inputs to prevent injection or malformed data.
- **Audit Logging:** (Open question) Consider logging changes to templates for audit/history.
- **Sensitive Data:** Templates may contain sensitive content; ensure secure storage and access control.
- **Idempotency:** POST create endpoint should be idempotent if client provides idempotency key (optional enhancement).
- **Rate Limiting:** Protect endpoints from abuse.

## Example Backend Stub Code

```typescript
// src/dtos/notificationTemplate.dto.ts
export type Channel = "email" | "sms" | "push";
export type Status = "active" | "inactive";

export interface NotificationTemplateCreateDTO {
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
}

export interface NotificationTemplateUpdateDTO {
  name?: string;
  channel?: Channel;
  subject?: string;
  body?: string;
  status?: Status;
}

export interface NotificationTemplateResponseDTO {
  id: string;
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
  createdAt: string;
  updatedAt: string;
}

// src/controllers/notificationTemplate.controller.ts
import { Request, Response } from "express";
import { NotificationTemplateService } from "../services/notificationTemplate.service";

export class NotificationTemplateController {
  constructor(private service: NotificationTemplateService) {}

  async list(req: Request, res: Response) {
    // Authorization check assumed done in middleware
    const templates = await this.service.listTemplates();
    res.status(200).json(templates);
  }

  async create(req: Request, res: Response) {
    try {
      const dto = req.body as NotificationTemplateCreateDTO;
      const created = await this.service.createTemplate(dto);
      res.status(201).json(created);
    } catch (err) {
      if (err instanceof ValidationError) {
        res.status(400).json({ error: err.message });
      } else if (err instanceof AuthorizationError) {
        res.status(403).json({ error: err.message });
      } else {
        res.status(500).json({ error: "Internal server error" });
      }
    }
  }

  async update(req: Request, res: Response) {
    try {
      const id = req.params.id;
      const dto = req.body as NotificationTemplateUpdateDTO;
      const updated = await this.service.updateTemplate(id, dto);
      res.status(200).json(updated);
    } catch (err) {
      if (err instanceof NotFoundError) {
        res.status(404).json({ error: err.message });
      } else if (err instanceof ValidationError) {
        res.status(400).json({ error: err.message });
      } else if (err instanceof AuthorizationError) {
        res.status(403).json({ error: err.message });
      } else {
        res.status(500).json({ error: "Internal server error" });
      }
    }
  }

  async deactivate(req: Request, res: Response) {
    try {
      const id = req.params.id;
      const deactivated = await this.service.deactivateTemplate(id);
      res.status(200).json(deactivated);
    } catch (err) {
      if (err instanceof NotFoundError) {
        res.status(404).json({ error: err.message });
      } else if (err instanceof AuthorizationError) {
        res.status(403).json({ error: err.message });
      } else {
        res.status(500).json({ error: "Internal server error" });
      }
    }
  }
}

// src/services/notificationTemplate.service.ts
import { NotificationTemplateCreateDTO, NotificationTemplateUpdateDTO, NotificationTemplateResponseDTO } from "../dtos/notificationTemplate.dto";
import { NotificationTemplateRepository } from "../repositories/notificationTemplate.repository";

export class NotificationTemplateService {
  constructor(private repository: NotificationTemplateRepository) {}

  async listTemplates(): Promise<NotificationTemplateResponseDTO[]> {
    return this.repository.findAll();
  }

  async createTemplate(dto: NotificationTemplateCreateDTO): Promise<NotificationTemplateResponseDTO> {
    this.validateDto(dto);
    // Additional business rules can be enforced here
    return this.repository.create(dto);
  }

  async updateTemplate(id: string, dto: NotificationTemplateUpdateDTO): Promise<NotificationTemplateResponseDTO> {
    const existing = await this.repository.findById(id);
    if (!existing) throw new NotFoundError("Notification template not found");
    const updatedDto = { ...existing, ...dto };
    this.validateDto(updatedDto);
    return this.repository.update(id, updatedDto);
  }

  async deactivateTemplate(id: string): Promise<NotificationTemplateResponseDTO> {
    const existing = await this.repository.findById(id);
    if (!existing) throw new NotFoundError("Notification template not found");
    if (existing.status === "inactive") return existing; // idempotent
    return this.repository.update(id, { status: "inactive" });
  }

  private validateDto(dto: NotificationTemplateCreateDTO | NotificationTemplateUpdateDTO) {
    if (!dto.name || dto.name.trim() === "") {
      throw new ValidationError("Name is required");
    }
    if (!dto.channel || !["email", "sms", "push"].includes(dto.channel)) {
      throw new ValidationError("Channel must be one of email, sms, push");
    }
    if (!dto.body || dto.body.trim() === "") {
      throw new ValidationError("Body is required");
    }
    if (!dto.status || !["active", "inactive"].includes(dto.status)) {
      throw new ValidationError("Status must be active or inactive");
    }
    if (dto.channel === "email" && (!dto.subject || dto.subject.trim() === "")) {
      throw new ValidationError("Subject is required for email channel templates");
    }
  }
}

// src/repositories/notificationTemplate.repository.ts
import { NotificationTemplateCreateDTO, NotificationTemplateResponseDTO } from "../dtos/notificationTemplate.dto";

export class NotificationTemplateRepository {
  // This is a stub; replace with actual DB calls (e.g., using TypeORM, Prisma, Sequelize)
  private templates: Map<string, NotificationTemplateResponseDTO> = new Map();

  async findAll(): Promise<NotificationTemplateResponseDTO[]> {
    return Array.from(this.templates.values());
  }

  async findById(id: string): Promise<NotificationTemplateResponseDTO | null> {
    return this.templates.get(id) ?? null;
  }

  async create(dto: NotificationTemplateCreateDTO): Promise<NotificationTemplateResponseDTO> {
    const id = generateUUID();
    const now = new Date().toISOString();
    const entity: NotificationTemplateResponseDTO = {
      id,
      createdAt: now,
      updatedAt: now,
      ...dto,
    };
    this.templates.set(id, entity);
    return entity;
  }

  async update(id: string, dto: Partial<NotificationTemplateResponseDTO>): Promise<NotificationTemplateResponseDTO> {
    const existing = this.templates.get(id);
    if (!existing) throw new Error("Not found");
    const updated = {
      ...existing,
      ...dto,
      updatedAt: new Date().toISOString(),
    };
    this.templates.set(id, updated);
    return updated;
  }
}

function generateUUID(): string {
  // Simple UUID generator stub; use a library like uuid in real code
  return Math.random().toString(36).substring(2, 15);
}

// Custom error classes
class ValidationError extends Error {}
class NotFoundError extends Error {}
class AuthorizationError extends Error {}
```

---

# .NET Web API Implementation Note Template

## .NET Mapping

This API can be implemented in ASP.NET Core using Controllers or Minimal APIs:

- Define `NotificationTemplateController` with endpoints matching the REST paths.
- Use `[Authorize(Roles = "Admin")]` attribute to restrict access.
- Use model binding with DTO classes for request and response.
- Use FluentValidation or DataAnnotations for validation.
- Use Entity Framework Core for persistence with a `NotificationTemplate` entity.
- Use dependency injection for services and repositories.

## Suggested Structure

- DTOs: `NotificationTemplateCreateDto`, `NotificationTemplateUpdateDto`, `NotificationTemplateResponseDto`
- Validators: FluentValidation classes for create/update DTOs enforcing required fields and conditional rules.
- Services: `NotificationTemplateService` implementing business logic and validation.
- Repositories: `NotificationTemplateRepository` using EF Core DbContext.
- EF Core Entity: `NotificationTemplate` with fields and configurations.

## Validation

- Use DataAnnotations for simple required and enum validation.
- Use FluentValidation for complex rules like conditional subject requirement.
- Business rules enforced in service layer.

## Persistence

- EF Core entity with properties matching fields.
- Configure unique indexes if needed.
- Use migrations to create/update schema.
- Use concurrency tokens (e.g., rowversion) if concurrent updates are a concern.

## Testing

- Unit tests with xUnit/NUnit for validators, services, and repositories.
- Integration tests using `WebApplicationFactory` to test API endpoints.
- API contract tests to verify request/response schemas and error handling.
- Authorization tests to ensure only admins can access endpoints.

---

This plan provides a clear, maintainable, and secure approach to implementing Notification Template Management in both Node.js and .NET environments following clean architecture principles.