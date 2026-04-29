## src/examples/jira-ticket.txt

Score: 59

As an admin user, I want to manage notification templates so that I can create, edit, activate, and deactivate templates used for customer notifications.

Acceptance criteria:
- Admin can view a list of templates
- Admin can create a template with name, channel, subject, body, and status
- Admin can edit existing templates
- Admin can deactivate templates
- Required fields must be validated
- Subject is required for email templates
- Only active templates can be used for sending notifications

---

## README.md

Score: 40

# AI SDLC Agent Prototype

A local TypeScript CLI prototype that demonstrates an agentic SDLC workflow.

## Workflow

```text
Jira Ticket
  -> Requirements Agent
  -> Human approval: requirements
  -> UI Agent
  -> API Agent
  -> Human approval: UI/API design
  -> Test Agent
  -> Review Agent
  -> Human approval: final SDLC package
  -> Output artifacts
```

## Setup

```bash
npm install
```

Create `.env`:

```bash
cp .env.example .env
```

Then update `.env`:

```bash
OPENAI_API_KEY=your_api_key_here
OPENAI_MODEL=gpt-4.1-mini
OPENAI_EMBEDDING_MODEL=text-embedding-3-small

# Optional: Jira Cloud input
JIRA_BASE_URL=https://your-domain.atlassian.net
JIRA_EMAIL=your.email@example.com
JIRA_API_TOKEN=your_jira_api_token
JIRA_ACCEPTANCE_CRITERIA_FIELD=customfield_12345

# Optional: GitHub issue input and artifact PR publishing
GITHUB_TOKEN=your_github_token
```

## Run

Run with the included example ticket:

```bash
npm run generate
```

Or pass a custom ticket:

```bash
npm run generate -- ./src/examples/jira-ticket.txt
```

Skip approval prompts for automation:

```bash
npm run generate -- --no-approval
```

Fetch a real Jira Cloud issue:

```bash
npm run generate -- --jira PROJ-123
```

Fetch a Jira issue without approval prompts:

```bash
npm run generate -- --jira PROJ-123 --no-approval
```

Example:

```bash
npm run generate -- --jira KAN-1 --no-approval
```

Fetch a GitHub issue:

```bash
npm run generate -- --github-issue OWNER/REPO#123 --no-approval
```

Use static project context:

```bash
npm run generate -- --context-dir ./context --no-approval
```

Use Jira plus project context:

```bash
npm run generate -- --jira KAN-1 --context-dir ./context --no-approval
```

Retrieve relevant files from a target repository:

```bash
npm run generate -- --jira KAN-1 --context-dir ./context --repo-dir ../inventory-api --no-approval
```

Use local vector RAG over a target repository:

```bash
npm run generate -- --jira KAN-1 --context-dir ./context --repo-dir ../inventory-api --rag vector --no-approval
```

Force rebuilding the local vector index:

```bash
npm run generate -- --jira KAN-1 --context-dir ./context --repo-dir ../inventory-api --rag vector --reindex --no-approval
```

Run local validation commands against the target repo:

```bash
npm run generate -- --repo-dir . --validate --no-approval
```

Choose a validation profile:

```bash
npm run generate -- --repo-dir ../inventory-api --validate --validation-profile dotnet --no-approval
```

Publish generated artifacts to a GitHub branch:

```bash
npm run generate -- --jira KAN-1 --no-approval --github-repo OWNER/REPO --github-base main
```

Generate from a GitHub issue and open a draft PR:

```bash
npm run generate -- --github-issue OWNER/REPO#123 --context-dir ./context --repo-dir . --rag vector --no-approval --github-base main --open-pr
```

Use a stable artifact branch for demos:

```bash
npm run generate -- --github-repo OWNER/REPO --github-branch ai-sdlc/demo-run --no-approval
```

`JIRA_ACCEPTANCE_CRITERIA_FIELD` is optional. Set it only if your Jira project stores acceptance criteria in a custom field.

`GITHUB_TOKEN` is required only when using GitHub flags. Fine-grained tokens need repository contents write access for artifact commits and pull request write access when using `--open-pr`.

## Output

Each run prints a run ID:

```text
Run ID: kan-1-20260428T104757Z
Run output: output/runs/kan-1-20260428T104757Z
```

The latest generated files are written to:

```text
output/
  ticket.md
  context.md
  retrieved-context.md
  requirements.json
  ui-plan.md
  api-plan.md
  tests.md
  review.md
  run-summary.json
```

Each execution is also archived under `output/runs/<run-id>/`:

```text
output/
  runs/
    <run-id>/
      ticket.md
      context.md
      retrieved-context.md
      requirements.json
      ui-plan.md
      api-plan.md
      tests.md
      review.md
      run-summary.json
```

The top-level files are convenient "latest" artifacts. The `output/runs/<run-id>/` folder is the traceable archive for that specific execution.

