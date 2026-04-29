# Test Plan Template

## Unit Tests

Focus on validation logic, business rules, mappers, and service behavior.

- **Validation Logic**
  - `validateCreate_withAllRequiredFields_shouldPass`
  - `validateCreate_missingName_shouldThrowValidationError`
  - `validateCreate_missingChannel_shouldThrowValidationError`
  - `validateCreate_invalidChannel_shouldThrowValidationError`
  - `validateCreate_missingBody_shouldThrowValidationError`
  - `validateCreate_missingStatus_shouldThrowValidationError`
  - `validateCreate_emailChannelMissingSubject_shouldThrowValidationError`
  - `validateUpdate_channelChangeToEmailWithoutSubject_shouldThrowValidationError`
  - `validateUpdate_invalidStatus_shouldThrowValidationError`
  - `validateUpdate_invalidChannel_shouldThrowValidationError`

- **Business Rules**
  - `deactivateTemplate_shouldSetStatusToInactive`
  - `deactivateTemplate_alreadyInactive_shouldBeIdempotent`
  - `onlyActiveTemplates_canBeUsedForSendingNotifications`
  - `createTemplate_withDuplicateNameAndChannel_shouldThrowConflictError` (if uniqueness enforced)

- **Mapper Tests**
  - `entityToDTO_mapping_shouldMapAllFieldsCorrectly`
  - `dtoToEntity_mapping_shouldMapAllFieldsCorrectly`

- **Service Behavior**
  - `createTemplate_withValidData_shouldReturnCreatedTemplate`
  - `updateTemplate_existingTemplate_shouldReturnUpdatedTemplate`
  - `updateTemplate_nonExistingTemplate_shouldThrowNotFound`
  - `deactivateTemplate_existingActiveTemplate_shouldReturnInactiveTemplate`
  - `deactivateTemplate_nonExistingTemplate_shouldThrowNotFound`

## API Tests

Cover REST endpoints, success responses, validation errors, not found, conflict, and edge cases.

- **GET /notification-templates**
  - `getTemplates_shouldReturnListOfTemplates`
  - `getTemplates_emptyDatabase_shouldReturnEmptyList`

- **POST /notification-templates**
  - `createTemplate_validRequest_shouldReturn201Created`
  - `createTemplate_missingRequiredFields_shouldReturn400BadRequest`
  - `createTemplate_emailChannelMissingSubject_shouldReturn400BadRequest`
  - `createTemplate_invalidChannel_shouldReturn400BadRequest`
  - `createTemplate_invalidStatus_shouldReturn400BadRequest`
  - `createTemplate_duplicateNameAndChannel_shouldReturn409Conflict` (if uniqueness enforced)

- **PUT /notification-templates/{id}**
  - `updateTemplate_validRequest_shouldReturn200Ok`
  - `updateTemplate_nonExistingId_shouldReturn404NotFound`
  - `updateTemplate_invalidChannel_shouldReturn400BadRequest`
  - `updateTemplate_emailChannelMissingSubject_shouldReturn400BadRequest`
  - `updateTemplate_invalidStatus_shouldReturn400BadRequest`
  - `updateTemplate_partialUpdate_shouldPreserveUnchangedFields`

- **PATCH /notification-templates/{id}/deactivate**
  - `deactivateTemplate_existingActiveTemplate_shouldReturn200Ok`
  - `deactivateTemplate_alreadyInactiveTemplate_shouldReturn200OkIdempotent`
  - `deactivateTemplate_nonExistingId_shouldReturn404NotFound`

- **Authorization**
  - `allEndpoints_nonAdminUser_shouldReturn403Forbidden`

- **Error Handling**
  - `apiServerError_shouldReturn500InternalServerError`

## E2E Tests

Cover primary user workflows from UI interaction through API response.

- `admin_canViewNotificationTemplatesList`
- `admin_canCreateNotificationTemplate_withValidData`
- `admin_cannotCreateNotificationTemplate_withMissingRequiredFields`
- `admin_canEditNotificationTemplate_andSeeUpdatedData`
- `admin_cannotEditNotificationTemplate_withInvalidData`
- `admin_canDeactivateNotificationTemplate_andStatusUpdatesInList`
- `deactivatedTemplate_cannotBeUsedForSendingNotifications` (simulate sending or check status)
- `formValidation_showsErrorsForMissingRequiredFields`
- `formValidation_requiresSubjectWhenChannelIsEmail`
- `loadingIndicator_shownDuringApiCalls`
- `errorMessage_shownOnApiFailure`
- `confirmationDialog_shownBeforeDeactivation_andCancellingAbortsAction`
- `cancelButton_closesFormWithoutSaving`

