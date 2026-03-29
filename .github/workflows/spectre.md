---
on:
  pull_request_target:
    types: [labeled]
labels: [spec-architecture-review-needed]
if: github.event.label.name == 'spec-architecture-review-needed'
description: "Spectre: Review a pull request for API specification design issues"
permissions:
  contents: read
  pull-requests: read
  actions: read
tools:
  github:
    toolsets: [context, repos, pull_requests, actions]
    min-integrity: unapproved
  bash: true
  cache-memory:
  repo-memory:
safe-outputs:
  create-pull-request-review-comment:
    max: 50
    side: "RIGHT"
    target: "${{ github.event.pull_request.number || github.event.issue.number }}"
  submit-pull-request-review:
    max: 1
    footer: "if-body"
    target: "${{ github.event.pull_request.number || github.event.issue.number }}"
  messages:
    footer: "> 👻 *Reviewed by [{workflow_name}]({run_url})*"
    run-started: "👻 [{workflow_name}]({run_url}) is reviewing this PR for API spec design issues…"
    run-success: "👻 [{workflow_name}]({run_url}) completed the spec architecture review. ✅"
    run-failure: "👻 [{workflow_name}]({run_url}) {status}. ❌"
timeout-minutes: 15

---

# Spec Architecture Review

Review pull request #${{ github.event.pull_request.number }} for API
specification design issues.

Follow the guidelines in [spec-architecture-review-guidelines.md](../prompts/spec-architecture-review-guidelines.md).

## Important Constraints

- Only review changes to **API specification files** (`.tsp`, `.json`,
  `tspconfig.yaml`). Ignore example files, documentation, CI config,
  and tooling.
- Only flag issues **introduced or worsened** by this pull request. Do not
  flag pre-existing issues in unchanged files.
- Do **not** re-flag issues already caught by automated CI checks
  (lintdiff, breaking-change detection, ARM auto-signoff). Instead,
  focus on design quality issues that automation misses.
- Do **not** comment on style, formatting, or whitespace.

## Step 0 — Context Gathering

1. **Check CI status** — use the Actions toolset to check whether CI
   checks are passing on this PR. Note any failing checks (lintdiff,
   breaking-change, avocado) as context but proceed with the review.
2. **Determine spec type** — identify whether this PR changes ARM
   resource-manager specs, data-plane specs, or TypeSpec definitions.
   This determines which rule categories apply.
3. **Recall past context** — use cache-memory to check if this PR,
   service, or resource provider has been reviewed before.

## Step 1 — Identify Changed Spec Surface

1. List the files changed in the pull request using the GitHub API.
2. Categorize changes:
   - **TypeSpec files** (`*.tsp`) — check model definitions, decorators,
     operations, versioning
   - **Swagger/OpenAPI JSON** (`*.json` under `specification/`) — check
     paths, schemas, parameters, responses, common-types usage
   - **Config files** (`tspconfig.yaml`) — check emitter configuration,
     API version settings
3. Ignore files under `examples/` directories and non-specification files.
4. If no specification surface was changed, post a single comment saying
   the spec looks good and stop.

## Step 2 — Check Against Guidelines

For TypeSpec changes:
- Verify proper use of ARM decorators and templates
- Check model inheritance and property design
- Validate versioning decorator usage
- Look for common TypeSpec anti-patterns

For Swagger/OpenAPI changes:
- **Determine the baseline API version** — identify the previous
  stable/preview version to compare against for breaking changes.
- Check ARM resource model requirements (CRUD completeness, path
  structure, common-types inheritance)
- Validate property design (enums vs booleans, mutability, secrets)
- Check pagination, LRO design, and error handling patterns
- Verify naming conventions and operation ID patterns

For both:
- Apply the full checklist from the spec architecture review guidelines
- Focus on issues that linters and automated checks typically miss:
  design patterns, resource modeling choices, API ergonomics

## Step 3 — Submit Review

Submit your findings as a **pull request review** with inline code
comments.

For each finding, create a **review comment** on the specific file and
line using `create-pull-request-review-comment`:

> 🔴 **Breaking** — `VirtualMachines_Get` response removed the
> `provisioningState` property that exists in `2024-01-01` stable.
> **Fix:** Keep the property and mark as deprecated if no longer used.

After all inline comments, **submit the review** using
`submit-pull-request-review` with:

- **event**: `COMMENT` (advisory review, not a blocking gate)
- **body**: A one-paragraph summary (count of findings by severity, or
  "No spec design issues found") followed by:

<pre>
&lt;details&gt;
&lt;summary&gt;📊 Structured Report&lt;/summary&gt;

```json
{"agent":"spectre","pr":NUMBER,"summary":"clean|issues_found","findings":[{"file":"...","line":0,"severity":"critical|medium|low","category":"...","description":"..."}]}
```

&lt;/details&gt;
</pre>

If no issues were found, submit a `COMMENT` review with a one-sentence
body confirming the specification looks good.

## Step 4 — Suggest a Service README

After posting the review, check whether the spec directory already
contains a `README.md` that documents the service (not just AutoRest
configuration). If it does not, post an **additional review comment** on
the PR with a suggested `README.md` as a code suggestion.

The README should include:

1. **Service name and one-paragraph description** — what the service
   does, who it is for, and what problem it solves.
2. **Hero scenarios** — comprehensive list of end-to-end tasks a
   customer would perform, derived from the operations and resource
   types defined in the spec. Cover the full breadth of the API:
   CRUD lifecycle, configuration, monitoring, troubleshooting, and
   any domain-specific workflows (e.g., "Create a workspace and
   attach compute", "Submit a training job and monitor progress",
   "Configure autoscale rules and review scaling events").

Format the suggestion as a fenced code block so the PR author can
commit it directly. Keep it concise (under 60 lines). If a README
already exists with substantive service documentation, skip this step.

## Step 5 — Update Memory

After posting, store a brief summary in cache-memory (PR number,
service/RP namespace, spec type, outcome) so future runs can detect
repeat patterns across the same service.
