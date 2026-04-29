This patch proposes adding a new Notification Template Management feature to the codebase, implementing both backend API and frontend React UI components as per the provided plans and requirements.

**Implementation approach:**

- Backend:
  - Add a new `NotificationTemplate` entity with fields: id, name, channel, subject, body, status, createdAt, updatedAt.
  - Implement REST API endpoints:
    - `GET /notification-templates` to list all templates.
    - `POST /notification-templates` to create a new template.
    - `PUT /notification-templates/{id}` to update an existing template.
    - `PATCH /notification-templates/{id}/deactivate` to deactivate a template.
  - Include validation rules enforcing required fields, conditional subject requirement for email channel, and allowed enum values.
  - Implement service layer enforcing business rules and validation.
  - Repository layer for DB access (stubbed or illustrative).
  - Error handling for validation errors, not found, and other cases.
  - Assume admin authorization middleware is in place (to be confirmed).

- Frontend:
  - Add React components:
    - `NotificationTemplatesPage` container managing state and API calls.
    - `NotificationTemplatesList` to display templates with edit and deactivate buttons.
    - `NotificationTemplateForm` for create/edit with validation.
    - `DeactivateTemplateButton` with confirmation dialog.
    - `LoadingIndicator` and `ErrorMessage` reusable components.
  - Implement form validation including conditional subject requirement.
  - Manage loading, error, empty states.
  - Accessibility features as described.
  - Use fetch API for backend communication.

**Likely files to change/add:**

- Backend:
  - `src/controllers/notificationTemplate.controller.ts`
  - `src/services/notificationTemplate.service.ts`
  - `src/repositories/notificationTemplate.repository.ts`
  - `src/dtos/notificationTemplate.dto.ts`
  - `src/routes/notificationTemplate.routes.ts` (or equivalent)
  - Database migration files (not included here, to be created)

- Frontend:
  - `src/components/NotificationTemplatesPage.tsx`
  - `src/components/NotificationTemplatesList.tsx`
  - `src/components/NotificationTemplateForm.tsx`
  - `src/components/DeactivateTemplateButton.tsx`
  - `src/components/LoadingIndicator.tsx`
  - `src/components/ErrorMessage.tsx`
  - `src/types.ts` (for shared types)

**Test impact:**

- New unit tests needed for validation logic, service methods, and API endpoints.
- UI tests for form validation, user interactions, and error handling.
- Integration and E2E tests for full workflows.
- Tests for authorization and error cases.

**Risks:**

- Concurrency control not implemented; potential race conditions on updates/deactivation.
- Authorization enforcement assumed but not shown; must be confirmed.
- No pagination/filtering on list endpoint; may impact scalability.
- No audit/versioning implemented.
- Input sanitization and security hardening not fully covered.
- UI error handling could be improved for API errors.