# React UI Implementation Plan Template

## UI Overview

The Notification Template Management UI allows admin users to manage notification templates used for customer notifications. The workflow consists of:

- **List View**: Admin sees a paginated list of all notification templates with key details (name, channel, status).
- **Create Template**: Admin clicks a button to open a form to create a new template.
- **Edit Template**: Admin selects a template from the list to edit its details.
- **Deactivate Template**: Admin can deactivate an active template via a button in the list or edit view.
- **Validation**: The form validates required fields, including conditional validation (subject required if channel is email).
- **Status Enforcement**: Only active templates are shown as active in the list; deactivated templates are clearly marked.

Screen-level structure:

- **NotificationTemplateListPage**
  - Displays list of templates with filters and actions.
  - Button to create a new template.
- **NotificationTemplateFormModal**
  - Modal dialog for create/edit form.
  - Fields: name, channel (select), subject, body (textarea), status (select).
  - Save and cancel buttons.
- **DeactivateTemplateButton**
  - Button with confirmation dialog to deactivate a template.

## Components

- **NotificationTemplateListPage**
  - Responsible for fetching and displaying the list of templates.
  - Handles loading, empty, and error states.
  - Contains "Create New Template" button.
  - Passes selected template to form modal for editing.
- **NotificationTemplateFormModal**
  - Controlled form component for creating/editing templates.
  - Handles form state, validation, and submission callbacks.
  - Shows validation errors inline.
- **DeactivateTemplateButton**
  - Dumb button component that triggers a confirmation dialog.
  - Calls back to parent to perform deactivation.
- **NotificationTemplateListItem**
  - Displays a single template row with details and action buttons (edit, deactivate).
- **LoadingSpinner**
  - Generic loading indicator.
- **ErrorBanner**
  - Displays API or validation errors.

## Component Props

### NotificationTemplateListPage

```ts
interface NotificationTemplateListPageProps {
  // No props; top-level page component
}
```

### NotificationTemplateFormModal

```ts
interface NotificationTemplateFormModalProps {
  isOpen: boolean;
  template?: NotificationTemplate; // undefined for create
  onClose: () => void;
  onSave: (templateInput: NotificationTemplateInput) => Promise<void>;
  loading: boolean;
  error?: string;
}
```

### DeactivateTemplateButton

```ts
interface DeactivateTemplateButtonProps {
  templateId: string;
  disabled?: boolean;
  onDeactivate: (templateId: string) => Promise<void>;
}
```

### NotificationTemplateListItem

```ts
interface NotificationTemplateListItemProps {
  template: NotificationTemplate;
  onEdit: (template: NotificationTemplate) => void;
  onDeactivate: (templateId: string) => Promise<void>;
  deactivating: boolean;
}
```

## State Management

- **Server State**: Use React Query or SWR to fetch and cache the list of templates (`GET /notification-templates`).
- **Local UI State**:
  - Modal open/close state.
  - Selected template for editing.
  - Form input state inside `NotificationTemplateFormModal`.
  - Loading and error states for create/update/deactivate API calls.
- **Form State**: Managed locally in the form modal with controlled inputs.
- **Loading/Error State**:
  - List loading and error states handled in list page.
  - Form submission loading and error states handled in form modal.
  - Deactivation loading state handled per item.

## Form Validation

- **Name**: Required, non-empty string.
- **Channel**: Required, must be one of "email", "sms", "push".
- **Body**: Required, non-empty string.
- **Status**: Required, must be "active" or "inactive".
- **Subject**: Required if channel is "email", optional otherwise.

Validation behaviors:

- Show inline error messages next to fields.
- Disable Save button if validation fails.
- On submit, prevent API call if validation errors exist.
- If API returns validation errors, display them prominently.

## User Interactions

- **View List**:
  - On page load, fetch and display templates.
  - Show loading spinner while fetching.
  - Show empty state if no templates.
- **Create Template**:
  - Click "Create New Template" opens form modal with empty fields.
  - Fill form, validation runs on change and on submit.
  - Submit calls `POST /notification-templates`.
  - On success, close modal and refresh list.
  - On error, show error message.
- **Edit Template**:
  - Click edit on a list item opens form modal with template data.
  - Modify fields, validation applies.
  - Submit calls `PUT /notification-templates/{id}`.
  - On success, close modal and refresh list.
  - On error, show error message.
