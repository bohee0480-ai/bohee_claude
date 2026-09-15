# CodexGPT · Claude · VS Code · Higgsfield Video Automation System Design

## 1. Document Purpose

This document defines implementation standards for applying the core `CLAUDE.md` principles—**Command → Agent → Skill**, feature-specific subagents, human approval gates, and progressive disclosure—to video production automation.

The goal is a consistent pipeline that processes a natural-language brief through planning, prompt authoring, Higgsfield generation, quality review, and result organization.

> Important: VS Code is a development and operations interface, not a video generation engine. The Higgsfield API performs generation, while Claude and CodexGPT divide planning, coding, review, and orchestration responsibilities.

## 2. System Goals

### MVP Goals

1. Convert a natural-language video brief into a structured job specification.
2. Prefer free and open APIs whenever they satisfy the required quality, license, privacy, and reliability constraints.
3. Before any paid API call, explain the provider, pricing basis, estimated number of requests, estimated cost or maximum cost, data sent externally, and available free alternatives; execute it only after the user gives explicit approval.
4. Submit asynchronous generation jobs and track them to completion.
5. Download generated videos into the local project.
6. Separate technical validation from AI-based content review.
7. Retry only failed shots and retain complete execution records.

### Free-First API Policy

- Use a reviewed allowlist of free/open APIs and open-source local tools as the default provider pool.
- “Free” does not automatically mean safe or unrestricted. Verify the license, commercial-use terms, rate limits, retention policy, privacy policy, and output rights before integration.
- Prefer local processing when a suitable open-source model or tool is available and practical.
- Never silently switch from a free provider to a paid provider when quotas, rate limits, or quality checks fail.
- If no acceptable free option exists, pause and present the paid option and its free alternatives for explicit user approval.

### Excluded from the MVP

- Fully unattended publishing
- Unapproved bulk generation
- Automatic payment or credit purchases
- Complex routing across multiple video models
- Automated After Effects post-production
- Automatic uploads to YouTube, Instagram, or TikTok

Add these in separate phases after the core pipeline is stable.

## 3. Tool Responsibilities

### Claude

- Interview users and organize requirements
- Write creative briefs
- Design story structure and shot lists
- Draft prompts for Higgsfield
- Coordinate the full Command → Agent → Skill flow
- Manage human approval points
- Review semantic and brand quality of outputs

### CodexGPT

In this document, CodexGPT means an OpenAI-family coding agent or the Codex CLI.

- Implement TypeScript/Python
- Build API clients, queues, and state trackers
- Write unit and integration tests
- Reproduce and fix errors
- Independently review Claude's implementation plans
- Write technical validation code using FFmpeg/ffprobe

Partition work so Claude and CodexGPT do not edit the same file concurrently. Use Git worktrees or separate branches when parallel development is required.

### VS Code

- Edit source code and Markdown specifications
- Run the orchestrator in the integrated terminal
- Manage environment variables and execution logs
- Review job-state JSON, prompts, and generated videos
- Serve as an operations console using Claude/Codex extensions or CLIs

### Higgsfield

- Process image and video generation requests
- Issue asynchronous job IDs
- Provide job-status queries or webhook notifications
- Provide completed media URLs

Treat Higgsfield as an optional provider. Use it only when its current plan satisfies the free-first policy or after the user explicitly approves any paid use.

Call all credentialed APIs from a trusted server-side process. Never place API keys or secrets in browser code, frontend bundles, generated pages, screenshots, prompts, terminal output, or client-visible error messages.

## 4. Overall Architecture

```text
[User Brief]
       |
       v
[/video-create Command]
       |
       v
[Claude Orchestrator]
       |
       +--> Brief Agent --------> brief.json
       +--> Director Agent -----> storyboard.json
       +--> Prompt Agent -------> generation-plan.json
       |
       v
[Human Approval: Cost, Prompts, Shot Count]
       |
       v
[Higgsfield Skill / API Adapter]
       |
       +--> submit
       +--> poll or webhook
       +--> download
       |
       v
[Technical QA + Creative QA]
       |
       +--> pass ----------> outputs/final/
       +--> fail ----------> regenerate only affected shot
```

## 5. Command → Agent → Skill Design

