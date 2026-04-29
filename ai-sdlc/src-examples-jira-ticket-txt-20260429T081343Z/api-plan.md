# Node API Implementation Plan Template

## API Overview

The Notification Template Management API enables admin users to manage notification templates used for customer notifications. Templates have fields such as name, channel (email, sms, push), subject, body, and status (active/inactive). Admins can list all templates, create new ones, update existing ones, and deactivate templates. Only active templates are eligible for sending notifications.

The lifecycle of a NotificationTemplate involves creation, optional editing, activation (via status field), and deactivation. Business rules enforce that only active templates are usable for sending notifications, and deactivated templates cannot be used.

## REST Endpoints

| Method | Path                               | Purpose                          | Request Body                              | Response Body                         | Status Codes                   |
|--------|----------------------------------|---------------------------------|------------------------------------------|-------------------------------------|-------------------------------|
| GET    | /notification-templates          | Retrieve list of all templates   | None                                     | Array of NotificationTemplateDTO    | 200 OK                        |
| POST   | /notification-templates          | Create a new notification template | CreateNotificationTemplateDTO            | NotificationTemplateDTO             | 201 Created, 400 Bad Request  |
| PUT    | /notification-templates/{id}     | Update an existing template      | UpdateNotificationTemplateDTO             | NotificationTemplateDTO             | 200 OK, 400 Bad Request, 404 Not Found |
| PATCH  | /notification-templates/{id}/deactivate | Deactivate a notification template | None                                     | NotificationTemplateDTO             | 200 OK, 404 Not Found          |

## DTOs

```typescript
// Allowed channels and statuses
export type Channel = "email" | "sms" | "push";
export type Status = "active" | "inactive";

export interface NotificationTemplateDTO {
  id: string;
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
  createdAt: string; // ISO string
  updatedAt: string; // ISO string
}

export interface CreateNotificationTemplateDTO {
  name: string;
  channel: Channel;
  subject?: string;
  body: string;
  status: Status;
}

export interface UpdateNotificationTemplateDTO {
  name?: string;
  channel?: Channel;
  subject?: string;
  body?: string;
  status?: Status;
}
```

## Validation Rules

| Validation Rule                                         | Request Validation                              | Service Layer Enforcement                          |
|---------------------------------------------------------|------------------------------------------------|---------------------------------------------------|
| All required fields must be provided on create/edit     | Validate presence of required fields on POST and PUT requests | Re-validate in service to prevent bypass           |
| Subject is required if channel is "email"               | Conditional validation: if channel === "email", subject must be non-empty | Enforce subject presence for email channel          |
| Channel must be one of "email", "sms", "push"            | Enum validation on channel field                | Enforce allowed values                              |
| Status must be "active" or "inactive"                    | Enum validation on status field                  | Enforce allowed values                              |
| Only active templates can be used for sending notifications | N/A (enforced outside this API)                  | Enforce business rule: only active templates usable |
| Deactivated templates cannot be used for sending notifications | N/A (enforced outside this API)                  | Enforce business rule                               |

## Database Model

**NotificationTemplate Entity**

| Field     | Type          | Constraints                         |
|-----------|---------------|-----------------------------------|
| id        | UUID          | Primary key, generated             |
| name      | string        | Not null                          |
| channel   | string        | Not null, enum: email, sms, push  |
| subject   | string        | Nullable, required if channel=email|
| body      | text          | Not null                         |
| status    | string        | Not null, enum: active, inactive  |
| createdAt | timestamp     | Not null, default current timestamp |
| updatedAt | timestamp     | Not null, updated on change       |

**Indexes and Constraints**

- Unique index on (name, channel) to prevent duplicate template names per channel (optional, depending on business need).
- Index on status for efficient filtering by active/inactive.
- Timestamps for audit and sorting.

## Service Responsibilities

- **Controller**: Handle HTTP requests, parse and validate input DTOs, call service methods, handle HTTP response codes.
- **Service**: Business logic enforcement including validation rules, status transitions, and business rules. Calls repository for persistence.
- **Repository**: Data access layer, CRUD operations on NotificationTemplate entity.
- **Mapper**: Convert between database entity and DTOs.

## Error Handling

- **Validation Errors (400 Bad Request)**: Missing required fields, invalid enum values, subject missing for email channel.
- **Not Found (404 Not Found)**: Template with given ID does not exist on update or deactivate.
- **Conflict (409 Conflict)**: Optional if enforcing unique constraints (e.g., duplicate name per channel).
- **Authorization (401/403)**: Only admin users allowed (assumed handled by middleware).
- **Unexpected Errors (500 Internal Server Error)**: Catch-all for unhandled exceptions.

## Security Considerations

- **Authentication & Authorization**: Only authenticated admin users can access these endpoints. Enforce via middleware.
- **Input Validation**: Strict validation to prevent injection attacks.
- **Audit Logging**: Log create, update, and deactivate actions with user info for traceability.
- **Sensitive Data**: Template body may contain sensitive content; ensure secure storage and access control.
- **Idempotency**: PUT and PATCH endpoints should be idempotent. POST creates new resource.

## Example Backend Stub Code

