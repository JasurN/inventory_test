# React UI Implementation Plan Template

## UI Overview

The Notification Template Management feature enables admin users to manage notification templates used for customer notifications. The workflow consists of:

- **List View:** Admin sees a paginated list of all notification templates with key details (name, channel, status).
- **Create/Edit Form:** Admin can create a new template or edit an existing one by filling in name, channel, subject, body, and status.
- **Activate/Deactivate:** Admin can toggle the status of a template between active and inactive via a button or toggle control.
- **Validation:** Form enforces required fields and conditional rules (e.g., subject required if channel is email).
- **Confirmation:** Deactivation requires confirmation to prevent accidental disabling.
- **Error Handling:** API errors and validation errors are surfaced clearly.
- **Access Control:** Only admin users can access and perform these actions.

## Components

- **NotificationTemplateList**
  - Displays list of templates with columns: Name, Channel, Status, Actions (Edit, Activate/Deactivate).
  - Handles loading, empty, and error states.
  - Triggers edit and deactivate actions via callbacks.

- **NotificationTemplateForm**
  - Form for creating or editing a notification template.
  - Inputs: Name, Channel (select), Subject, Body (textarea), Status (select).
  - Handles form validation and submission.
  - Shows validation errors inline.
  - Supports both create and edit modes.

- **ActivateDeactivateButton**
  - Button or toggle to activate or deactivate a template.
  - Shows current status.
  - Confirms deactivation with a modal dialog.
  - Disabled during API calls.

- **ConfirmationDialog**
  - Generic reusable confirmation modal for deactivation confirmation.

- **ErrorBanner**
  - Displays API or unexpected errors at the top of the screen.

## Component Props

- **NotificationTemplateList**
  ```ts
  interface NotificationTemplateListProps {
    templates: NotificationTemplate[];
    loading: boolean;
    error: string | null;
    onEdit: (template: NotificationTemplate) => void;
    onToggleStatus: (template: NotificationTemplate) => Promise<void>;
  }
  ```

- **NotificationTemplateForm**
  ```ts
  interface NotificationTemplateFormProps {
    initialData?: NotificationTemplate;
    onSubmit: (data: NotificationTemplateFormData) => Promise<void>;
    onCancel: () => void;
  }

  interface NotificationTemplateFormData {
    name: string;
    channel: 'email' | 'sms' | 'push';
    subject?: string;
    body: string;
    status: 'active' | 'inactive';
  }
  ```

- **ActivateDeactivateButton**
  ```ts
  interface ActivateDeactivateButtonProps {
    status: 'active' | 'inactive';
    onActivate: () => Promise<void>;
    onDeactivate: () => Promise<void>;
    disabled?: boolean;
  }
  ```

- **ConfirmationDialog**
  ```ts
  interface ConfirmationDialogProps {
    isOpen: boolean;
    title: string;
    message: string;
    onConfirm: () => void;
    onCancel: () => void;
  }
  ```

- **ErrorBanner**
  ```ts
  interface ErrorBannerProps {
    message: string;
    onClose: () => void;
  }
  ```

## State Management

- **Local state:**
  - Form input values and validation errors managed inside `NotificationTemplateForm`.
  - Confirmation dialog open/close state in `ActivateDeactivateButton` or parent.
  - Loading and error states for API calls in container components.

- **Server state:**
  - List of notification templates fetched from `GET /notification-templates`.
  - Updates via POST, PUT, PATCH endpoints trigger refetch or local cache update.

- **Loading/Error state:**
  - Loading spinners shown during API calls.
  - Error banners for API failures.
  - Inline validation errors on form fields.

- **Form state:**
  - Controlled inputs with validation on change and on submit.
  - Conditional validation for subject field based on channel.

## Form Validation

| Validation Rule                              | User-facing Behavior                                      |
|----------------------------------------------|-----------------------------------------------------------|
| Name, channel, body, status are required     | Show inline error if empty on blur or submit              |
| Subject required if channel is email         | Show inline error if empty when channel is email          |
| Status must be active or inactive             | Enforced by select input options                          |
| Only active templates used for sending        | Not editable here, but status toggle disables sending     |

Validation runs on form submit and on blur for each field. Errors are announced and visually highlighted.

## User Interactions

- **Create:**
  - Admin clicks "Create Template" button.
  - `NotificationTemplateForm` opens empty.
  - Admin fills form, submits.
  - On success, list refreshes and form closes.

- **Read:**
  - On page load, list fetches templates.
  - Shows loading spinner, error banner, or empty state as appropriate.

- **Update:**
  - Admin clicks "Edit" on a template row.
  - `NotificationTemplateForm` opens with pre-filled data.
  - Admin edits and submits.
  - On success, list refreshes and form closes.

- **Activate/Deactivate:**
  - Admin clicks toggle or button in list row.
  - If deactivating, confirmation dialog appears.
  - On confirm, PATCH request sent.
  - On success, list updates status.

