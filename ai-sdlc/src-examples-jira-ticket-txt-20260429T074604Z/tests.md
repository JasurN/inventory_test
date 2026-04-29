# Test Plan Template

## Unit Tests

- **Validation Logic**
  - `validateTemplate_withValidEmailTemplate_shouldPass`
  - `validateTemplate_withMissingName_shouldReturnError`
  - `validateTemplate_withInvalidChannel_shouldReturnError`
  - `validateTemplate_withMissingBody_shouldReturnError`
  - `validateTemplate_withInvalidStatus_shouldReturnError`
  - `validateTemplate_emailChannelMissingSubject_shouldReturnError`
  - `validateTemplate_nonEmailChannelMissingSubject_shouldPass`
- **Business Rules**
  - `enforceOnlyAdminCanManageTemplates_shouldThrowIfNotAdmin`
  - `enforceOnlyActiveTemplatesUsedForSending_shouldRejectInactiveTemplates`
  - `deactivateTemplate_shouldSetStatusToInactive`
  - `deactivateTemplate_alreadyInactive_shouldBeIdempotent`
- **Mappers**
  - `mapEntityToResponseDTO_shouldMapAllFieldsCorrectly`
  - `mapCreateDTOToEntity_shouldTrimFields`
- **Service Behavior**
  - `createTemplate_withValidData_shouldCreateSuccessfully`
  - `createTemplate_withInvalidData_shouldThrowValidationError`
  - `updateTemplate_existingTemplate_shouldUpdateFields`
  - `updateTemplate_nonExistingTemplate_shouldThrowNotFound`
  - `deactivateTemplate_existingActiveTemplate_shouldDeactivate`
  - `deactivateTemplate_nonExistingTemplate_shouldThrowNotFound`
  - `listTemplates_shouldReturnAllTemplates`

## API Tests

- **GET /notification-templates**
  - `getTemplates_shouldReturn200AndList`
  - `getTemplates_noTemplates_shouldReturnEmptyArray`
  - `getTemplates_unauthenticated_shouldReturn401`
  - `getTemplates_nonAdminUser_shouldReturn403`
- **POST /notification-templates**
  - `createTemplate_validInput_shouldReturn201AndCreatedTemplate`
  - `createTemplate_missingRequiredFields_shouldReturn400WithErrors`
  - `createTemplate_subjectMissingForEmail_shouldReturn400`
  - `createTemplate_invalidChannel_shouldReturn400`
  - `createTemplate_invalidStatus_shouldReturn400`
  - `createTemplate_unauthenticated_shouldReturn401`
  - `createTemplate_nonAdminUser_shouldReturn403`
  - `createTemplate_duplicateNameChannel_shouldReturn409` (if uniqueness enforced)
- **PUT /notification-templates/{id}**
  - `updateTemplate_validInput_shouldReturn200AndUpdatedTemplate`
  - `updateTemplate_missingRequiredFields_shouldReturn400`
  - `updateTemplate_subjectMissingForEmail_shouldReturn400`
  - `updateTemplate_invalidChannel_shouldReturn400`
  - `updateTemplate_invalidStatus_shouldReturn400`
  - `updateTemplate_nonExistingId_shouldReturn404`
  - `updateTemplate_unauthenticated_shouldReturn401`
  - `updateTemplate_nonAdminUser_shouldReturn403`
- **PATCH /notification-templates/{id}/deactivate**
  - `deactivateTemplate_existingActiveTemplate_shouldReturn200AndInactiveStatus`
  - `deactivateTemplate_alreadyInactive_shouldReturn200Idempotent`
  - `deactivateTemplate_nonExistingId_shouldReturn404`
  - `deactivateTemplate_unauthenticated_shouldReturn401`
  - `deactivateTemplate_nonAdminUser_shouldReturn403`
- **Error Handling**
  - `api_shouldReturn500OnUnexpectedError`
  - `api_shouldReturnProperErrorMessagesForValidationFailures`

## E2E Tests

- `admin_canViewListOfTemplates`
- `admin_canCreateNewEmailTemplate_withSubject`
- `admin_cannotCreateEmailTemplate_withoutSubject`
- `admin_canCreateSmsTemplate_withoutSubject`
- `admin_canEditExistingTemplate_andSeeChanges`
- `admin_canDeactivateActiveTemplate_andStatusUpdates`
- `deactivatedTemplate_isNotUsedForSendingNotifications`
- `nonAdminUser_cannotAccessAnyTemplateManagementEndpoints`
- `formValidation_preventsSubmissionWithInvalidData`
- `loadingStates_displayDuringApiCalls`
- `errorStates_displayOnApiFailures`
- `confirmationDialog_appearsOnDeactivate_andCancellingWorks`
- `modal_closesOnCancel_andDiscardsChanges`
- `listPage_showsEmptyState_whenNoTemplatesExist`

