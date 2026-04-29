# Test Plan Template

## Unit Tests

Cover validation logic, business rules, mappers, and service behavior.

- **Validation Logic**
  - `validateDTO` rejects missing required fields:  
    - Test `validateDTO` throws error if `name` is empty or missing.  
    - Test `validateDTO` throws error if `channel` is missing or invalid (not in `email`, `sms`, `push`).  
    - Test `validateDTO` throws error if `body` is empty or missing.  
    - Test `validateDTO` throws error if `status` is missing or invalid (not `active` or `inactive`).  
    - Test `validateDTO` throws error if `channel` is `email` and `subject` is missing or empty.  
    - Test `validateDTO` passes valid DTOs with optional `subject` for non-email channels.  

- **Business Rules Enforcement**
  - Only admin users can manage templates:  
    - Test service methods throw or reject if user role is not admin (mock auth).  
  - Only active templates can be used for sending notifications (outside API scope but service method can check):  
    - Test method `canSendNotification(template)` returns true only if `status === 'active'`.  
  - Deactivating a template sets status to `inactive` and prevents sending:  
    - Test `deactivate` method sets status to `inactive`.  
    - Test `deactivate` method throws if template not found.  

- **Repository Mappers**
  - Test mapping from DTO to entity and back preserves all fields correctly.  
  - Test merging update DTO into existing entity applies only allowed fields and validates.  

- **Service Behavior**
  - `create` method:  
    - Creates new template when valid and no duplicates exist.  
    - Throws conflict error if template with same name and channel exists.  
  - `update` method:  
    - Updates existing template fields correctly.  
    - Throws not found if template ID does not exist.  
    - Throws conflict if updating name/channel to duplicate existing template.  
  - `listAll` returns all templates.  
  - `deactivate` sets status and persists.  

- **Form Validation (React)**
  - Validate required fields on blur and submit:  
    - Test form shows error if `name` is empty on blur.  
    - Test form shows error if `channel` is empty or invalid.  
    - Test form shows error if `body` is empty.  
    - Test form shows error if `status` is empty or invalid.  
    - Test form shows error if `channel` is `email` and `subject` is empty.  
  - Validate no errors for valid inputs.  
  - Validate that changing channel from `email` to `sms` removes subject required error.  
  - Validate form disables inputs and buttons during submission.  

## API Tests

Cover REST endpoints, success responses, validation errors, not found, conflict, and edge cases.

- **GET /notification-templates**
  - Returns 200 with list of templates for admin user.  
  - Returns 401 Unauthorized for non-authenticated or non-admin users.  
  - Returns empty array if no templates exist.  

- **POST /notification-templates**
  - Creates template successfully with valid data, returns 201 and created template.  
  - Returns 400 Bad Request if required fields missing or invalid (test each required field).  
  - Returns 400 if `subject` missing when `channel` is `email`.  
  - Returns 409 Conflict if template with same `name` and `channel` exists.  
  - Returns 401 Unauthorized if user not admin or unauthenticated.  

- **PUT /notification-templates/{id}**
  - Updates existing template successfully with valid partial or full data, returns 200.  
  - Returns 400 Bad Request if invalid fields or validation fails.  
  - Returns 404 Not Found if template ID does not exist.  
  - Returns 409 Conflict if update causes duplicate name/channel.  
  - Returns 401 Unauthorized if user not admin or unauthenticated.  

- **PATCH /notification-templates/{id}/deactivate**
  - Deactivates template successfully, returns 200 with updated template.  
  - Returns 404 Not Found if template ID does not exist.  
  - Returns 401 Unauthorized if user not admin or unauthenticated.  

- **Error Handling**
  - API returns consistent error JSON with `error` field and details if applicable.  
  - Handles unexpected errors with 500 Internal Server Error.  

- **Authorization**
  - All endpoints reject non-admin users with 401 Unauthorized.  

## E2E Tests

Cover the primary user workflows from UI interaction through API response.

- **Admin views list of notification templates**
  - Load page, see loading spinner, then list or empty state.  
  - Handle API error and show error banner with retry.  

- **Admin creates a new notification template**
  - Click "Create Template" button.  
  - Fill form with valid data (email channel with subject).  
  - Submit form, see loading state on submit button.  
  - On success, form closes and list refreshes showing new template.  
  - On validation error (e.g. missing subject for email), inline errors shown, focus on first error.  
  - On API error, error banner shown with retry option.  

- **Admin edits an existing notification template**
  - Click "Edit" on a template row.  
  - Form opens with pre-filled data.  
  - Change fields, including changing channel from email to sms (subject becomes optional).  
  - Submit and verify updated data in list.  
  - Handle validation and API errors as above.  

