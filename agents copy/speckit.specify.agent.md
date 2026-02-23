---
description: Create or update the feature specification from a natural language feature description.
handoffs: 
  - label: Build Technical Plan
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
  - label: Clarify Spec Requirements
    agent: speckit.clarify
    prompt: Clarify specification requirements
    send: true
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Outline

The text the user typed after `/speckit.specify` in the triggering message **is** the feature description. Assume you always have it available in this conversation even if `$ARGUMENTS` appears literally below. Do not ask the user to repeat it unless they provided an empty command.

**Step 0A — Detect Input Mode:**

Inspect `$ARGUMENTS`. If it contains a token matching the pattern `[A-Z]+-[0-9]+` (e.g., `FOOD-123`, `ORD-42`):
- Extract the Jira key
- Enter **Jira Mode**: follow Steps J0–J5 defined in the attached `speckit.specify` prompt file alongside the normal steps below
- The Jira story `summary` field becomes the feature description; `description` field becomes additional context
- Branch naming, impact analysis posting, and spec posting all follow the Jira-specific rules in that prompt
- Impact analysis skip logic (Step J3) overrides the default impact analysis step when a prior comment exists

If no Jira key is detected → proceed in **free-text mode** (normal flow, Jira steps are ignored).

---

**Step 0B — Check for an existing in-progress feature:**

Inspect the current git branch name and check whether a `impact-analysis.md` file already exists inside any matching `specs/` directory.

```
git branch --show-current
```

- If the current branch matches **either** pattern:
  - Free-text: `[0-9]+-<short-name>` (e.g., `5-user-auth`)
  - Jira: `[A-Z]+-[0-9]+-<short-name>` (e.g., `FOOD-123-user-auth`)

  **AND** `specs/<branch-name>/impact-analysis.md` exists:
  - Read the `impact-analysis.md` file
  - Check its `**Status**` field:
    - `Reviewed — On Hold` → **RESUME MODE**: inform the user that an in-progress feature was found, display the saved impact analysis, and ask: `"Continue with the specification for [feature name]?"`. On confirmation skip directly to step 4 (Load template & write spec). Update `Status` to `Reviewed — Proceed`.
    - `Reviewed — Proceed` → **RESUME MODE**: spec was already approved. Check if `spec.md` is empty or incomplete. If so, skip to step 4. If spec already has content, report it and ask if the user wants to update it.
    - Any other status → treat as a fresh run, proceed normally from step 1.
- If no matching branch or file is found → proceed normally from step 1 below.

---

Given that feature description, do this:

1. **Perform Impact Analysis** before writing the specification:

   Scan the existing project to understand what the proposed feature touches, conflicts with, or depends on.

   a. **Scan existing specs**: List all directories under `specs/` (if the folder exists) and read any `spec.md` files found. Identify specs whose title or scope overlaps with the new feature description.

   b. **Read `llms.txt`** from the project root — this is a curated, LLM-optimised index of the entire project. Read it first as the primary source of truth:

      - It contains a plain-English summary of the service, all features, business rules, endpoints, entities, and links to every doc
      - Use it to immediately understand what already exists and what the feature description overlaps with
      - If deeper detail is needed on a specific area (e.g., exact business rules, full schema), follow the relevant link from `llms.txt` to the specific doc file in `docs/`

      From `llms.txt` (and any targeted follow-up doc reads), identify:
      - Components and services that will need to change
      - Shared data models or entities that will be affected
      - Existing endpoints or workflows that overlap with the requested feature

   c. **Identify risks**: Based on the above, surface:
      - **Overlapping features**: Existing specs or code that already partially covers this request
      - **Conflicting behaviour**: Places where the new feature would contradict or break existing rules
      - **Upstream dependencies**: Features or systems the new feature relies on that don't exist yet
      - **Downstream impact**: Features or consumers that will be affected by this change

   d. **Proceed with branch and project structure creation** (steps 2 and 3 below) — these always run immediately after the analysis is complete, regardless of whether the user continues now or later.

   e. **Write the impact analysis as a file** at `FEATURE_DIR/impact-analysis.md` using this structure:

      ```markdown
      # Impact Analysis: [Feature Short Description]

      **Feature**: [link to spec.md]
      **Date**: [DATE]
      **Status**: [Pending Review | Reviewed — Proceed | Reviewed — On Hold]

      ## Overlapping / Related Specs
      | Spec | Overlap |
      |------|---------|
      | [spec name or "None found"] | [brief description of overlap or "—"] |

      ## Affected Codebase Areas
      | Area | Nature of Impact |
      |------|-----------------|
      | [file / module / component] | [what will need to change] |

      ## Risks & Dependencies
      | Type | Description |
      |------|-------------|
      | Dependency | [upstream feature or system needed] |
      | Conflict | [existing behaviour that conflicts] |
      | Downstream | [what this change may break or affect] |

      ## Recommendation
      [One of: "Safe to proceed", "Proceed with caution — [reason]", or "STOP — [blocking reason requiring resolution first]"]

      ## Notes
      [Any additional context, user-provided scope adjustments, or follow-up questions]
      ```

   f. **Present the impact summary to the user** (display the same content from the file) and ask:

      > **The impact analysis has been saved to `FEATURE_DIR/impact-analysis.md`.**
      > Would you like to continue with the specification now, or come back to it later?
      > - Reply `continue` to proceed with the full spec now
      > - Reply `later` to stop here — the branch and impact file are saved, pick this up anytime

   g. **Handle user response**:
      - `continue` (or equivalent): update `Status` in the impact file to `Reviewed — Proceed`, then continue to step 4
      - `later` (or equivalent): update `Status` to `Reviewed — On Hold`, stop here — branch exists, impact file is saved
      - Additional context provided: incorporate it into the impact file's **Notes** section, update the analysis tables if scope changes significantly, re-present the summary, then ask continue/later again

2. **Generate a concise short name** (2-4 words) for the branch:
   - Analyze the feature description and extract the most meaningful keywords
   - Create a 2-4 word short name that captures the essence of the feature
   - Use action-noun format when possible (e.g., "add-user-auth", "fix-payment-bug")
   - Preserve technical terms and acronyms (OAuth2, API, JWT, etc.)
   - Keep it concise but descriptive enough to understand the feature at a glance
   - Examples:
     - "I want to add user authentication" → "user-auth"
     - "Implement OAuth2 integration for the API" → "oauth2-api-integration"
     - "Create a dashboard for analytics" → "analytics-dashboard"
     - "Fix payment processing timeout bug" → "fix-payment-timeout"

3. **Create branch and project structure**:

   > **Jira Mode**: Skip sub-steps a–c entirely. The branch name is already determined by Step J2 as `{JIRA_KEY}-{short-name}` (e.g., `FOOD-123-user-auth`). Pass it directly to the script using `--short-name "{JIRA_KEY}-{short-name}" --number 1` — the Jira key acts as the unique identifier, sequential numbering is not used.

   > **Free-text Mode**: Follow sub-steps a–c below.

   a. First, fetch all remote branches to ensure we have the latest information:

      ```bash
      git fetch --all --prune
      ```

   b. Find the highest feature number across all sources for the short-name:
      - Remote branches: `git ls-remote --heads origin | grep -E 'refs/heads/[0-9]+-<short-name>$'`
      - Local branches: `git branch | grep -E '^[* ]*[0-9]+-<short-name>$'`
      - Specs directories: Check for directories matching `specs/[0-9]+-<short-name>`

   c. Determine the next available number:
      - Extract all numbers from all three sources
      - Find the highest number N
      - Use N+1 for the new branch number

   d. Run the script `.specify/scripts/powershell/create-new-feature.ps1` with the calculated values:
      - Pass `--number N+1` and `--short-name "your-short-name"` along with the feature description
      - Bash example: `.specify/scripts/powershell/create-new-feature.ps1 -Json "$ARGUMENTS" --json --number 5 --short-name "user-auth" "Add user authentication"`
      - PowerShell example: `.specify/scripts/powershell/create-new-feature.ps1 -Json "$ARGUMENTS" -Json -Number 5 -ShortName "user-auth" "Add user authentication"`

   **IMPORTANT**:
   - Check all three sources (remote branches, local branches, specs directories) to find the highest number
   - Only match branches/directories with the exact short-name pattern
   - If no existing branches/directories found with this short-name, start with number 1
   - You must only ever run this script once per feature
   - The JSON is provided in the terminal as output - always refer to it to get the actual content you're looking for
   - The JSON output will contain BRANCH_NAME and SPEC_FILE paths
   - For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot")