## Negative Tests

- `createTemplate_withEmptyName_shouldFailValidation`
- `createTemplate_withInvalidChannel_shouldFailValidation`
- `createTemplate_withEmptyBody_shouldFailValidation`
- `createTemplate_withInvalidStatus_shouldFailValidation`
- `createTemplate_emailChannelMissingSubject_shouldFailValidation`
- `updateTemplate_withInvalidId_shouldReturnNotFound`
- `deactivateTemplate_withInvalidId_shouldReturnNotFound`
- `nonAdminUser_attemptsCreate_shouldReturnForbidden`
- `nonAdminUser_attemptsUpdate_shouldReturnForbidden`
- `nonAdminUser_attemptsDeactivate_shouldReturnForbidden`
- `submitForm_withMissingRequiredFields_shouldShowInlineErrors`
- `submitForm_withApiValidationError_shouldShowErrorMessage`
- `deactivateTemplate_apiFailure_shouldShowErrorMessage`
- `submitForm_withNetworkFailure_shouldAllowRetry`
- `listPage_apiFailure_shouldShowRetryOption`

## Edge Cases

- `createTemplate_withMaxLengthNameSubjectBody` (if max length defined)
- `createTemplate_withWhitespaceOnlyFields_shouldFailValidation`
- `updateTemplate_changeChannelFromEmailToSms_shouldAllowSubjectToBeEmpty`
- `updateTemplate_changeChannelFromSmsToEmail_shouldRequireSubject`
- `deactivateTemplate_multipleTimes_shouldBeIdempotent`
- `listTemplates_withLargeNumberOfTemplates_shouldPaginateOrPerformEfficiently`
- `createTemplate_withBodyContainingRichText_shouldAcceptOrRejectBasedOnSpec` (pending open question)
- `createTemplate_withEmptyList_shouldShowEmptyState`
- `createTemplate_withVeryLargeBody_shouldHandleProperly`
- `deactivateTemplate_whenAnotherUserIsEditing_shouldHandleConcurrency`
- `sendingNotification_withInactiveTemplate_shouldRejectSending`

## Regression Risks

- Validation logic for conditional subject requirement on email channel
- Authorization checks restricting access to admin users only
- Status enforcement: only active templates used for sending notifications
- Deactivation logic and idempotency of deactivate endpoint
- Form validation disabling save button correctly
- Error handling and display of API errors in UI
- Loading and state refresh after create/update/deactivate actions
- Correct mapping of DTOs and entity fields, especially timestamps
- Handling of empty or large data sets in list view

## Suggested Test Data

### Valid Examples

- Email template (active)
  ```json
  {
    "name": "Welcome Email",
    "channel": "email",
    "subject": "Welcome to Our Service",
    "body": "Hello {{user}}, welcome!",
    "status": "active"
  }
  ```
- SMS template (inactive)
  ```json
  {
    "name": "Verification SMS",
    "channel": "sms",
    "body": "Your code is {{code}}",
    "status": "inactive"
  }
  ```
- Push notification template (active)
  ```json
  {
    "name": "App Alert",
    "channel": "push",
    "body": "You have a new alert",
    "status": "active"
  }
  ```

### Invalid Examples

- Missing name
  ```json
  {
    "channel": "email",
    "subject": "Hello",
    "body": "Body text",
    "status": "active"
  }
  ```
- Invalid channel
  ```json
  {
    "name": "Invalid Channel",
    "channel": "fax",
    "body": "Body text",
    "status": "active"
  }
  ```
- Email channel missing subject
  ```json
  {
    "name": "Missing Subject",
    "channel": "email",
    "body": "Body text",
    "status": "active"
  }
  ```
- Empty body
  ```json
  {
    "name": "Empty Body",
    "channel": "sms",
    "body": "",
    "status": "active"
  }
  ```
- Invalid status
  ```json
  {
    "name": "Invalid Status",
    "channel": "push",
    "body": "Body text",
    "status": "pending"
  }
  ```
- Non-admin user token or role for API calls

---

This test plan ensures comprehensive coverage of validation, business rules, UI and API behaviors, error handling, and security constraints for the Notification Template Management feature. It aligns with the provided requirements, UI plan, and API plan.