- **Confirmation:**
  - Deactivation requires confirmation modal.
  - Cancel closes modal without action.

## Loading, Empty, and Error States

- **Initial loading:** Spinner in list area while fetching templates.
- **Empty list:** Show friendly message "No notification templates found. Create one to get started."
- **Validation errors:** Inline messages next to fields, focus moves to first error.
- **API failures:** ErrorBanner with retry button for list fetch or form submit.
- **Retry behavior:** Retry button refetches list or resubmits form.

## Accessibility

- All form inputs have associated labels.
- Keyboard navigation supported for list, form, buttons, and dialogs.
- Focus moves logically: on open form, focus first input; on error, focus first error field.
- Confirmation dialog traps focus and is announced by screen readers.
- Error messages are linked to inputs via aria-describedby.
- Semantic HTML: tables for list, form elements with fieldsets if needed.
- Buttons have accessible names and roles.

## Example React TypeScript Code

```tsx
// types.ts
export interface NotificationTemplate {
  id: string;
  name: string;
  channel: 'email' | 'sms' | 'push';
  subject?: string;
  body: string;
  status: 'active' | 'inactive';
}

// NotificationTemplateList.tsx
import React from 'react';

interface NotificationTemplateListProps {
  templates: NotificationTemplate[];
  loading: boolean;
  error: string | null;
  onEdit: (template: NotificationTemplate) => void;
  onToggleStatus: (template: NotificationTemplate) => Promise<void>;
}

export const NotificationTemplateList: React.FC<NotificationTemplateListProps> = ({
  templates,
  loading,
  error,
  onEdit,
  onToggleStatus,
}) => {
  if (loading) return <p>Loading templates...</p>;
  if (error) return <p role="alert">Error loading templates: {error}</p>;
  if (templates.length === 0) return <p>No notification templates found.</p>;

  return (
    <table aria-label="Notification Templates">
      <thead>
        <tr>
          <th>Name</th>
          <th>Channel</th>
          <th>Status</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        {templates.map((t) => (
          <tr key={t.id}>
            <td>{t.name}</td>
            <td>{t.channel}</td>
            <td>{t.status}</td>
            <td>
              <button onClick={() => onEdit(t)} aria-label={`Edit template ${t.name}`}>
                Edit
              </button>
              <ActivateDeactivateButton
                status={t.status}
                onActivate={() => onToggleStatus({ ...t, status: 'active' })}
                onDeactivate={() => onToggleStatus({ ...t, status: 'inactive' })}
              />
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
};

// ActivateDeactivateButton.tsx
import React, { useState } from 'react';

interface ActivateDeactivateButtonProps {
  status: 'active' | 'inactive';
  onActivate: () => Promise<void>;
  onDeactivate: () => Promise<void>;
  disabled?: boolean;
}

export const ActivateDeactivateButton: React.FC<ActivateDeactivateButtonProps> = ({
  status,
  onActivate,
  onDeactivate,
  disabled = false,
}) => {
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [loading, setLoading] = useState(false);

  const handleDeactivate = async () => {
    setLoading(true);
    try {
      await onDeactivate();
    } finally {
      setLoading(false);
      setConfirmOpen(false);
    }
  };

  if (status === 'active') {
    return (
      <>
        <button
          onClick={() => setConfirmOpen(true)}
          disabled={disabled || loading}
          aria-label="Deactivate template"
        >
          Deactivate
        </button>
        {confirmOpen && (
          <ConfirmationDialog
            isOpen={confirmOpen}
            title="Confirm Deactivation"
            message="Are you sure you want to deactivate this template? It will no longer be used for notifications."
            onConfirm={handleDeactivate}
            onCancel={() => setConfirmOpen(false)}
          />
        )}
      </>
    );
  } else {
    return (
      <button onClick={onActivate} disabled={disabled || loading} aria-label="Activate template">
        Activate
      </button>
    );
  }
};

// ConfirmationDialog.tsx
import React, { useEffect, useRef } from 'react';

interface ConfirmationDialogProps {
  isOpen: boolean;
  title: string;
  message: string;
  onConfirm: () => void;
  onCancel: () => void;
}

export const ConfirmationDialog: React.FC<ConfirmationDialogProps> = ({
  isOpen,
  title,
  message,
  onConfirm,
  onCancel,
}) => {
  const dialogRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen && dialogRef.current) {
      dialogRef.current.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="confirm-dialog-title"
      aria-describedby="confirm-dialog-desc"
      tabIndex={-1}
      ref={dialogRef}
      style={{
        position: 'fixed',
        top: '30%',
        left: '50%',
        transform: 'translateX(-50%)',
        backgroundColor: 'white',
        padding: '1rem',
        border: '1px solid black',
        zIndex: 1000,
      }}
    >
      <h2 id="confirm-dialog-title">{title}</h2>
      <p id="confirm-dialog-desc">{message}</p>
      <button onClick={onConfirm}>Confirm</button>
      <button onClick={onCancel}>Cancel</button>
    </div>
  );
};

// NotificationTemplateForm.tsx
import React, { useState, useEffect, FormEvent } from 'react';

interface NotificationTemplateFormProps {
  initialData?: NotificationTemplate;
  onSubmit: (data: NotificationTemplateFormData) => Promise<void>;
  onCancel: () => void;
}

interface NotificationTemplateFormData {
  name: string;
  channel: 'email' | 'sms' | 'push';
  subject?: string;
  body: string;
  status: 'active' | 'inactive';
}

export const NotificationTemplateForm: React.FC<NotificationTemplateFormProps> = ({
  initialData,
  onSubmit,
  onCancel,
}) => {
  const [formData, setFormData] = useState<NotificationTemplateFormData>({
    name: initialData?.name || '',
    channel: initialData?.channel || 'email',
    subject: initialData?.subject || '',
    body: initialData?.body || '',
    status: initialData?.status || 'inactive',
  });

  const [errors, setErrors] = useState<Record<string, string>>({});
  const [submitting, setSubmitting] = useState(false);

  // Validate fields and set errors
  const validate = (): boolean => {
    const newErrors: Record<string, string> = {};
    if (!formData.name.trim()) newErrors.name = 'Name is required.';
    if (!formData.channel) newErrors.channel = 'Channel is required.';
    if (!formData.body.trim()) newErrors.body = 'Body is required.';
    if (!formData.status) newErrors.status = 'Status is required.';
    if (formData.channel === 'email' && !formData.subject?.trim())
      newErrors.subject = 'Subject is required for email channel.';
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleChange = (
    e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement | HTMLTextAreaElement>
  ) => {
    const { name, value } = e.target;
    setFormData((prev) => ({
      ...prev,
      [name]: value,
    }));
    setErrors((prev) => ({ ...prev, [name]: undefined }));
  };

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    if (!validate()) return;
    setSubmitting(true);
    try {
      await onSubmit(formData);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} aria-label={initialData ? 'Edit Notification Template' : 'Create Notification Template'}>
      <div>
        <label htmlFor="name">Name *</label>
        <input
          id="name"
          name="name"
          value={formData.name}
          onChange={handleChange}
          aria-invalid={!!errors.name}
          aria-describedby={errors.name ? 'name-error' : undefined}
          disabled={submitting}
        />
        {errors.name && (
          <span role="alert" id="name-error" style={{ color: 'red' }}>
            {errors.name}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="channel">Channel *</label>
        <select
          id="channel"
          name="channel"
          value={formData.channel}
          onChange={handleChange}
          aria-invalid={!!errors.channel}
          aria-describedby={errors.channel ? 'channel-error' : undefined}
          disabled={submitting}
        >
          <option value="email">Email</option>
          <option value="sms">SMS</option>
          <option value="push">Push</option>
        </select>
        {errors.channel && (
          <span role="alert" id="channel-error" style={{ color: 'red' }}>
            {errors.channel}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="subject">
          Subject {formData.channel === 'email' ? '*' : '(optional)'}
        </label>
        <input
          id="subject"
          name="subject"
          value={formData.subject}
          onChange={handleChange}
          aria-invalid={!!errors.subject}
          aria-describedby={errors.subject ? 'subject-error' : undefined}
          disabled={submitting}
        />
        {errors.subject && (
          <span role="alert" id="subject-error" style={{ color: 'red' }}>
            {errors.subject}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="body">Body *</label>
        <textarea
          id="body"
          name="body"
          value={formData.body}
          onChange={handleChange}
          aria-invalid={!!errors.body}
          aria-describedby={errors.body ? 'body-error' : undefined}
          disabled={submitting}
          rows={5}
        />
        {errors.body && (
          <span role="alert" id="body-error" style={{ color: 'red' }}>
            {errors.body}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="status">Status *</label>
        <select
          id="status"
          name="status"
          value={formData.status}
          onChange={handleChange}
          aria-invalid={!!errors.status}
          aria-describedby={errors.status ? 'status-error' : undefined}
          disabled={submitting}
        >
          <option value="active">Active</option>
          <option value="inactive">Inactive</option>
        </select>
        {errors.status && (
          <span role="alert" id="status-error" style={{ color: 'red' }}>
            {errors.status}
          </span>
        )}
      </div>

      <div>
        <button type="submit" disabled={submitting}>
          {initialData ? 'Update Template' : 'Create Template'}
        </button>
        <button type="button" onClick={onCancel} disabled={submitting}>
          Cancel
        </button>
      </div>
    </form>
  );
};
```

---

This plan and example code provide a clear, maintainable, and accessible React UI implementation for Notification Template Management, following enterprise best practices and the given requirements.