4. Load `.specify/templates/spec-template.md` to understand required sections.

5. Follow this execution flow:

    1. Parse user description from Input
       If empty: ERROR "No feature description provided"
    2. Extract key concepts from description
       Identify: actors, actions, data, constraints
    3. For unclear aspects:
       - Make informed guesses based on context and industry standards
       - Only mark with [NEEDS CLARIFICATION: specific question] if:
         - The choice significantly impacts feature scope or user experience
         - Multiple reasonable interpretations exist with different implications
         - No reasonable default exists
       - **LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total**
       - Prioritize clarifications by impact: scope > security/privacy > user experience > technical details
    4. Fill User Scenarios & Testing section
       If no clear user flow: ERROR "Cannot determine user scenarios"
    5. Generate Functional Requirements
       Each requirement must be testable
       Use reasonable defaults for unspecified details (document assumptions in Assumptions section)
    6. Define Success Criteria
       Create measurable, technology-agnostic outcomes
       Include both quantitative metrics (time, performance, volume) and qualitative measures (user satisfaction, task completion)
       Each criterion must be verifiable without implementation details
    7. Identify Key Entities (if data involved)
    8. Return: SUCCESS (spec ready for planning)

6. Write the specification to SPEC_FILE using the template structure, replacing placeholders with concrete details derived from the feature description (arguments) while preserving section order and headings.