## Negative Tests

Include invalid input, missing required fields, invalid state transitions, duplicates, and permission failures.

- `createTemplate_missingName_shouldFail`
- `createTemplate_emptyBody_shouldFail`
- `createTemplate_channelNotInEnum_shouldFail`
- `createTemplate_emailChannelWithoutSubject_shouldFail`
- `updateTemplate_invalidStatus_shouldFail`
- `updateTemplate_changeChannelToEmailWithoutSubject_shouldFail`
- `deactivateTemplate_alreadyInactive_shouldNotError`
- `deactivateTemplate_nonExistingId_shouldReturnNotFound`
- `createTemplate_duplicateNameAndChannel_shouldFail` (if uniqueness enforced)
- `nonAdminUser_accessEndpoints_shouldFailWithForbidden`
- `submitForm_withWhitespaceOnlyFields_shouldTrimAndValidate`
- `submitForm_withExcessivelyLongSubjectOrBody_shouldFailOrHandleGracefully` (if max length enforced)
- `apiRejectsMalformedJson_shouldReturn400BadRequest`

## Edge Cases

Cover boundaries, empty data, large payloads, status changes, and dependent business rules.

- `createTemplate_withMinimumLengthFields_shouldSucceed`
- `createTemplate_withMaximumLengthSubjectAndBody_shouldSucceedOrFailGracefully`
- `listTemplates_withNoTemplates_shouldShowEmptyMessage`
- `listTemplates_withLargeNumberOfTemplates_shouldPaginateOrPerformEfficiently`
- `deactivateTemplate_multipleTimes_shouldRemainIdempotent`
- `updateTemplate_changeStatusFromInactiveToActive_shouldSucceed`
- `updateTemplate_changeChannelFromSmsToEmail_requiresSubject`
- `createTemplate_withWhitespaceAroundFields_shouldTrimBeforeSave`
- `createTemplate_withNullSubjectForNonEmailChannel_shouldSucceed`
- `updateTemplate_removeSubject_whenChannelIsNotEmail_shouldSucceed`
- `updateTemplate_removeSubject_whenChannelIsEmail_shouldFailValidation`

## Regression Risks

List behavior likely to break when requirements change.

- Validation rules around conditional subject requirement for email channel.
- Status transitions and enforcement of active/inactive states.
- Uniqueness constraints on name and channel (if added later).
- API error response formats and status codes.
- UI form validation logic and error display.
- Deactivation flow and confirmation dialog behavior.
- Permission enforcement for admin-only access.
- Handling of empty or large lists in UI.
- Trimming of input fields before save.

## Suggested Test Data

### Valid Examples

- Template 1 (Email, Active)
  ```json
  {
    "name": "Welcome Email",
    "channel": "email",
    "subject": "Welcome to Our Service",
    "body": "Hello {{user}}, welcome!",
    "status": "active"
  }
  ```

- Template 2 (SMS, Inactive)
  ```json
  {
    "name": "Promo SMS",
    "channel": "sms",
    "body": "Get 20% off your next purchase!",
    "status": "inactive"
  }
  ```

- Template 3 (Push, Active)
  ```json
  {
    "name": "App Update",
    "channel": "push",
    "body": "New features are available in the app.",
    "status": "active"
  }
  ```

### Invalid Examples

- Missing required `name`
  ```json
  {
    "channel": "email",
    "subject": "Subject",
    "body": "Body text",
    "status": "active"
  }
  ```

- Invalid `channel`
  ```json
  {
    "name": "Invalid Channel",
    "channel": "fax",
    "body": "Body text",
    "status": "active"
  }
  ```

- Email channel missing `subject`
  ```json
  {
    "name": "Missing Subject",
    "channel": "email",
    "body": "Body text",
    "status": "active"
  }
  ```

- Empty `body`
  ```json
  {
    "name": "Empty Body",
    "channel": "sms",
    "body": "",
    "status": "active"
  }
  ```

- Invalid `status`
  ```json
  {
    "name": "Invalid Status",
    "channel": "push",
    "body": "Body text",
    "status": "pending"
  }
  ```

- Duplicate name and channel (if uniqueness enforced)
  ```json
  {
    "name": "Welcome Email",
    "channel": "email",
    "subject": "Duplicate",
    "body": "Body text",
    "status": "active"
  }
  ```

---

This test plan ensures comprehensive coverage of validation, business rules, UI and API interactions, error handling, and edge cases for the Notification Template Management feature.