`context.md` is written only when `--context-dir` is provided.
`retrieved-context.md` is written only when `--repo-dir` is provided.

If requirements JSON does not match the schema, the Requirements Agent retries with validation feedback before failing. If repair fails after 3 total attempts, a failure report is written to:

```text
output/requirements-repair-failure.md
```

## Templates

The planning agents use Markdown templates from `templates/` to keep generated artifacts consistent:

```text
templates/
  react-ui-template.md
  node-api-template.md
  dotnet-api-template.md
  test-plan-template.md
  pr-description-template.md
```

These templates shape the generated plans only. The prototype does not generate code changes or pull requests yet.

## Project Context

Static project context files live under `context/`:

```text
context/
  requirements-guidelines.md
  ui-guidelines.md
  backend-guidelines.md
  testing-guideli

[File truncated before retrieval scoring.]

---

## templates/react-ui-template.md

Score: 34

# React UI Implementation Plan Template

## UI Overview

Describe the user workflow and screen-level structure.

## Components

List components with responsibilities and ownership boundaries.

## Component Props

Define key props, types, and callbacks.

## State Management

Describe local state, server state, loading/error state, and form state.

## Form Validation

Map each validation rule to a user-facing form behavior.

## User Interactions

Describe create, read, update, activate/deactivate, delete, and confirmation flows as applicable.

## Loading, Empty, and Error States

Define states for initial loading, empty lists, validation errors, API failures, and retry behavior.

## Accessibility

Include keyboard navigation, labels, focus handling, error announcements, and semantic structure.

## Example React TypeScript Code

Provide practical sample code with typed props and clean component boundaries.

---

## src/integrations/jira-client.ts

Score: 32

type JiraDocNode = {
    type?: string;
    text?: string;
    content?: JiraDocNode[];
};

type JiraIssueResponse = {
    key: string;
    fields: {
        summary?: string;
        description?: JiraDocNode | string | null;
        issuetype?: {
            name?: string;
        };
        status?: {
            name?: string;
        };
        priority?: {
            name?: string;
        } | null;
        labels?: string[];
        components?: Array<{
            name?: string;
        }>;
        [key: string]: unknown;
    };
};

type JiraConfig = {
    baseUrl: string;
    email: string;
    apiToken: string;
    acceptanceCriteriaField?: string;
};

function getRequiredEnv(name: string): string {
    const value = process.env[name];

    if (!value) {
        throw new Error(`Missing ${name} environment variable for Jira integration.`);
    }

    return value;
}

function getJiraConfig(): JiraConfig {
    const acceptanceCriteriaField = process.env.JIRA_ACCEPTANCE_CRITERIA_FIELD;

    return {
        baseUrl: getRequiredEnv("JIRA_BASE_URL").replace(/\/$/, ""),
        email: getRequiredEnv("JIRA_EMAIL"),
        apiToken: getRequiredEnv("JIRA_API_TOKEN"),
        ...(acceptanceCriteriaField ? { acceptanceCriteriaField } : {})
    };
}

function toBasicAuth(email: string, apiToken: string): string {
    return Buffer.from(`${email}:${apiToken}`).toString("base64");
}

function jiraDocToText(value: JiraDocNode | string | null | undefined): string {
    if (!value) {
        return "";
    }

    if (typeof value === "string") {
        return value;
    }

    const parts: string[] = [];

    function visit(node: JiraDocNode): void {
        if (node.text) {
            parts.push(node.text);
        }

        if (node.type === "hardBreak") {
            parts.push("\n");
        }

        for (const child of node.content ?? []) {
            visit(child);
        }

        if (node.type === "paragraph" || node.type === "heading" || node.type === "listItem") {
            parts.push("\n");
        }
    }

    visit(value);

    return parts
        .join("")
        .replace(/\n{3,}/g, "\n\n")
        .trim();
}

function unknownFieldToText(value: unknown): string {
    if (!value) {
        return "";
    }

    if (typeof value === "string") {
        return value;
    }

    if (typeof value === "number" || typeof value === "boolean") {
        return String(value);
    }

    if (typeof value === "object" && value !== null) {
        return jiraDocToText(value as JiraDocNode) || JSON.stringify(value, null, 2);
    }

    return "";
}

function formatList(label: string, values: string[]): string {
    if (values.length === 0) {
        return "";
    }

    return `${label}: ${values.join(", ")}`;
}

function formatIssueAsTicket(issue: JiraIssueResponse, acceptanceCriteriaField?: string): string {
    const fields = issue.fields;
    const description = jiraDocToText(fields.description);
    const acceptanceCriteria = acceptanceCriteriaField ? unknownFieldToText(fields[acceptanceCriteriaField]) : "";
    const labels = formatList("Labels", fields.labels ?? []);
    const components = formatList(
        "Components",
        (fields.components ?? []).map((component) => component.name).filter((name): name is string => Boolean(name))
    );

    return `