### Commands

#### `/video-create [brief-file]`

Entry point for the complete production workflow.

1. Validate the input brief
2. Ask for missing information
3. Run planning agents
4. Write the generation plan
5. Request human approval
6. Submit Higgsfield jobs after approval
7. Review completed results
8. Write an execution report

#### `/video-status [project-id]`

- Display queued/running/completed/failed states for all shots
- Display estimated cost and actual request count
- Display download status and QA results

#### `/video-retry [project-id] [shot-id]`

- Regenerate only failed or rejected shots
- Do not overwrite existing successful results
- Record the retry reason and revised prompt

#### `/video-review [project-id]`

- Review existing results without calling the generation API
- Check resolution, duration, codec, and file integrity
- Evaluate prompt compliance and scene consistency

### Feature-specific Agents

#### `brief-analyst`

Input:

- Intended use
- Platform
- Target audience
- Core message
- Video duration
- Aspect ratio
- Brand constraints
- Reference images

Output: `brief.json`

Ask the user for missing required information rather than inferring it.

#### `creative-director`

- Structure the hook, development, and ending
- Define each shot's purpose and transition
- Design camera movement within the requested scope
- Define character, product, and color continuity rules

Output: `storyboard.json`

#### `prompt-engineer`

- Convert shot descriptions into Higgsfield input format
- Separate subject, action, environment, camera, lighting, and style
- Apply prohibited-element and continuity tokens
- Validate model-specific options

Output: `generation-plan.json`

#### `generation-operator`

- Send only approved specifications to the API
- Store request IDs
- Update state through polling or webhooks
- Download result files immediately
- Apply the retry policy

#### `qa-reviewer`

- Separate technical review from creative review
- Structure failure reasons
- Identify only shots requiring regeneration
- Preserve original generations

## 6. Skills

### `higgsfield-submit`

- Build authentication headers
- Send requests to the selected model endpoint
- Store `request_id`, `status_url`, and `cancel_url`
- Remove secrets from request bodies before logging

### `higgsfield-monitor`

- Handle queued, running, completed, failed, and nsfw states
- Apply exponential backoff and a maximum wait time
- Prevent duplicate polling
- Verify signatures when using webhooks

### `media-download`

- Download result URLs before they expire
- Download to a temporary file, then rename atomically on success
- Record SHA-256, size, and MIME type

### `technical-video-qa`

Use `ffprobe` to verify:

- File readability
- Presence of video/audio streams
- Actual duration
- Resolution and aspect ratio
- Frame rate
- Codec
- Abnormally small files

### `creative-video-qa`

- Brief-to-scene alignment
- Product, person, and text distortion
- Style continuity across shots
- Camera-movement compliance
- Exposure of prohibited brand elements

## 7. Data Contracts

### Project State

```json
{
  "projectId": "campaign-20260914-001",
  "status": "awaiting_approval",
  "createdAt": "2026-09-14T16:00:00+09:00",
  "briefPath": "projects/campaign-20260914-001/brief.json",
  "shotCount": 3,
  "approvedBy": null,
  "approvedAt": null
}
```

### Generation Plan

```json
{
  "projectId": "campaign-20260914-001",
  "provider": "higgsfield",
  "modelEndpoint": "${HIGGSFIELD_MODEL_ENDPOINT}",
  "aspectRatio": "9:16",
  "shots": [
    {
      "shotId": "shot-001",
      "purpose": "Capture product attention within the first 2 seconds",
      "prompt": "Actual generation prompt written before approval",
      "durationSeconds": 6,
      "inputImages": [],
      "generateAudio": false,
      "status": "draft",
      "attempt": 0
    }
  ]
}
```

Manage model endpoints and supported options through a model registry or configuration rather than hard-coding them. Because request fields may differ by Higgsfield model, validate the schema before API calls.

### Job State

```text
draft
  -> awaiting_approval
  -> approved
  -> queued
  -> running
  -> downloading
  -> qa_pending
  -> completed

Error branches:
queued/running -> failed | nsfw | timed_out | cancelled
qa_pending     -> rejected -> approved(regeneration)
```

## 8. Recommended Project Structure

