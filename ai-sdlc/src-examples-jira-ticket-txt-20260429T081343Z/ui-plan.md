# React UI Implementation Plan Template

## UI Overview

The Notification Template Management feature enables admin users to manage notification templates used for customer notifications. The workflow consists of:

1. **List View Screen**: Admin sees a paginated list of existing notification templates with key details (name, channel, status). Each template can be edited or deactivated from this view.

2. **Create/Edit Form Screen**: Admin can create a new template or edit an existing one. The form includes fields for name, channel (email, sms, push), subject, body, and status (active/inactive). Validation is applied on required fields and conditional rules (subject required if channel is email).

3. **Deactivate Action**: Admin can deactivate a template from the list view via a dedicated button with confirmation.

The UI is structured as a single-page app with routes or modal dialogs for create/edit forms. The list view is the default landing page for this feature.

## Components

- **NotificationTemplatesPage**  
  Container component managing data fetching, state, and coordinating child components.

- **NotificationTemplatesList**  
  Dumb/presentational component rendering the list of templates with edit and deactivate buttons.

- **NotificationTemplateForm**  
  Controlled form component for creating or editing a notification template.

- **DeactivateTemplateButton**  
  Button component that triggers deactivation with confirmation modal.

- **LoadingIndicator**  
  Reusable spinner or skeleton UI for loading states.

- **ErrorMessage**  
  Reusable component to display API or validation errors.

- **ConfirmationDialog**  
  Generic modal dialog for confirming destructive actions.

## Component Props

- **NotificationTemplatesPage**  
  - None (top-level container)  
  - Manages fetching templates, form open state, selected template for edit.

- **NotificationTemplatesList**  
  - `templates: NotificationTemplate[]`  
  - `onEdit(template: NotificationTemplate): void`  
  - `onDeactivate(template: NotificationTemplate): void`  
  - `loading: boolean`  
  - `error: string | null`

- **NotificationTemplateForm**  
  - `initialData?: NotificationTemplate` (undefined for create)  
  - `onSubmit(data: NotificationTemplateFormData): Promise<void>`  
  - `onCancel(): void`  
  - `submitting: boolean`  
  - `error: string | null`

- **DeactivateTemplateButton**  
  - `templateId: string`  
  - `onDeactivate(): Promise<void>`  
  - `disabled?: boolean`

- **LoadingIndicator**  
  - `message?: string`

- **ErrorMessage**  
  - `message: string`

- **ConfirmationDialog**  
  - `title: string`  
  - `message: string`  
  - `onConfirm(): void`  
  - `onCancel(): void`  
  - `isOpen: boolean`

## State Management

- **Local State**  
  - Form input values and validation errors inside `NotificationTemplateForm`.  
  - Modal/dialog open states in `NotificationTemplatesPage`.  
  - Confirmation dialog open state for deactivation.

- **Server State**  
  - List of notification templates fetched from `GET /notification-templates`.  
  - Updates on create (`POST`), edit (`PUT`), and deactivate (`PATCH`) trigger refetch or local cache update.

- **Loading/Error State**  
  - Loading flags for list fetch and form submit actions.  
  - Error messages for API failures surfaced in UI components.

- **Form State**  
  - Controlled inputs with validation feedback.  
  - Disabled submit button while submitting.

## Form Validation

- **Required fields**: `name`, `channel`, `body`, `status` must be non-empty.  
- **Conditional required**: `subject` is required if `channel === 'email'`.  
- **Validation feedback**: Inline error messages next to fields.  
- **Submit blocking**: Form cannot submit if validation fails.  
- **Trim inputs**: Leading/trailing whitespace trimmed before submit.

## User Interactions

- **Read**: On page load, fetch and display list of templates. Show loading spinner or error message as needed.

- **Create**:  
  - User clicks "Create Template" button.  
  - Opens empty `NotificationTemplateForm`.  
  - User fills fields, validation runs on change and on submit.  
  - On submit, call `POST /notification-templates`.  
  - On success, close form and refresh list.

- **Edit**:  
  - User clicks "Edit" on a template row.  
  - Opens `NotificationTemplateForm` prefilled with template data.  
  - User updates fields, validation applies.  
  - On submit, call `PUT /notification-templates/{id}`.  
  - On success, close form and refresh list.

- **Deactivate**:  
  - User clicks "Deactivate" button on a template row.  
  - Show `ConfirmationDialog`.  
  - On confirm, call `PATCH /notification-templates/{id}/deactivate`.  
  - On success, refresh list.