7. **Specification Quality Validation**: After writing the initial spec, validate it against quality criteria:

   a. **Create Spec Quality Checklist**: Generate a checklist file at `FEATURE_DIR/checklists/requirements.md` using the checklist template structure with these validation items:

      ```markdown
      # Specification Quality Checklist: [FEATURE NAME]
      
      **Purpose**: Validate specification completeness and quality before proceeding to planning
      **Created**: [DATE]
      **Feature**: [Link to spec.md]
      
      ## Content Quality
      
      - [ ] No implementation details (languages, frameworks, APIs)
      - [ ] Focused on user value and business needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      
      ## Requirement Completeness
      
      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] Requirements are testable and unambiguous
      - [ ] Success criteria are measurable
      - [ ] Success criteria are technology-agnostic (no implementation details)
      - [ ] All acceptance scenarios are defined
      - [ ] Edge cases are identified
      - [ ] Scope is clearly bounded
      - [ ] Dependencies and assumptions identified
      
      ## Feature Readiness
      
      - [ ] All functional requirements have clear acceptance criteria
      - [ ] User scenarios cover primary flows
      - [ ] Feature meets measurable outcomes defined in Success Criteria
      - [ ] No implementation details leak into specification
      
      ## Notes
      
      - Items marked incomplete require spec updates before `/speckit.clarify` or `/speckit.plan`
      ```

   b. **Run Validation Check**: Review the spec against each checklist item:
      - For each item, determine if it passes or fails
      - Document specific issues found (quote relevant spec sections)

   c. **Handle Validation Results**:

      - **If all items pass**: Mark checklist complete and proceed to step 8

      - **If items fail (excluding [NEEDS CLARIFICATION])**:
        1. List the failing items and specific issues
        2. Update the spec to address each issue
        3. Re-run validation until all items pass (max 3 iterations)
        4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user

      - **If [NEEDS CLARIFICATION] markers remain**:
        1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
        2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical (by scope/security/UX impact) and make informed guesses for the rest
        3. For each clarification needed (max 3), present options to user in this format:

           ```markdown
           ## Question [N]: [Topic]
           
           **Context**: [Quote relevant spec section]
           
           **What we need to know**: [Specific question from NEEDS CLARIFICATION marker]
           
           **Suggested Answers**:
           
           | Option | Answer | Implications |
           |--------|--------|--------------|
           | A      | [First suggested answer] | [What this means for the feature] |
           | B      | [Second suggested answer] | [What this means for the feature] |
           | C      | [Third suggested answer] | [What this means for the feature] |
           | Custom | Provide your own answer | [Explain how to provide custom input] |
           
           **Your choice**: _[Wait for user response]_
           ```

        4. **CRITICAL - Table Formatting**: Ensure markdown tables are properly formatted:
           - Use consistent spacing with pipes aligned
           - Each cell should have spaces around content: `| Content |` not `|Content|`
           - Header separator must have at least 3 dashes: `|--------|`
           - Test that the table renders correctly in markdown preview
        5. Number questions sequentially (Q1, Q2, Q3 - max 3 total)
        6. Present all questions together before waiting for responses
        7. Wait for user to respond with their choices for all questions (e.g., "Q1: A, Q2: Custom - [details], Q3: B")
        8. Update the spec by replacing each [NEEDS CLARIFICATION] marker with the user's selected or provided answer
        9. Re-run validation after all clarifications are resolved

   d. **Update Checklist**: After each validation iteration, update the checklist file with current pass/fail status

8. Report completion with branch name, spec file path, checklist results, and readiness for the next phase (`/speckit.clarify` or `/speckit.plan`).

**NOTE:** The script creates and checks out the new branch and initializes the spec file before writing.

## General Guidelines

## Quick Guidelines

- Focus on **WHAT** users need and **WHY**.
- Avoid HOW to implement (no tech stack, APIs, code structure).
- Written for business stakeholders, not developers.
- DO NOT create any checklists that are embedded in the spec. That will be a separate command.

### Section Requirements

- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant to the feature
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

### For AI Generation

When creating this spec from a user prompt:

1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Assumptions section
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers - use only for critical decisions that:
   - Significantly impact feature scope or user experience
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
4. **Prioritize clarifications**: scope > security/privacy > user experience > technical details
5. **Think like a tester**: Every vague requirement should fail the "testable and unambiguous" checklist item
6. **Common areas needing clarification** (only if no reasonable default exists):
   - Feature scope and boundaries (include/exclude specific use cases)
   - User types and permissions (if multiple conflicting interpretations possible)
   - Security/compliance requirements (when legally/financially significant)

**Examples of reasonable defaults** (don't ask about these):

- Data retention: Industry-standard practices for the domain
- Performance targets: Standard web/mobile app expectations unless specified
- Error handling: User-friendly messages with appropriate fallbacks
- Authentication method: Standard session-based or OAuth2 for web apps
- Integration patterns: RESTful APIs unless specified otherwise

### Success Criteria Guidelines

Success criteria must be:

1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective, not system internals
4. **Verifiable**: Can be tested/validated without knowing implementation details

**Good examples**:

- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"
- "95% of searches return results in under 1 second"
- "Task completion rate improves by 40%"

**Bad examples** (implementation-focused):

- "API response time is under 200ms" (too technical, use "Users see results instantly")
- "Database can handle 1000 TPS" (implementation detail, use user-facing metric)
- "React components render efficiently" (framework-specific)
- "Redis cache hit rate above 80%" (technology-specific)