- **Deactivate Template**:
  - Click deactivate button triggers confirmation dialog.
  - Confirm calls `PATCH /notification-templates/{id}/deactivate`.
  - On success, refresh list.
  - On error, show error message.
- **Cancel Form**:
  - Close modal without saving discards changes.

## Loading, Empty, and Error States

- **Initial Loading**: Show spinner in list page while fetching templates.
- **Empty List**: Show friendly message "No notification templates found. Create one to get started."
- **Form Validation Errors**: Show inline messages next to invalid fields.
- **API Errors**:
  - List fetch failure: show error banner with retry button.
  - Create/update failure: show error message in form modal.
  - Deactivate failure: show inline error toast or banner.
- **Retry Behavior**:
  - Retry button on list fetch error triggers refetch.
  - Form submission can be retried after fixing errors.
  - Deactivation can be retried after failure.

## Accessibility

- Use semantic HTML elements (e.g., `<form>`, `<label>`, `<button>`, `<table>` or `<ul>` for list).
- Associate labels with inputs via `htmlFor` and `id`.
- Keyboard navigation:
  - Tab order logical and visible focus outlines.
  - Modal traps focus while open.
  - Escape key closes modal.
- Announce validation errors via ARIA live regions.
- Confirmation dialogs use `role="alertdialog"` and focus management.
- Buttons have accessible names.
- Status changes (loading, errors) announced to screen readers.

## Example React TypeScript Code