```text
video-automation/
├─ CLAUDE.md
├─ README.md
├─ .env.example
├─ .gitignore
├─ .claude/
│  ├─ commands/
│  │  ├─ video-create.md
│  │  ├─ video-status.md
│  │  ├─ video-retry.md
│  │  └─ video-review.md
│  ├─ agents/
│  │  ├─ brief-analyst.md
│  │  ├─ creative-director.md
│  │  ├─ prompt-engineer.md
│  │  ├─ generation-operator.md
│  │  └─ qa-reviewer.md
│  ├─ skills/
│  │  ├─ higgsfield-submit/SKILL.md
│  │  ├─ higgsfield-monitor/SKILL.md
│  │  ├─ media-download/SKILL.md
│  │  ├─ technical-video-qa/SKILL.md
│  │  └─ creative-video-qa/SKILL.md
│  └─ rules/
│     ├─ video-generation.md
│     └─ secrets-and-cost.md
├─ src/
│  ├─ cli/
│  ├─ orchestration/
│  ├─ providers/higgsfield/
│  ├─ validation/
│  ├─ storage/
│  └─ types/
├─ tests/
│  ├─ unit/
│  ├─ integration/
│  └─ fixtures/
├─ projects/
│  └─ <project-id>/
│     ├─ brief.json
│     ├─ storyboard.json
│     ├─ generation-plan.json
│     ├─ jobs.json
│     ├─ qa-report.md
│     ├─ inputs/
│     ├─ outputs/raw/
│     └─ outputs/final/
└─ logs/
```

## 9. Higgsfield API Integration Principles

According to the official documentation, Higgsfield provides an asynchronous generation API.

1. Check the provider registry for a suitable free/open provider before selecting Higgsfield.
2. If the selected Higgsfield request may incur a charge, complete Gate 2 and obtain explicit approval before submission.
3. Authenticate on the server with `Authorization: Key KEY_ID:KEY_SECRET`.
4. Submit generation requests to the model endpoint.
5. Persist the returned `request_id` and status URL.
6. Confirm completion through `/requests/{request_id}/status` polling or a webhook.
7. Completed URLs are not guaranteed long-term retention, so copy results immediately to local or object storage.

Required environment variables:

```dotenv
HIGGSFIELD_API_KEY_ID=
HIGGSFIELD_API_KEY_SECRET=
HIGGSFIELD_MODEL_ENDPOINT=
HIGGSFIELD_WEBHOOK_SECRET=
OUTPUT_ROOT=./projects
MAX_CONCURRENT_JOBS=2
MAX_RETRIES=2
POLL_INTERVAL_MS=5000
```

Do not commit `.env` to Git. Record only key names with empty values in `.env.example`.

### API Credential Security

- Read secrets only from server-side environment variables or an operating-system/cloud secret manager.
- Keep `.env`, credential files, and local secret stores outside version control and deny them to static hosting.
- Send API requests through a backend adapter; the browser receives only sanitized job IDs, statuses, and result references.
- Redact authorization headers, query-string tokens, secrets, and signed URLs from logs and error reports.
- Never echo secret values in terminals, prompts, generated Markdown/HTML, screenshots, telemetry, or analytics.
- Restrict each key to the minimum required scope, rotate it if exposure is suspected, and maintain separate development and production credentials.
- Add automated secret scanning and tests that verify responses and logs cannot expose credential values.

## 10. Human Approval Gates

Execute the following operations only after user confirmation.

### Gate 1 — Brief Approval

- Purpose and platform
- Video duration and aspect ratio
- Brand and prohibited elements

### Gate 2 — Paid Generation Approval

- Confirmation that no acceptable free/open option meets the requirement
- Provider and model
- Final prompts
- Shot count
- Pricing basis, estimated request count, estimated total cost, and hard cost ceiling
- Data and files sent to the provider
- Free alternatives considered and why they were not selected
- Permission to use reference images
- Explicit user approval recorded with timestamp; silence or a default option is not approval

### Gate 3 — Regeneration Approval

- Failure reason
- Prompt changes
- Additional cost

### Gate 4 — Publishing Approval

- Final selections
- Review of subtitles, music rights, and likeness rights
- Publishing platform and visibility

