# Mode 2: Git-Based Update & Change Detection

Defines how the agent detects changes via git and incrementally updates only affected documentation.

---

## Table of Contents
- [Trigger Phrases](#trigger-phrases)
- [Git Change Detection Strategies](#git-change-detection-strategies)
- [Update Workflow](#update-workflow)
- [File Classification Rules](#file-classification-rules)
- [Change Detection Matrix](#change-detection-matrix)
- [Update Granularity](#update-granularity)
- [Branch & PR Mode](#branch--pr-mode)
- [Changelog Auto-Update](#changelog-auto-update)
- [LLM-Awareness Artifact Updates](#llm-awareness-artifact-updates)
- [Documentation Health Check](#documentation-health-check)
- [Update Report Template](#update-report-template)

---

## Trigger Phrases

Activate **Mode 2 (Git-Based Update)** when the user says any of:
- "update docs" / "update documentation"
- "refresh documentation" / "sync docs"
- "docs are outdated"
- "update docs from git"
- "update docs for branch {name}"
- "update docs for PR #{number}"
- "update docs since last commit"
- "update docs for commit {hash}"
- "update docs for commits {hash1}..{hash2}"
- "what changed? update docs"
- "check if docs are up to date"

---

## Git Change Detection Strategies

The agent selects the appropriate git strategy based on the user request:

### Strategy 1: Working Tree Changes (Default)
**When:** User says "update docs" without specifying a scope
**Commands:**
```bash
# Unstaged changes
git diff --name-only

# Staged changes
git diff --name-only --cached

# Untracked new files
git ls-files --others --exclude-standard

# Combined: all uncommitted changes
git status --porcelain
```
**Use case:** Developer made local changes and wants docs updated before committing.

### Strategy 2: Last N Commits
**When:** User says "update docs since last commit" or "update docs for last 3 commits"
**Commands:**
```bash
# Last 1 commit
git diff --name-only HEAD~1

# Last N commits
git diff --name-only HEAD~{N}

# With change type (Added/Modified/Deleted/Renamed)
git diff --name-status HEAD~{N}
```
**Use case:** Developer committed recently and wants docs to catch up.

### Strategy 3: Branch Comparison
**When:** User says "update docs for branch feature/xyz"
**Commands:**
```bash
# Find merge base and diff
git merge-base main {branch}
git diff --name-only $(git merge-base main {branch})..{branch}

# Shorthand
git diff --name-only main..{branch}

# With change type
git diff --name-status main..{branch}

# List commits on branch
git log --oneline main..{branch}
```
**Use case:** Feature branch is ready for review; generate/update docs for all changes on the branch.

### Strategy 4: Commit Range
**When:** User says "update docs for commits abc123..def456"
**Commands:**
```bash
git diff --name-only {commit1}..{commit2}
git diff --name-status {commit1}..{commit2}
git log --oneline {commit1}..{commit2}
```
**Use case:** Update docs for a specific range of work.

### Strategy 5: PR Mode
**When:** User says "update docs for PR #42"
**Commands:**
```bash
# If PR branch is checked out locally
git diff --name-only main..HEAD

# If using GitHub CLI
gh pr diff {number} --name-only

# Get PR details
gh pr view {number} --json title,body,changedFiles,additions,deletions
```
**Use case:** Reviewing a PR and need docs updated to match the PR changes.

### Strategy 6: Specific Files
**When:** User says "update docs - I changed UserService.java"
**Commands:**
```bash
# Get diff for specific file
git diff {file_path}

# Compare with main branch
git diff main -- {file_path}
```
**Use case:** Developer knows exactly which file changed.

---

## Update Workflow

### Step 1: Detect Changes
```
Run appropriate git command based on user request
Parse output into a change list:
  [{ file: "src/.../UserService.java", status: "modified" },
   { file: "src/.../PaymentController.java", status: "added" },
   { file: "src/.../OldDto.java", status: "deleted" }]
```

### Step 2: Classify Changed Files
Map each changed file to its category using the File Classification Rules below.

### Step 3: Map to Affected Docs
Use the Change Detection Matrix to determine which doc files need updating.

### Step 4: Read Context
- Read each changed source file (current version)
- Read each affected doc file (existing version)
- For deleted files: note removal from docs
- For renamed files: update all references

### Step 5: Show Change Plan
Present to the user:

```
=============================================
  Documentation Update Plan
=============================================
Git Scope: {strategy description}
Changed Source Files: {count}

| # | File | Status | Category |
|---|------|--------|----------|
| 1 | src/.../UserService.java | Modified | Service |
| 2 | src/.../PaymentController.java | Added | Controller |

Docs to Update: {count}

| # | Document | Reason |
|---|----------|--------|
| 1 | docs/services/SERVICE_REFERENCE.md | UserService modified |
| 2 | docs/api/API_REFERENCE.md | PaymentController added |
| 3 | docs/knowledge-graph/METHOD_SUMMARY.md | Methods changed |
| 4 | docs/guides/CHANGELOG.md | New changes to log |

Proceed with update? (yes/no)
=============================================
```

### Step 6: Execute Updates
- Regenerate affected documentation sections
- Update timestamps
- Update CHANGELOG.md
- Update DOCUMENTATION_INDEX.md if structure changed

### Step 7: Show Report
Use the Update Report Template (see below).

---

## File Classification Rules

Classify each changed file by matching its path:

| Path Pattern | Category |
|--------------|----------|
| `**/controller/**` or `*Controller.java` | Controller |
| `**/service/**` or `*Service.java` or `*ServiceImpl.java` | Service |
| `**/model/**` or `**/entity/**` or `**/domain/**` | Entity |
| `**/dto/**` or `*Request.java` or `*Response.java` or `*Dto.java` | DTO |
| `**/exception/**` or `*Exception.java` or `*ExceptionHandler.java` | Exception |
| `**/repository/**` or `*Repository.java` | Repository |
| `**/config/**` or `*Config.java` or `*Configuration.java` | Config |
| `application.yml` or `application.properties` or `application-*.yml` | Config |
| `pom.xml` or `build.gradle` or `build.gradle.kts` | Build |
| `**/test/**` or `*Test.java` or `*Tests.java` or `*Spec.java` | Test |
| `Dockerfile` or `docker-compose*.yml` | Docker |
| `.github/**` or `Jenkinsfile` or `.gitlab-ci.yml` | CI/CD |
| `*.md` (in docs/) | Documentation |
| Everything else | Other |

---

## Change Detection Matrix

Maps source file categories to the documentation files that must be updated:

### Controller Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Endpoint added | API_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md, ENDPOINT_FLOW_GRAPH.md, SEQUENCE_DIAGRAMS.md |
| Endpoint modified | API_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md, ENDPOINT_FLOW_GRAPH.md |
| Endpoint deleted | API_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md, ENDPOINT_FLOW_GRAPH.md |
| New controller class | API_REFERENCE.md, CLASS_DIAGRAM.md, COMPONENT_DIAGRAM.md, SERVICE_KNOWLEDGE_GRAPH.md, CALL_GRAPH.md |
| Controller deleted | API_REFERENCE.md, CLASS_DIAGRAM.md, COMPONENT_DIAGRAM.md, SERVICE_KNOWLEDGE_GRAPH.md |

### Service Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Method added | SERVICE_REFERENCE.md, METHOD_SUMMARY.md, CALL_GRAPH.md, BUSINESS_RULES.md |
| Method modified | SERVICE_REFERENCE.md, METHOD_SUMMARY.md, CALL_GRAPH.md, BUSINESS_RULES.md |
| Method deleted | SERVICE_REFERENCE.md, METHOD_SUMMARY.md, CALL_GRAPH.md |
| New service class | SERVICE_REFERENCE.md, CLASS_DIAGRAM.md, SERVICE_KNOWLEDGE_GRAPH.md, DEPENDENCY_GRAPH.md |
| Business rule changed | BUSINESS_RULES.md, SERVICE_REFERENCE.md |
| Transaction boundary changed | SERVICE_REFERENCE.md, DATA_FLOW.md |

### Entity / Model Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Field added/removed | ENTITY_REFERENCE.md, DATABASE_SCHEMA.md, ENTITY_RELATIONSHIP.md |
| Relationship changed | ENTITY_REFERENCE.md, DATABASE_SCHEMA.md, ENTITY_RELATIONSHIP.md, SERVICE_KNOWLEDGE_GRAPH.md |
| Enum modified | ENTITY_REFERENCE.md, DTO_REFERENCE.md |
| New entity | ENTITY_REFERENCE.md, DATABASE_SCHEMA.md, ENTITY_RELATIONSHIP.md, CLASS_DIAGRAM.md |
| Validation added | ENTITY_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md |

### DTO Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Field added/removed | DTO_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md |
| Validation changed | REQUEST_RESPONSE_SCHEMAS.md, DTO_REFERENCE.md |
| New DTO class | DTO_REFERENCE.md, REQUEST_RESPONSE_SCHEMAS.md, CLASS_DIAGRAM.md, SERVICE_KNOWLEDGE_GRAPH.md |

### Exception Changes
| Source Change | Docs to Update |
|---------------|----------------|
| New exception class | ERROR_CODES.md, EXCEPTION_HANDLING.md, EXCEPTION_FLOW.md, SERVICE_KNOWLEDGE_GRAPH.md |
| Handler mapping changed | ERROR_CODES.md, EXCEPTION_HANDLING.md, EXCEPTION_FLOW.md |
| Error response changed | ERROR_CODES.md, API_REFERENCE.md |

### Repository Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Query method added | SERVICE_REFERENCE.md, METHOD_SUMMARY.md, CALL_GRAPH.md |
| Custom query changed | SERVICE_REFERENCE.md, DATABASE_SCHEMA.md |
| New repository | CLASS_DIAGRAM.md, SERVICE_KNOWLEDGE_GRAPH.md, DEPENDENCY_GRAPH.md |

### Configuration Changes
| Source Change | Docs to Update |
|---------------|----------------|
| application.yml changed | APP_CONFIGURATION.md |
| Profile added | APP_CONFIGURATION.md, GETTING_STARTED.md |
| Port/URL changed | APP_CONFIGURATION.md, API_REFERENCE.md, GETTING_STARTED.md |
| Database config changed | APP_CONFIGURATION.md, DATABASE_SCHEMA.md |

### Build File Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Dependency added/removed | DEPENDENCIES.md, BUILD_SETUP.md |
| Plugin changed | BUILD_SETUP.md |
| Java version changed | BUILD_SETUP.md, GETTING_STARTED.md, DEVELOPMENT_GUIDE.md |

### Test Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Test class added | TEST_REFERENCE.md, TEST_COVERAGE_REPORT.md |
| Test method added | TEST_REFERENCE.md, TEST_COVERAGE_REPORT.md |
| Test class deleted | TEST_REFERENCE.md, TEST_COVERAGE_REPORT.md |

### Javadoc Changes
| Source Change | Docs to Update |
|---------------|----------------|
| Javadoc added/updated | METHOD_SUMMARY.md (re-verify accuracy) |

### Cross-Cutting (Always Update)
| Condition | Docs to Update |
|-----------|----------------|
| Any source file changed | CHANGELOG.md (append change entry) |
| New file added to project | DOCUMENTATION_INDEX.md, SERVICE_KNOWLEDGE_GRAPH.md |
| File deleted from project | DOCUMENTATION_INDEX.md, SERVICE_KNOWLEDGE_GRAPH.md |
| File renamed | All docs referencing old path |

### LLM-Awareness Artifacts (Always Update)
| Condition | Artifacts to Update |
|-----------|---------------------|
| Any endpoint added/modified/deleted | `docs/CONTEXT.md` — Section 5 (REST API Endpoints) |
| Any business rule changed | `docs/CONTEXT.md` — Section 8 (Business Rules) |
| Any service method added/modified | `docs/CONTEXT.md` — Section 9 (Key Method Call Chains), `docs/knowledge-graph/CALL_GRAPH.md` — Natural Language Call Summaries |
| Any exception or handler changed | `docs/CONTEXT.md` — Section 10 (Exception Handling) |
| Any entity field added/removed | `docs/CONTEXT.md` — Section 7 (User Entity) |
| Any Kafka topic/consumer added/changed | `docs/CONTEXT.md` — Section 6 (Kafka Integration) |
| Any config property changed | `docs/CONTEXT.md` — Section 11 (Configuration Properties), `llms.txt` |
| TODO resolved or new TODO added | `docs/CONTEXT.md` — Section 12 (Known TODOs) |
| New class added | `docs/CONTEXT.md` — Section 4 (All Classes), `llms.txt` source file list |
| Class deleted | `docs/CONTEXT.md` — Section 4 (All Classes), `llms.txt` source file list |
| New call flow introduced | `docs/knowledge-graph/CALL_GRAPH.md` — add new Natural Language Call Summary |

---

## Update Granularity

When updating a doc, prefer **surgical updates** over full regeneration:

### Section-Level Updates
If a single method changed in UserService:
- **DO**: Update only the UserService section in SERVICE_REFERENCE.md
- **DO**: Update only the affected method row in METHOD_SUMMARY.md
- **DO NOT**: Regenerate the entire SERVICE_REFERENCE.md

### Full Regeneration Triggers
Regenerate the entire doc file when:
- More than 50% of rows/sections in the doc are affected
- A new class is added to the project (need new sections)
- A class is deleted (need to remove sections + references)
- The doc template itself was updated

### Timestamp Updates
Every updated doc MUST have its timestamp refreshed:
```markdown
> Auto-generated on {NEW_DATE} by Documentation Agent
> Last updated: {NEW_DATE} | Reason: {brief_change_description}
```

---

## Branch & PR Mode

### Branch Documentation
When updating docs for a feature branch:

1. **Show branch info:**
   ```
   Branch: feature/payment-gateway
   Base: main
   Commits: 12
   Changed files: 8
   ```

2. **Classify all changes on the branch** (not just the latest commit)
3. **Generate a Branch Change Summary** before updating docs:
   ```
   Branch Change Summary: feature/payment-gateway
   ================================================
   New Files:
     + src/.../PaymentController.java (Controller)
     + src/.../PaymentService.java (Service)
     + src/.../Payment.java (Entity)

   Modified Files:
     ~ src/.../UserService.java (Service)
     ~ src/.../application.yml (Config)

   Deleted Files:
     - src/.../LegacyPaymentHelper.java (Service)
   ================================================
   ```

### PR Documentation
When updating docs for a PR:

1. **Fetch PR metadata** (title, description, labels)
2. **Get PR changed files** via `gh pr diff` or `git diff main..HEAD`
3. **Include PR context** in CHANGELOG entry:
   ```markdown
   ### [{DATE}] PR #{number}: {title}
   - Added: {description of additions}
   - Modified: {description of changes}
   - Removed: {description of removals}
   ```

---

## Changelog Auto-Update

When updating docs via Mode 2, **always** append to CHANGELOG.md:

### From Git Log
```bash
# Get commit messages for changelog
git log --oneline --no-merges HEAD~{N}..HEAD
git log --oneline --no-merges main..{branch}
```

### Changelog Entry Format
```markdown
## [{DATE}] Documentation Update -- {scope}

### Source Changes
| Commit | Message | Files |
|--------|---------|-------|
| {short_hash} | {message} | {count} |

### Documentation Updates
| Document | Action | Reason |
|----------|--------|--------|
| API_REFERENCE.md | Updated | New endpoint added |
| SERVICE_REFERENCE.md | Updated | UserService.processPayment() added |
| CLASS_DIAGRAM.md | Regenerated | New PaymentController class |
```

---

## Documentation Health Check

When the user says "check if docs are up to date":

### Step 1: Get Last Doc Generation Timestamps
Parse the `Auto-generated on` or `Last updated` line from each doc file.

### Step 2: Get Source File Timestamps
```bash
# Last modified time per source file
git log -1 --format="%ai" -- {file_path}
```

### Step 3: Compare & Report
```
=============================================
  Documentation Health Check
=============================================
[OK]      Up to date:  API_REFERENCE.md
[OK]      Up to date:  ENTITY_REFERENCE.md
[STALE]   Stale:       SERVICE_REFERENCE.md
             -> UserService.java modified {date} (after doc generated {date})
[STALE]   Stale:       METHOD_SUMMARY.md
             -> Javadoc changes detected in UserService.java
[MISSING] Missing:     CALL_GRAPH.md (never generated)
[NEW]     New source:  PaymentController.java (not in any docs)
=============================================
Summary: 2 current | 2 stale | 1 missing | 1 undocumented

Would you like me to update stale docs and generate missing ones?
=============================================
```

### Step 4: Act on User Response
- "yes" -> Update stale + generate missing docs
- "only stale" -> Update only stale docs
- "only missing" -> Generate only missing docs
- "no" -> Do nothing

---

## Update Report Template

After completing a Mode 2 update, show this report:

```
=============================================
  Documentation Update Complete
=============================================
Git Scope: {strategy -- e.g., "Working tree changes" or "Branch: feature/xyz vs main"}
Source Files Changed: {count}
Docs Updated: {count}
Docs Unchanged: {count}

| Document | Action | Reason |
|----------|--------|--------|
| API_REFERENCE.md | Updated | UserController: 2 endpoints modified |
| SERVICE_REFERENCE.md | Updated | UserService: 1 method added |
| CLASS_DIAGRAM.md | Regenerated | New class added |
| CHANGELOG.md | Appended | Change log entry added |
| DOCUMENTATION_INDEX.md | Unchanged | No structural changes |

Timestamps refreshed on {count} documents
CHANGELOG.md updated with {count} entries

Next: Run "check if docs are up to date" to verify.
=============================================
```

---

## Edge Cases

### No Changes Detected
If git shows no changes:
```
No source file changes detected.
All documentation appears to be up to date.
Run "check if docs are up to date" for a detailed health check.
```

### Docs Folder Does Not Exist
If `docs/` is missing:
```
No existing documentation found. Switching to Mode 1 (Full Scan).
```

### Non-Source Files Changed
If only non-source files changed (README.md, .gitignore, scripts):
```
Only non-source files changed. No documentation updates needed.
Changed files: {list}
```
Exception: If `pom.xml`, `build.gradle`, `Dockerfile`, or CI/CD files changed, still update relevant docs.

### Merge Conflicts in Docs
If docs have uncommitted changes:
```
The following doc files have uncommitted changes:
  - docs/api/API_REFERENCE.md
  - docs/services/SERVICE_REFERENCE.md

Overwrite with regenerated content? (yes/no/skip)
```

### Renamed Source Files
If a source file was renamed:
1. Find all docs referencing the old file path
2. Update all references to the new path
3. Report the renames in the update report

---

## Update Granularity

When updating a doc, prefer **surgical updates** over full regeneration:

### Section-Level Updates
If a single method changed in UserService:
- **DO**: Update only the UserService section in SERVICE_REFERENCE.md
- **DO**: Update only the affected method row in METHOD_SUMMARY.md
- **DO NOT**: Regenerate the entire SERVICE_REFERENCE.md

### Full Regeneration Triggers
Regenerate the entire doc file when:
- More than 50% of rows/sections in the doc are affected
- A new class is added to the project (need new sections)
- A class is deleted (need to remove sections + references)
- The doc template itself was updated

### Timestamp Updates
Every updated doc MUST have its timestamp refreshed:
```markdown
> Auto-generated on {NEW_DATE} by Documentation Agent
> Last updated: {NEW_DATE} | Reason: {brief_change_description}
```

---

## Branch & PR Mode

### Branch Documentation
When updating docs for a feature branch:

1. **Show branch info:**
   ```
   Branch: feature/payment-gateway
   Base: main
   Commits: 12
   Changed files: 8
   ```

2. **Classify all changes on the branch** (not just the latest commit)
3. **Generate a Branch Change Summary** before updating docs:
   ```
   Branch Change Summary: feature/payment-gateway
   ================================================
   New Files:
     + src/.../PaymentController.java (Controller)
     + src/.../PaymentService.java (Service)
     + src/.../Payment.java (Entity)

   Modified Files:
     ~ src/.../UserService.java (Service)
     ~ src/.../application.yml (Config)

   Deleted Files:
     - src/.../LegacyPaymentHelper.java (Service)
   ================================================
   ```

### PR Documentation
When updating docs for a PR:

1. **Fetch PR metadata** (title, description, labels)
2. **Get PR changed files** via `gh pr diff` or `git diff main..HEAD`
3. **Include PR context** in CHANGELOG entry:
   ```markdown
   ### [{DATE}] PR #{number}: {title}
   - Added: {description of additions}
   - Modified: {description of changes}
   - Removed: {description of removals}
   ```

---

## Changelog Auto-Update

When updating docs via Mode 2, **always** append to CHANGELOG.md:

### From Git Log
```bash
# Get commit messages for changelog
git log --oneline --no-merges HEAD~{N}..HEAD
git log --oneline --no-merges main..{branch}
```

### Changelog Entry Format
```markdown
## [{DATE}] Documentation Update -- {scope}

### Source Changes
| Commit | Message | Files |
|--------|---------|-------|
| {short_hash} | {message} | {count} |

### Documentation Updates
| Document | Action | Reason |
|----------|--------|--------|
| API_REFERENCE.md | Updated | New endpoint added |
| SERVICE_REFERENCE.md | Updated | UserService.processPayment() added |
| CLASS_DIAGRAM.md | Regenerated | New PaymentController class |
```


## LLM-Awareness Artifact Updates

The project maintains three files specifically for LLM and AI assistant consumption. Keep these in sync after every code change.

### File: `llms.txt` (project root)
**Purpose:** Priority-ordered reading guide for AI assistants — equivalent to `robots.txt` for LLMs.

**When to update:**
- New source files added → add to the source file list section
- Source files deleted → remove from source file list
- Tech stack changes (new framework, Java version bump) → update the tech summary line
- Key config properties change (port, base URL) → update the description paragraph
- Version bumped → update the `Version:` header line

**How to update:** Edit only the affected lines — do NOT regenerate the entire file. Update the `# Updated:` date line at the top.

---

### File: `docs/CONTEXT.md`
**Purpose:** Single-file complete LLM context dump — everything an AI needs to understand and reason about the codebase in one file.

**When to update (section-level):**

| Section | Update when... |
|---------|----------------|
| 1. What This Service Does | Core responsibility changes, auth added, new major feature |
| 2. Technology Stack | Dependency added/removed, Java version changed |
| 3. Architecture | New layer added (e.g., cache, messaging), pattern change |
| 4. All Classes | Class added, deleted, or renamed |
| 5. REST API Endpoints | Endpoint added, modified, deleted, auth added to a route |
| 6. Kafka Integration | New topic, consumer group changed, retry policy changed |
| 7. Entity Fields | Field added/removed, constraint changed |
| 8. Business Rules | New validation rule, existing rule changed |
| 9. Method Call Chains | New flow added, existing flow steps changed |
| 10. Exception Handling | New exception, handler mapping changed |
| 11. Config Properties | New property, default changed, new env override |
| 12. Known TODOs | TODO resolved, new planned feature added |
| 13. File Map | New source file added or deleted |
| 14. Quick Answers | New common question emerges from the change |

**How to update:** Use surgical section-level edits. Do NOT regenerate the entire file unless > 50% of sections are affected. Always update the date line at the top. Mark inferred info with `[INFERRED]`.

---

### File: `docs/knowledge-graph/CALL_GRAPH.md` — Natural Language Section
**Purpose:** Prose descriptions of each major execution flow for LLM reasoning and code review.

**When to update:**
- New call chain introduced → add a new `### Flow N: {name}` subsection
- Existing flow steps changed → update that flow's paragraph
- Flow removed → remove that flow subsection

**Each flow must include:** entry point, delegation chain in plain English, failure path (exception → HTTP status), and any architectural insights. Write full prose — no bullet lists.

---

### Automation Checklist for LLM Files

After every Mode 2 update, verify before closing:

- [ ] `llms.txt` — version/date header current, source file list accurate
- [ ] `docs/CONTEXT.md` — all affected sections updated, date header updated
- [ ] `CALL_GRAPH.md` Natural Language section — new flows added, changed flows edited

Include in the Update Report as a separate **LLM Artifacts** row group.

---
---

## Documentation Health Check

When the user says "check if docs are up to date":

### Step 1: Get Last Doc Generation Timestamps
Parse the `Auto-generated on` or `Last updated` line from each doc file.

### Step 2: Get Source File Timestamps
```bash
# Last modified time per source file
git log -1 --format="%ai" -- {file_path}
```

### Step 3: Compare & Report
```
=============================================
  Documentation Health Check
=============================================
[OK]      Up to date:  API_REFERENCE.md
[OK]      Up to date:  ENTITY_REFERENCE.md
[STALE]   Stale:       SERVICE_REFERENCE.md
             -> UserService.java modified {date} (after doc generated {date})
[STALE]   Stale:       METHOD_SUMMARY.md
             -> Javadoc changes detected in UserService.java
[MISSING] Missing:     CALL_GRAPH.md (never generated)
[NEW]     New source:  PaymentController.java (not in any docs)
=============================================
Summary: 2 current | 2 stale | 1 missing | 1 undocumented

Would you like me to update stale docs and generate missing ones?
=============================================
```

### Step 4: Act on User Response
- "yes" -> Update stale + generate missing docs
- "only stale" -> Update only stale docs
- "only missing" -> Generate only missing docs
- "no" -> Do nothing

---

## Update Report Template

After completing a Mode 2 update, show this report:

```
=============================================
  Documentation Update Complete
=============================================
Git Scope: {strategy -- e.g., "Working tree changes" or "Branch: feature/xyz vs main"}
Source Files Changed: {count}
Docs Updated: {count}
Docs Unchanged: {count}

| Document | Action | Reason |
|----------|--------|--------|
| API_REFERENCE.md | Updated | UserController: 2 endpoints modified |
| SERVICE_REFERENCE.md | Updated | UserService: 1 method added |
| CLASS_DIAGRAM.md | Regenerated | New class added |
| CHANGELOG.md | Appended | Change log entry added |
| DOCUMENTATION_INDEX.md | Unchanged | No structural changes |

Timestamps refreshed on {count} documents
CHANGELOG.md updated with {count} entries

Next: Run "check if docs are up to date" to verify.
=============================================
```

---

## Edge Cases

### No Changes Detected
If git shows no changes:
```
No source file changes detected.
All documentation appears to be up to date.
Run "check if docs are up to date" for a detailed health check.
```

### Docs Folder Does Not Exist
If `docs/` is missing:
```
No existing documentation found. Switching to Mode 1 (Full Scan).
```

### Non-Source Files Changed
If only non-source files changed (README.md, .gitignore, scripts):
```
Only non-source files changed. No documentation updates needed.
Changed files: {list}
```
Exception: If `pom.xml`, `build.gradle`, `Dockerfile`, or CI/CD files changed, still update relevant docs.

### Merge Conflicts in Docs
If docs have uncommitted changes:
```
The following doc files have uncommitted changes:
  - docs/api/API_REFERENCE.md
  - docs/services/SERVICE_REFERENCE.md

Overwrite with regenerated content? (yes/no/skip)
```

### Renamed Source Files
If a source file was renamed:
1. Find all docs referencing the old file path
2. Update all references to the new path
3. Report the renames in the update report