Jira issue: ${issue.key}
Issue type: ${fields.issuetype?.name ?? "Unknown"}
Status: ${fields.status?.name ?? "Unknown"}
Priority: ${fields.priority?.name ?? "Unknown"}
${labels}
${components}

Summary:
${fields.summary ?? ""}

Description:
${description || "(No description provided)"}
${acceptanceCriteria ? `\nAcceptance criteria:\n${acceptanceCriteria}` : ""}
    `.trim();
}

export async function fetchJiraTicket(issueKey: string): Promise<string> {
    const config = getJiraConfig();
    const fields = [
        "summary",
        "description",
        "issuetype",
        "status",
        "priority",
        "labels",
        "components",
        config.acceptanceCriteriaField
    ].filter((field): field is string => Boolean(field));
    const url = new URL(`${config.baseUrl}/rest/api/3/issue/${encodeURIComponent(issueKey)}`);

    url.searchParams.set("fields", fields.join(","));

    const response = await fetch(url, {
        headers: {
            Accept: "application/json",
            Authorization: `Basic ${toBasicAuth(config.email, config.apiToken)}`
        }
    });

    if (!response.ok) {
        const body = await response.text();
        throw new Error(`Failed to fetch Jira issue ${issueKey}: ${response.status} ${response.statusText}${body ? ` - ${body}` : ""}`);
    }

    const issue = await response.json() as JiraIssueResponse;
    return formatIssueAsTicket(issue, config.acceptanceCriteriaField);
}

---

## PATH.md

Score: 31

Yes — for production, your prototype should evolve from a **local sequential CLI** into a **controlled SDLC automation platform**.

The key idea:

> Do **not** jump directly to “autonomous agent writes code and commits.”
> First build a reliable pipeline: structured requirements → context retrieval → implementation plan → code changes → tests → review → human approval.

That is the production mindset.

---

# 1. Production evolution roadmap

## Current implementation status

Completed:

- [x] Stage 0 local TypeScript CLI prototype
- [x] Sequential agent workflow: Requirements → UI → API → Test → Review
- [x] Local ticket input from `src/examples/jira-ticket.txt`
- [x] Jira Cloud ticket input via `--jira ISSUE-KEY`
- [x] GitHub issue input via `--github-issue OWNER/REPO#123`
- [x] GitHub artifact branch and draft PR publishing
- [x] Output artifacts written to `output/`
- [x] Per-run output archive with run IDs
- [x] Markdown project templates for consistent generated artifacts
- [x] Static project context loader via `--context-dir`
- [x] Local repo retrieval via `--repo-dir`
- [x] Local vector RAG via `--rag vector`
- [x] LangGraph orchestration parity
- [x] LangGraph durable checkpointing/resume
- [x] Zod schema validation for Requirements Agent output
- [x] Requirements JSON repair loop with validation feedback
- [x] Repository-level AI instructions with `.github/copilot-instructions.md`
- [x] Agent role documentation with `AGENTS.md`
- [x] Local human approval gates after requirements, UI/API design, and final SDLC package
- [x] `--no-approval` flag for automated local runs
- [x] Basic OpenAI quota/billing error message

Partially completed:

- [ ] Stage 1 reliability hardening
  - [x] Schema validation
  - [x] Retry logic for requirements repair
  - [x] JSON repair for requirements
  - [x] Failure report for requirements repair
  - [x] Basic run summary artifact
  - [ ] Detailed agent logs
  - [ ] Token usage tracking
  - [ ] Config per agent
  - [ ] Prompt versioning
  - [ ] Output versioning
- [ ] Stage 2 production-grade approval gates
  - [x] Local CLI approval gates
  - [x] Durable approval records
  - [x] Approval comments
  - [ ] Approver identity
  - [x] Resume after approval
  - [ ] PR approval gate

Remaining larger roadmap items:

- [x] Real ticket system integration: Jira Cloud issue fetch
- [ ] Atlassian Rovo MCP integration for Jira/Confluence context and workflow actions
- [x] Local keyword retrieval over existing repos/docs
- [x] Local JSON vector/RAG search over existing repos/docs
- [ ] Production vector database/search integration
- [x] LangGraph orchestration parity
- [x] LangGraph durable checkpointing/resume
- [ ] Codebase-aware patch generation
- [x] GitHub PR integration for generated SDLC artifacts
- [ ] Azure DevOps PR integration
- [x] Local CI validation integration
- [ ] Remote CI validation integration
- [ ] Full observability/evaluation
- [ ] Security/governance controls
- [ ] Product dashboard/backend/worker architecture

## Stage 0 — Local CLI prototype — DONE

This is what we started:

```text
Ticket text
  → Requirements Agent
  → UI Agent
  → API Agent
  → Test Agent
  → Review Agent
  → output files
```