```typescript
// controller/notificationTemplate.controller.ts
import { Request, Response } from "express";
import { NotificationTemplateService } from "../services/notificationTemplate.service";
import { CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO } from "../dtos/notificationTemplate.dto";

export class NotificationTemplateController {
  constructor(private service: NotificationTemplateService) {}

  async list(req: Request, res: Response) {
    const templates = await this.service.listAll();
    res.status(200).json(templates);
  }

  async create(req: Request, res: Response) {
    const dto: CreateNotificationTemplateDTO = req.body;
    const created = await this.service.create(dto);
    res.status(201).json(created);
  }

  async update(req: Request, res: Response) {
    const id = req.params.id;
    const dto: UpdateNotificationTemplateDTO = req.body;
    const updated = await this.service.update(id, dto);
    res.status(200).json(updated);
  }

  async deactivate(req: Request, res: Response) {
    const id = req.params.id;
    const updated = await this.service.deactivate(id);
    res.status(200).json(updated);
  }
}

// services/notificationTemplate.service.ts
import { NotificationTemplateRepository } from "../repositories/notificationTemplate.repository";
import { CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO, NotificationTemplateDTO, Channel, Status } from "../dtos/notificationTemplate.dto";

export class NotificationTemplateService {
  constructor(private repo: NotificationTemplateRepository) {}

  async listAll(): Promise<NotificationTemplateDTO[]> {
    return this.repo.findAll();
  }

  async create(dto: CreateNotificationTemplateDTO): Promise<NotificationTemplateDTO> {
    this.validateCreate(dto);
    const entity = await this.repo.create(dto);
    return entity;
  }

  async update(id: string, dto: UpdateNotificationTemplateDTO): Promise<NotificationTemplateDTO> {
    const existing = await this.repo.findById(id);
    if (!existing) throw new Error("NotFound");
    this.validateUpdate(dto, existing);
    const updated = await this.repo.update(id, dto);
    return updated;
  }

  async deactivate(id: string): Promise<NotificationTemplateDTO> {
    const existing = await this.repo.findById(id);
    if (!existing) throw new Error("NotFound");
    if (existing.status === "inactive") return existing; // idempotent
    const updated = await this.repo.update(id, { status: "inactive" });
    return updated;
  }

  private validateCreate(dto: CreateNotificationTemplateDTO) {
    if (!dto.name) throw new Error("ValidationError: name required");
    if (!dto.channel) throw new Error("ValidationError: channel required");
    if (!dto.body) throw new Error("ValidationError: body required");
    if (!dto.status) throw new Error("ValidationError: status required");
    if (!["email", "sms", "push"].includes(dto.channel)) throw new Error("ValidationError: invalid channel");
    if (!["active", "inactive"].includes(dto.status)) throw new Error("ValidationError: invalid status");
    if (dto.channel === "email" && (!dto.subject || dto.subject.trim() === "")) {
      throw new Error("ValidationError: subject required for email channel");
    }
  }

  private validateUpdate(dto: UpdateNotificationTemplateDTO, existing: NotificationTemplateDTO) {
    if (dto.channel && !["email", "sms", "push"].includes(dto.channel)) throw new Error("ValidationError: invalid channel");
    if (dto.status && !["active", "inactive"].includes(dto.status)) throw new Error("ValidationError: invalid status");
    // If channel changes to email, subject must be present
    const channel = dto.channel ?? existing.channel;
    const subject = dto.subject ?? existing.subject;
    if (channel === "email" && (!subject || subject.trim() === "")) {
      throw new Error("ValidationError: subject required for email channel");
    }
  }
}

// repositories/notificationTemplate.repository.ts
import { NotificationTemplateDTO, CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO } from "../dtos/notificationTemplate.dto";

export class NotificationTemplateRepository {
  async findAll(): Promise<NotificationTemplateDTO[]> {
    // DB call to fetch all templates
    return [];
  }

  async findById(id: string): Promise<NotificationTemplateDTO | null> {
    // DB call to find by id
    return null;
  }

  async create(dto: CreateNotificationTemplateDTO): Promise<NotificationTemplateDTO> {
    // DB insert and return created entity
    return {
      id: "generated-uuid",
      ...dto,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
  }

  async update(id: string, dto: UpdateNotificationTemplateDTO): Promise<NotificationTemplateDTO> {
    // DB update and return updated entity
    return {
      id,
      name: dto.name ?? "existing-name",
      channel: dto.channel ?? "email",
      subject: dto.subject,
      body: dto.body ?? "existing-body",
      status: dto.status ?? "active",
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
  }
}
```

---

# .NET Web API Implementation Note Template

## .NET Mapping

This API can be implemented using ASP.NET Core Web API controllers or minimal APIs. Controllers would expose endpoints matching the REST paths, using attribute routing. DTOs would be defined as C# classes. Validation can be done using DataAnnotations or FluentValidation. Entity Framework Core would be used for persistence.

## Suggested Structure

- **DTOs**: CreateNotificationTemplateDTO, UpdateNotificationTemplateDTO, NotificationTemplateDTO classes.
- **Validators**: FluentValidation validators for create and update DTOs enforcing required fields and conditional subject validation.
- **Services**: NotificationTemplateService implementing business logic and validation.
- **Repositories**: NotificationTemplateRepository using EF Core DbContext for data access.
- **Entities**: NotificationTemplate EF Core entity with fields and constraints.

## Validation

Use DataAnnotations for simple required and enum validations. Use FluentValidation for conditional rules (e.g., subject required if channel is email). Business rules like status enforcement should live in the service layer.

## Persistence

Configure EF Core entity with appropriate data types, required fields, and enums. Use unique indexes if needed (e.g., on name + channel). Use migrations to create/update database schema. Consider concurrency tokens (e.g., rowversion) for optimistic concurrency control.

## Testing

- **Unit Tests**: Use xUnit or NUnit to test validation logic, service methods, and repository mocks.
- **Integration Tests**: Use WebApplicationFactory to test API endpoints with in-memory or test database.
- **API Contract Tests**: Verify request/response contracts and error handling.

---

This plan provides a clean architecture approach with clear separation of concerns, validation, business rules, and security considerations for managing notification templates in a Node.js backend, with a complementary note on .NET Web API implementation.