```tsx
// types.ts
export interface NotificationTemplate {
  id: string;
  name: string;
  channel: "email" | "sms" | "push";
  subject?: string;
  body: string;
  status: "active" | "inactive";
}

export interface NotificationTemplateInput {
  name: string;
  channel: "email" | "sms" | "push";
  subject?: string;
  body: string;
  status: "active" | "inactive";
}

// NotificationTemplateListPage.tsx
import React, { useState } from "react";
import { useQuery, useMutation, useQueryClient } from "react-query";
import {
  NotificationTemplate,
  NotificationTemplateInput,
} from "./types";
import NotificationTemplateFormModal from "./NotificationTemplateFormModal";
import DeactivateTemplateButton from "./DeactivateTemplateButton";
import LoadingSpinner from "./LoadingSpinner";
import ErrorBanner from "./ErrorBanner";

async function fetchTemplates(): Promise<NotificationTemplate[]> {
  const res = await fetch("/notification-templates");
  if (!res.ok) throw new Error("Failed to fetch templates");
  return res.json();
}

async function deactivateTemplate(id: string): Promise<void> {
  const res = await fetch(`/notification-templates/${id}/deactivate`, {
    method: "PATCH",
  });
  if (!res.ok) throw new Error("Failed to deactivate template");
}

export default function NotificationTemplateListPage() {
  const queryClient = useQueryClient();
  const { data, error, isLoading, refetch } = useQuery(
    "notificationTemplates",
    fetchTemplates
  );

  const [editingTemplate, setEditingTemplate] = useState<NotificationTemplate | null>(null);
  const [formOpen, setFormOpen] = useState(false);

  const deactivateMutation = useMutation(deactivateTemplate, {
    onSuccess: () => queryClient.invalidateQueries("notificationTemplates"),
  });

  const openCreateForm = () => {
    setEditingTemplate(null);
    setFormOpen(true);
  };

  const openEditForm = (template: NotificationTemplate) => {
    setEditingTemplate(template);
    setFormOpen(true);
  };

  const closeForm = () => setFormOpen(false);

  const onSave = async (input: NotificationTemplateInput) => {
    const method = editingTemplate ? "PUT" : "POST";
    const url = editingTemplate
      ? `/notification-templates/${editingTemplate.id}`
      : "/notification-templates";

    const res = await fetch(url, {
      method,
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(input),
    });

    if (!res.ok) {
      const errorText = await res.text();
      throw new Error(errorText || "Failed to save template");
    }
    await queryClient.invalidateQueries("notificationTemplates");
    closeForm();
  };

  if (isLoading) return <LoadingSpinner />;

  if (error)
    return (
      <ErrorBanner>
        Error loading templates.{" "}
        <button onClick={() => refetch()}>Retry</button>
      </ErrorBanner>
    );

  return (
    <div>
      <h1>Notification Templates</h1>
      <button onClick={openCreateForm}>Create New Template</button>
      {data && data.length === 0 && <p>No notification templates found.</p>}
      {data && data.length > 0 && (
        <table aria-label="Notification templates list" role="grid">
          <thead>
            <tr>
              <th>Name</th>
              <th>Channel</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {data.map((template) => (
              <tr key={template.id}>
                <td>{template.name}</td>
                <td>{template.channel}</td>
                <td>{template.status}</td>
                <td>
                  <button onClick={() => openEditForm(template)}>Edit</button>
                  {template.status === "active" && (
                    <DeactivateTemplateButton
                      templateId={template.id}
                      disabled={deactivateMutation.isLoading}
                      onDeactivate={async (id) => {
                        try {
                          await deactivateMutation.mutateAsync(id);
                        } catch (e) {
                          alert(
                            `Failed to deactivate template: ${
                              e instanceof Error ? e.message : String(e)
                            }`
                          );
                        }
                      }}
                    />
                  )}
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
      {formOpen && (
        <NotificationTemplateFormModal
          isOpen={formOpen}
          template={editingTemplate ?? undefined}
          onClose={closeForm}
          onSave={onSave}
          loading={false}
        />
      )}
    </div>
  );
}

// NotificationTemplateFormModal.tsx
import React, { useEffect, useState, useRef } from "react";
import { NotificationTemplate, NotificationTemplateInput } from "./types";

interface Props {
  isOpen: boolean;
  template?: NotificationTemplate;
  onClose: () => void;
  onSave: (input: NotificationTemplateInput) => Promise<void>;
  loading: boolean;
  error?: string;
}

interface ValidationErrors {
  name?: string;
  channel?: string;
  subject?: string;
  body?: string;
  status?: string;
}

const CHANNELS = ["email", "sms", "push"] as const;
const STATUSES = ["active", "inactive"] as const;

export default function NotificationTemplateFormModal({
  isOpen,
  template,
  onClose,
  onSave,
  loading,
  error,
}: Props) {
  const [name, setName] = useState(template?.name ?? "");
  const [channel, setChannel] = useState<typeof CHANNELS[number]>(
    template?.channel ?? "email"
  );
  const [subject, setSubject] = useState(template?.subject ?? "");
  const [body, setBody] = useState(template?.body ?? "");
  const [status, setStatus] = useState<typeof STATUSES[number]>(
    template?.status ?? "active"
  );
  const [validationErrors, setValidationErrors] = useState<ValidationErrors>(
    {}
  );
  const [submitError, setSubmitError] = useState<string | undefined>(error);

  const firstInputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    if (isOpen) {
      setName(template?.name ?? "");
      setChannel(template?.channel ?? "email");
      setSubject(template?.subject ?? "");
      setBody(template?.body ?? "");
      setStatus(template?.status ?? "active");
      setValidationErrors({});
      setSubmitError(undefined);
      setTimeout(() => firstInputRef.current?.focus(), 0);
    }
  }, [isOpen, template]);

  function validate(): ValidationErrors {
    const errors: ValidationErrors = {};
    if (!name.trim()) errors.name = "Name is required";
    if (!channel || !CHANNELS.includes(channel)) errors.channel = "Channel is required";
    if (!body.trim()) errors.body = "Body is required";
    if (!status || !STATUSES.includes(status)) errors.status = "Status is required";
    if (channel === "email" && !subject.trim()) errors.subject = "Subject is required for email channel";
    return errors;
  }

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const errors = validate();
    setValidationErrors(errors);
    if (Object.keys(errors).length > 0) return;

    setSubmitError(undefined);
    try {
      await onSave({ name, channel, subject: subject.trim() || undefined, body, status });
    } catch (e) {
      setSubmitError(e instanceof Error ? e.message : String(e));
    }
  };

  if (!isOpen) return null;

  return (
    <div
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      className="modal-overlay"
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose();
      }}
    >
      <form className="modal-content" onSubmit={handleSubmit} noValidate>
        <h2 id="modal-title">{template ? "Edit" : "Create"} Notification Template</h2>
        <div>
          <label htmlFor="name">Name<span aria-hidden="true">*</span></label>
          <input
            id="name"
            type="text"
            ref={firstInputRef}
            value={name}
            onChange={(e) => setName(e.target.value)}
            aria-invalid={!!validationErrors.name}
            aria-describedby={validationErrors.name ? "name-error" : undefined}
            required
          />
          {validationErrors.name && (
            <div id="name-error" role="alert" className="error">
              {validationErrors.name}
            </div>
          )}
        </div>

        <div>
          <label htmlFor="channel">Channel<span aria-hidden="true">*</span></label>
          <select
            id="channel"
            value={channel}
            onChange={(e) => setChannel(e.target.value as typeof CHANNELS[number])}
            aria-invalid={!!validationErrors.channel}
            aria-describedby={validationErrors.channel ? "channel-error" : undefined}
            required
          >
            {CHANNELS.map((ch) => (
              <option key={ch} value={ch}>
                {ch.toUpperCase()}
              </option>
            ))}
          </select>
          {validationErrors.channel && (
            <div id="channel-error" role="alert" className="error">
              {validationErrors.channel}
            </div>
          )}
        </div>

        <div>
          <label htmlFor="subject">
            Subject{channel === "email" && <span aria-hidden="true">*</span>}
          </label>
          <input
            id="subject"
            type="text"
            value={subject}
            onChange={(e) => setSubject(e.target.value)}
            aria-invalid={!!validationErrors.subject}
            aria-describedby={validationErrors.subject ? "subject-error" : undefined}
            required={channel === "email"}
            disabled={channel !== "email"}
          />
          {validationErrors.subject && (
            <div id="subject-error" role="alert" className="error">
              {validationErrors.subject}
            </div>
          )}
        </div>

        <div>
          <label htmlFor="body">Body<span aria-hidden="true">*</span></label>
          <textarea
            id="body"
            value={body}
            onChange={(e) => setBody(e.target.value)}
            aria-invalid={!!validationErrors.body}
            aria-describedby={validationErrors.body ? "body-error" : undefined}
            required
            rows={6}
          />
          {validationErrors.body && (
            <div id="body-error" role="alert" className="error">
              {validationErrors.body}
            </div>
          )}
        </div>

        <div>
          <label htmlFor="status">Status<span aria-hidden="true">*</span></label>
          <select
            id="status"
            value={status}
            onChange={(e) => setStatus(e.target.value as typeof STATUSES[number])}
            aria-invalid={!!validationErrors.status}
            aria-describedby={validationErrors.status ? "status-error" : undefined}
            required
          >
            {STATUSES.map((st) => (
              <option key={st} value={st}>
                {st.charAt(0).toUpperCase() + st.slice(1)}
              </option>
            ))}
          </select>
          {validationErrors.status && (
            <div id="status-error" role="alert" className="error">
              {validationErrors.status}
            </div>
          )}
        </div>

        {submitError && (
          <div role="alert" className="error submit-error">
            {submitError}
          </div>
        )}

        <div className="modal-actions">
          <button type="submit" disabled={loading}>
            {loading ? "Saving..." : "Save"}
          </button>
          <button type="button" onClick={onClose} disabled={loading}>
            Cancel
          </button>
        </div>
      </form>
    </div>
  );
}

// DeactivateTemplateButton.tsx
import React, { useState } from "react";

interface Props {
  templateId: string;
  disabled?: boolean;
  onDeactivate: (templateId: string) => Promise<void>;
}

export default function DeactivateTemplateButton({
  templateId,
  disabled,
  onDeactivate,
}: Props) {
  const [confirming, setConfirming] = useState(false);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleClick = () => setConfirming(true);
  const handleCancel = () => {
    setConfirming(false);
    setError(null);
  };

  const handleConfirm = async () => {
    setLoading(true);
    setError(null);
    try {
      await onDeactivate(templateId);
      setConfirming(false);
    } catch (e) {
      setError(e instanceof Error ? e.message : String(e));
    } finally {
      setLoading(false);
    }
  };

  if (confirming) {
    return (
      <span role="alertdialog" aria-modal="true" aria-labelledby="confirm-title">
        <p id="confirm-title">Confirm Deactivation</p>
        <p>Are you sure you want to deactivate this template?</p>
        {error && <p role="alert" className="error">{error}</p>}
        <button onClick={handleConfirm} disabled={loading}>
          {loading ? "Deactivating..." : "Yes, Deactivate"}
        </button>
        <button onClick={handleCancel} disabled={loading}>
          Cancel
        </button>
      </span>
    );
  }

  return (
    <button onClick={handleClick} disabled={disabled}>
      Deactivate
    </button>
  );
}
```

---

This plan and example code provide a robust, accessible, and maintainable React UI implementation for Notification Template Management, following enterprise best practices and the given requirements.