Goal:

> Prove the workflow.

Tech:

```text
TypeScript
OpenAI API / Azure OpenAI
Zod schemas
Markdown outputs
Local file system
```

This is enough for demo, but not production.

---

# 2. Stage 1 — Make the local prototype reliable — IN PROGRESS

Before adding LangGraph or vector DBs, improve reliability.

Add:

- [x] Schema validation
- [x] Retry logic for requirements repair
- [x] JSON repair for requirements
- [x] Failure report for requirements repair
- [ ] Agent logs
- [ ] Token usage tracking
- [ ] Config per agent
- [ ] Prompt versioning
- [ ] Output versioning

Current problem:

```text
LLM returns invalid JSON → app fails
```

Production behavior should be:

```text
LLM returns invalid JSON
  → validation fails
  → send validation errors back to repair agent
  → validate again
  → fail only after N attempts
```

This is a strong interview point:

> I would not trust raw LLM output. Every structured artifact should be validated with schemas, and invalid output should go through a repair loop.

Suggested tools:

```text
Zod
Pino / Winston for logs
OpenTelemetry later
PostgreSQL for saved runs later
```

---

# 3. Stage 2 — Add human approval gates — PARTIAL

This is critical.

Agentic SDLC should not mean “AI blindly modifies production code.”

Add approval gates after:

- [x] Requirements extraction
- [x] Architecture/API plan
- [ ] Code generation
- [ ] Test generation approval as a separate gate
- [ ] PR creation

Flow:

```text
Ticket
  → Requirements Agent
  → Human approves requirements
  → UI/API planning agents
  → Human approves design
  → Code generation
  → Tests/lint/typecheck
  → Review Agent
  → Human approves PR
```

Why this matters:

> In enterprise SDLC, human review and auditability are more important than full autonomy.

LangGraph is usef

[File truncated before retrieval scoring.]

---

## templates/node-api-template.md

Score: 26

# Node API Implementation Plan Template

## API Overview

Summarize the REST resource, lifecycle, and business boundaries.

## REST Endpoints

Document method, path, purpose, request body, response body, and status codes.

## DTOs

Define request and response DTOs with TypeScript types.

## Validation Rules

Map each validation rule to request validation and service-layer enforcement.

## Database Model

Describe the entity fields, indexes, unique constraints, and timestamps.

## Service Responsibilities

Separate controller, service, repository, and mapper responsibilities.

## Error Handling

Describe validation, not found, conflict, authorization, and unexpected error responses.

## Security Considerations

Call out authentication, authorization, input validation, audit logging, and sensitive data concerns.

## Example Backend Stub Code

Provide TypeScript/Node.js examples that show controller, DTO, service, and repository boundaries.

---

## templates/test-plan-template.md

Score: 24

# Test Plan Template

## Unit Tests

Cover validation logic, business rules, mappers, and service behavior.

## API Tests

Cover REST endpoints, success responses, validation errors, not found, conflict, and edge cases.

## E2E Tests

Cover the primary user workflows from UI interaction through API response.

## Negative Tests

Include invalid input, missing required fields, invalid state transitions, duplicates, and permission failures.

## Edge Cases

Cover boundaries, empty data, large payloads, status changes, and dependent business rules.

## Regression Risks

List behavior likely to break when requirements change.

## Suggested Test Data

Provide concrete valid and invalid examples.

---

## src/agents/review-agent.ts

Score: 22

import { generateText } from "../core/llm.js";
import { readTemplate } from "../core/template-utils.js";
import type { FullSdlcContext } from "../core/types.js";

export async function runReviewAgent(context: FullSdlcContext): Promise<string> {
    const prTemplate = await readTemplate("pr-description-template.md");

    return generateText({
        system: `
You are a senior engineering reviewer and solution architect.

Review the generated requirements, UI plan, API plan, and tests.

Return Markdown with:
- Executive summary
- Missing requirements
- Architectural risks
- Security risks
- Test gaps
- Maintainability concerns
- Recommended improvements
- Human approval checklist
- Final readiness score from 1 to 10

Rules:
- Be direct and practical.
- Do not simply praise the output.
- Identify real risks and gaps.
- Think like a lead engineer reviewing AI-generated work before PR creation.
- Include a "PR readiness guidance" section that explains how this output should map to the PR template below.

PR description template:

${prTemplate}
    `.trim(),
        user: `
Original ticket:

${context.ticket}

Requirements:

${JSON.stringify(context.requirements, null, 2)}

UI plan:

${context.uiPlan}

API plan:

${context.apiPlan}

Test plan:

${context.testPlan}

${context.validationSummary ? `Validation results:\n\n${context.validationSummary}` : ""}

${context.agentContext ? `Project context:\n\n${context.agentContext.content}` : ""}
    `.trim()
    });
}