## 11. Error and Retry Policy

- HTTP 429/5xx: Retry a limited number of times with exponential backoff
- Authentication error: Stop immediately and verify key configuration
- Input schema error: Revise the plan without automatic resubmission
- NSFW rejection: Report it to the user without automatically generating an evasion prompt
- timeout: Query server state again, then check for duplicate submission
- Download error: Retry only the download using the same result URL
- QA failure: Regenerate only the affected shot, not the entire project

Assign an internal idempotency key to every API submission and prevent duplicate submission of the same approved job after restart.

## 12. Validation Strategy

### Unit Tests

- Brief schema validation
- State-transition rules
- Authentication header generation
- Secret masking
- Classification of retryable and non-retryable errors
- Output filename collision prevention

### Integration Tests

- Submit to a mock Higgsfield server
- queued → running → completed flow
- failed, nsfw, and timeout flows
- Job recovery after process restart
- Duplicate webhook prevention

### Live API Smoke Test

- Generate one shot with the shortest, least expensive settings
- Obtain user cost approval before generation
- Download the result and validate it with ffprobe
- Verify that API responses do not leak sensitive data into logs

## 13. Implementation Phases

### Phase 0 — Decisions

- Implementation language: TypeScript recommended
- Operating model: CLI-first
- State storage: JSON files for MVP; SQLite/PostgreSQL for the multi-user phase
- Start with polling and add webhooks in the operations phase

### Phase 1 — Minimal Vertical Slice

Accept one brief, generate one shot, and download it.

Completion criteria:

- No secrets in the repository or logs
- Request ID recorded
- State polling can resume after interruption
- Result file passes technical validation

### Phase 2 — Multi-shot

- Shot list
- Concurrency limits
- Partial failures and per-shot retries
- Project status view

### Phase 3 — AI Review

- Claude content QA
- CodexGPT technical QA review
- Revised prompt suggestions based on failure reasons
- Selective regeneration after human approval

### Phase 4 — Post-production

- FFmpeg integration
- Shot concatenation
- Subtitles, loudness normalization, and thumbnails
- Separate After Effects adapter when needed

### Phase 5 — Operations

- Webhooks
- Database and object storage
- Budget and usage dashboard
- User permissions
- Publishing-platform integrations

## 14. Claude and CodexGPT Collaboration Rules

1. Claude writes requirements and the approved implementation plan.
2. CodexGPT completes narrow implementation units with code and tests.
3. Claude reviews results from the brief and workflow perspectives.
4. CodexGPT cross-reviews tests, types, and error handling.
5. A different agent independently validates code written by one agent.
6. Pass only conclusions and supporting files instead of loading all research output into the main context.
7. Scope each feature task small enough to complete within half of one session's context.

## 15. Completion Criteria

The MVP is complete when all following conditions are met.

- A natural-language brief is converted into validatable JSON.
- Free/open providers are evaluated and preferred before any paid provider.
- No paid request is executed without a detailed cost disclosure and explicit user approval.
- The complete lifecycle of every Higgsfield request is stored.
- In-progress jobs can resume after process restart.
- Successful results are downloaded locally and checksums are recorded.
- Technical QA results are generated automatically.
- Only failed shots can be retried.
- API keys and secrets do not appear in the browser, frontend bundles, prompts, terminal output, screenshots, code, Git, logs, or result reports.
- Mock-based automated tests and a live one-shot smoke test pass.

## 16. Next Steps

1. Finalize the new project root and implementation language.
2. Write a video-automation-specific `CLAUDE.md` in 200 lines or fewer.
3. Create `brief.schema.json` and `generation-plan.schema.json`.
4. Choose either the official Higgsfield SDK or REST.
5. Implement the one-shot vertical slice for `/video-create` first.
6. Complete mock integration tests before any live API call.

## References

- Original design principles: `CLAUDE.md` in the current directory
- Official Higgsfield documentation: <https://docs.higgsfield.ai/docs>
- Higgsfield Node.js/TypeScript SDK: <https://github.com/higgsfield-ai/higgsfield-js>

Higgsfield model endpoints and input fields may vary by model and API version. Recheck the current official schema at implementation time, and never infer or send undocumented fields.