- **Admin deactivates a notification template**
  - Click "Deactivate" button on active template.  
  - Confirmation dialog appears.  
  - Cancel closes dialog without change.  
  - Confirm sends PATCH request, disables button during API call.  
  - On success, status updates to inactive in list.  
  - On API error, error banner shown.  

- **Activate an inactive template**
  - Click "Activate" button on inactive template.  
  - Button disables during API call.  
  - On success, status updates to active.  

- **Access control**
  - Non-admin user tries to access UI or API endpoints, denied access or redirected.  

## Negative Tests

Include invalid input, missing required fields, invalid state transitions, duplicates, and permission failures.

- **Invalid Inputs**
  - Submit create/update with empty `name`, `channel`, `body`, or `status`.  
  - Submit create/update with invalid `channel` (e.g., "fax").  
  - Submit create/update with invalid `status` (e.g., "pending").  
  - Submit create/update with missing `subject` when `channel` is `email`.  
  - Submit update with partial invalid data (e.g., empty body).  

- **Duplicates**
  - Create template with name and channel that already exists.  
  - Update template to name and channel that conflicts with another template.  

- **Invalid State Transitions**
  - Attempt to deactivate a template that is already inactive (should succeed idempotently or no-op).  
  - Attempt to activate a template that is already active (should succeed or no-op).  

- **Permission Failures**
  - API calls without authentication return 401.  
  - API calls with authenticated non-admin user return 401.  
  - UI does not show management UI to non-admin users or redirects.  

- **API Error Handling**
  - Simulate server errors (500) and verify UI shows error banners and retry options.  
  - Network failures during API calls handled gracefully with user feedback.  

## Edge Cases

Cover boundaries, empty data, large payloads, status changes, and dependent business rules.

- **Empty Data**
  - List returns empty array, UI shows empty state message.  
  - Create template with minimal valid data (e.g., subject omitted for sms channel).  

- **Large Payloads**
  - Create/update with very long `name`, `subject`, and `body` fields (test system limits if known).  
  - Verify UI handles large text inputs without breaking layout.  

- **Status Changes**
  - Rapidly toggle activate/deactivate multiple times, verify system consistency and no race conditions.  
  - Verify deactivated templates are excluded from sending notifications (if testable).  

- **Dependent Business Rules**
  - Changing channel from `email` to `sms` removes subject requirement.  
  - Changing channel from `sms` to `email` requires subject to be filled before submit.  

- **Concurrency**
  - Simulate concurrent updates to same template, verify conflict errors or last-write-wins behavior.  

## Regression Risks

List behavior likely to break when requirements change.

- Validation rules for required fields and conditional subject requirement.  
- Enforcement of unique name + channel constraint (if implemented).  
- Authorization checks restricting access to admin users only.  
- Status toggling logic and confirmation dialog behavior.  
- API error response formats and HTTP status codes.  
- UI form validation and error display logic.  
- Handling of empty and large data sets in list and form.  
- State management for loading and error states in UI components.  

## Suggested Test Data

Provide concrete valid and invalid examples.

| Test Case                          | Data Example                                                                                     |
|----------------------------------|------------------------------------------------------------------------------------------------|
| Valid Email Template              | `{ name: "Welcome Email", channel: "email", subject: "Welcome!", body: "Hello, user!", status: "active" }` |
| Valid SMS Template                | `{ name: "Promo SMS", channel: "sms", body: "Get 20% off!", status: "inactive" }`               |
| Valid Push Template               | `{ name: "Alert Push", channel: "push", body: "You have a new alert.", status: "active" }`      |
| Missing Required Field (name)     | `{ channel: "email", subject: "Hi", body: "Body", status: "active" }`                           |
| Missing Subject for Email         | `{ name: "Email No Subject", channel: "email", body: "Body", status: "active" }`                |
| Invalid Channel                  | `{ name: "Invalid Channel", channel: "fax", body: "Body", status: "active" }`                   |
| Invalid Status                   | `{ name: "Invalid Status", channel: "sms", body: "Body", status: "pending" }`                   |
| Duplicate Template               | Create two templates with `{ name: "Promo", channel: "sms", body: "Body", status: "active" }`    |
| Large Body Text                 | `{ name: "Large Body", channel: "email", subject: "Subject", body: "A".repeat(10000), status: "active" }` |
| Empty List                      | No templates in DB                                                                                 |

---

This test plan ensures comprehensive coverage of the Notification Template Management feature, validating correctness, security, usability, and robustness across unit, API, and end-to-end levels. It also addresses negative and edge cases to prevent regressions and unexpected failures.