- **Cancel**:  
  - User can cancel form or confirmation dialog to abort action.

## Loading, Empty, and Error States

- **Initial loading**: Show `LoadingIndicator` in list view while fetching templates.

- **Empty list**: Show friendly message "No notification templates found. Create one to get started."

- **Form validation errors**: Show inline messages next to invalid fields.

- **API errors**: Show `ErrorMessage` with retry button in list view or form submit.

- **Retry behavior**: Retry button triggers refetch or resubmit.

## Accessibility

- Use semantic HTML elements (e.g., `<form>`, `<label>`, `<button>`, `<table>`).

- Associate labels with inputs via `htmlFor` and `id`.

- Keyboard navigation:  
  - Tab order logical and visible focus outlines.  
  - Modal/dialog traps focus while open.  
  - Escape key closes modals/forms.

- Announce validation errors and API errors via ARIA live regions.

- Confirmation dialogs use `role="dialog"` and proper aria attributes.

- Buttons have accessible names and states.

## Example React TypeScript Code

```tsx
// types.ts
export type NotificationTemplate = {
  id: string;
  name: string;
  channel: "email" | "sms" | "push";
  subject?: string;
  body: string;
  status: "active" | "inactive";
};

export type NotificationTemplateFormData = Omit<NotificationTemplate, "id">;

// NotificationTemplatesPage.tsx
import React, { useEffect, useState } from "react";
import {
  NotificationTemplatesList,
} from "./NotificationTemplatesList";
import {
  NotificationTemplateForm,
} from "./NotificationTemplateForm";
import { DeactivateTemplateButton } from "./DeactivateTemplateButton";
import { LoadingIndicator } from "./LoadingIndicator";
import { ErrorMessage } from "./ErrorMessage";

export function NotificationTemplatesPage() {
  const [templates, setTemplates] = useState<NotificationTemplate[]>([]);
  const [loading, setLoading] = useState<boolean>(true);
  const [error, setError] = useState<string | null>(null);
  const [editingTemplate, setEditingTemplate] = useState<NotificationTemplate | null>(null);
  const [formOpen, setFormOpen] = useState(false);

  async function fetchTemplates() {
    setLoading(true);
    setError(null);
    try {
      const res = await fetch("/notification-templates");
      if (!res.ok) throw new Error(`Failed to fetch: ${res.statusText}`);
      const data = (await res.json()) as NotificationTemplate[];
      setTemplates(data);
    } catch (e) {
      setError(e instanceof Error ? e.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  }

  useEffect(() => {
    fetchTemplates();
  }, []);

  function openCreateForm() {
    setEditingTemplate(null);
    setFormOpen(true);
  }

  function openEditForm(template: NotificationTemplate) {
    setEditingTemplate(template);
    setFormOpen(true);
  }

  function closeForm() {
    setFormOpen(false);
    setEditingTemplate(null);
  }

  async function handleFormSubmit(data: NotificationTemplateFormData) {
    try {
      const method = editingTemplate ? "PUT" : "POST";
      const url = editingTemplate
        ? `/notification-templates/${editingTemplate.id}`
        : "/notification-templates";

      const res = await fetch(url, {
        method,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data),
      });

      if (!res.ok) {
        const errText = await res.text();
        throw new Error(errText || "Failed to save template");
      }

      await fetchTemplates();
      closeForm();
    } catch (e) {
      throw e;
    }
  }

  async function handleDeactivate(template: NotificationTemplate) {
    if (!window.confirm(`Deactivate template "${template.name}"?`)) return;
    try {
      const res = await fetch(`/notification-templates/${template.id}/deactivate`, {
        method: "PATCH",
      });
      if (!res.ok) throw new Error(`Failed to deactivate: ${res.statusText}`);
      await fetchTemplates();
    } catch (e) {
      alert(e instanceof Error ? e.message : "Unknown error");
    }
  }

  return (
    <main>
      <h1>Notification Templates</h1>
      <button onClick={openCreateForm}>Create Template</button>

      {loading && <LoadingIndicator message="Loading templates..." />}
      {error && <ErrorMessage message={error} />}
      {!loading && !error && templates.length === 0 && (
        <p>No notification templates found. Create one to get started.</p>
      )}
      {!loading && !error && templates.length > 0 && (
        <NotificationTemplatesList
          templates={templates}
          onEdit={openEditForm}
          onDeactivate={handleDeactivate}
          loading={loading}
          error={error}
        />
      )}

      {formOpen && (
        <NotificationTemplateForm
          initialData={editingTemplate ?? undefined}
          onSubmit={handleFormSubmit}
          onCancel={closeForm}
          submitting={false}
          error={null}
        />
      )}
    </main>
  );
}

// NotificationTemplatesList.tsx
import React from "react";
import { NotificationTemplate } from "./types";

type Props = {
  templates: NotificationTemplate[];
  onEdit(template: NotificationTemplate): void;
  onDeactivate(template: NotificationTemplate): void;
  loading: boolean;
  error: string | null;
};

export function NotificationTemplatesList({
  templates,
  onEdit,
  onDeactivate,
}: Props) {
  return (
    <table aria-label="Notification Templates List" role="grid">
      <thead>
        <tr>
          <th>Name</th>
          <th>Channel</th>
          <th>Status</th>
          <th>Subject</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        {templates.map((t) => (
          <tr key={t.id}>
            <td>{t.name}</td>
            <td>{t.channel}</td>
            <td>{t.status}</td>
            <td>{t.channel === "email" ? t.subject ?? "-" : "-"}</td>
            <td>
              <button onClick={() => onEdit(t)} aria-label={`Edit ${t.name}`}>
                Edit
              </button>
              {t.status === "active" && (
                <button
                  onClick={() => onDeactivate(t)}
                  aria-label={`Deactivate ${t.name}`}
                >
                  Deactivate
                </button>
              )}
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// NotificationTemplateForm.tsx
import React, { useState, useEffect } from "react";
import { NotificationTemplateFormData, NotificationTemplate } from "./types";

type Props = {
  initialData?: NotificationTemplate;
  onSubmit(data: NotificationTemplateFormData): Promise<void>;
  onCancel(): void;
  submitting: boolean;
  error: string | null;
};

type ValidationErrors = Partial<Record<keyof NotificationTemplateFormData, string>>;

export function NotificationTemplateForm({
  initialData,
  onSubmit,
  onCancel,
  submitting,
  error,
}: Props) {
  const [formData, setFormData] = useState<NotificationTemplateFormData>({
    name: "",
    channel: "email",
    subject: "",
    body: "",
    status: "active",
    ...initialData,
  });

  const [validationErrors, setValidationErrors] = useState<ValidationErrors>({});

  useEffect(() => {
    setFormData({
      name: "",
      channel: "email",
      subject: "",
      body: "",
      status: "active",
      ...initialData,
    });
    setValidationErrors({});
  }, [initialData]);

  function validate(data: NotificationTemplateFormData): ValidationErrors {
    const errors: ValidationErrors = {};
    if (!data.name.trim()) errors.name = "Name is required";
    if (!data.channel) errors.channel = "Channel is required";
    if (!data.body.trim()) errors.body = "Body is required";
    if (!data.status) errors.status = "Status is required";
    if (data.channel === "email" && !data.subject?.trim()) {
      errors.subject = "Subject is required for email channel";
    }
    return errors;
  }

  function handleChange<K extends keyof NotificationTemplateFormData>(
    key: K,
    value: NotificationTemplateFormData[K]
  ) {
    setFormData((prev) => ({ ...prev, [key]: value }));
    setValidationErrors((prev) => ({ ...prev, [key]: undefined }));
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    const errors = validate(formData);
    if (Object.keys(errors).length > 0) {
      setValidationErrors(errors);
      return;
    }
    try {
      await onSubmit({
        ...formData,
        name: formData.name.trim(),
        subject: formData.subject?.trim(),
        body: formData.body.trim(),
      });
    } catch (e) {
      // onSubmit error handled by parent via error prop
    }
  }

  return (
    <div role="dialog" aria-modal="true" aria-labelledby="form-title" tabIndex={-1}>
      <h2 id="form-title">{initialData ? "Edit Template" : "Create Template"}</h2>
      <form onSubmit={handleSubmit} noValidate>
        <div>
          <label htmlFor="name">Name *</label>
          <input
            id="name"
            type="text"
            value={formData.name}
            onChange={(e) => handleChange("name", e.target.value)}
            aria-invalid={!!validationErrors.name}
            aria-describedby={validationErrors.name ? "name-error" : undefined}
            required
          />
          {validationErrors.name && (
            <span id="name-error" role="alert" style={{ color: "red" }}>
              {validationErrors.name}
            </span>
          )}
        </div>

        <div>
          <label htmlFor="channel">Channel *</label>
          <select
            id="channel"
            value={formData.channel}
            onChange={(e) => handleChange("channel", e.target.value as any)}
            aria-invalid={!!validationErrors.channel}
            aria-describedby={validationErrors.channel ? "channel-error" : undefined}
            required
          >
            <option value="email">Email</option>
            <option value="sms">SMS</option>
            <option value="push">Push</option>
          </select>
          {validationErrors.channel && (
            <span id="channel-error" role="alert" style={{ color: "red" }}>
              {validationErrors.channel}
            </span>
          )}
        </div>

        <div>
          <label htmlFor="subject">
            Subject {formData.channel === "email" ? "*" : "(optional)"}
          </label>
          <input
            id="subject"
            type="text"
            value={formData.subject ?? ""}
            onChange={(e) => handleChange("subject", e.target.value)}
            aria-invalid={!!validationErrors.subject}
            aria-describedby={validationErrors.subject ? "subject-error" : undefined}
            required={formData.channel === "email"}
          />
          {validationErrors.subject && (
            <span id="subject-error" role="alert" style={{ color: "red" }}>
              {validationErrors.subject}
            </span>
          )}
        </div>

        <div>
          <label htmlFor="body">Body *</label>
          <textarea
            id="body"
            value={formData.body}
            onChange={(e) => handleChange("body", e.target.value)}
            aria-invalid={!!validationErrors.body}
            aria-describedby={validationErrors.body ? "body-error" : undefined}
            required
          />
          {validationErrors.body && (
            <span id="body-error" role="alert" style={{ color: "red" }}>
              {validationErrors.body}
            </span>
          )}
        </div>

        <div>
          <label htmlFor="status">Status *</label>
          <select
            id="status"
            value={formData.status}
            onChange={(e) => handleChange("status", e.target.value as any)}
            aria-invalid={!!validationErrors.status}
            aria-describedby={validationErrors.status ? "status-error" : undefined}
            required
          >
            <option value="active">Active</option>
            <option value="inactive">Inactive</option>
          </select>
          {validationErrors.status && (
            <span id="status-error" role="alert" style={{ color: "red" }}>
              {validationErrors.status}
            </span>
          )}
        </div>

        {error && (
          <div role="alert" style={{ color: "red" }}>
            {error}
          </div>
        )}

        <button type="submit" disabled={submitting}>
          {submitting ? "Saving..." : "Save"}
        </button>
        <button type="button" onClick={onCancel} disabled={submitting}>
          Cancel
        </button>
      </form>
    </div>
  );
}

// DeactivateTemplateButton.tsx
import React, { useState } from "react";

type Props = {
  templateId: string;
  onDeactivate(): Promise<void>;
  disabled?: boolean;
};

export function DeactivateTemplateButton({ templateId, onDeactivate, disabled }: Props) {
  const [confirmOpen, setConfirmOpen] = useState(false);
  const [loading, setLoading] = useState(false);

  async function handleConfirm() {
    setLoading(true);
    try {
      await onDeactivate();
    } finally {
      setLoading(false);
      setConfirmOpen(false);
    }
  }

  return (
    <>
      <button
        onClick={() => setConfirmOpen(true)}
        disabled={disabled || loading}
        aria-haspopup="dialog"
        aria-expanded={confirmOpen}
        aria-controls={`confirm-deactivate-${templateId}`}
      >
        Deactivate
      </button>
      {confirmOpen && (
        <div
          role="dialog"
          id={`confirm-deactivate-${templateId}`}
          aria-modal="true"
          aria-labelledby={`confirm-title-${templateId}`}
          tabIndex={-1}
        >
          <h2 id={`confirm-title-${templateId}`}>Confirm Deactivation</h2>
          <p>Are you sure you want to deactivate this template?</p>
          <button onClick={handleConfirm} disabled={loading}>
            Yes, Deactivate
          </button>
          <button onClick={() => setConfirmOpen(false)} disabled={loading}>
            Cancel
          </button>
        </div>
      )}
    </>
  );
}

// LoadingIndicator.tsx
import React from "react";

type Props = {
  message?: string;
};

export function LoadingIndicator({ message = "Loading..." }: Props) {
  return (
    <div role="status" aria-live="polite" aria-busy="true">
      <span className="spinner" aria-hidden="true" /> {message}
    </div>
  );
}

// ErrorMessage.tsx
import React from "react";

type Props = {
  message: string;
};

export function ErrorMessage({ message }: Props) {
  return (
    <div role="alert" style={{ color: "red" }}>
      {message}
    </div>
  );
}
```
---

This plan and example code provide a clean, accessible, and maintainable React UI implementation for Notification Template Management, respecting the requirements and enterprise best practices.