---
description: "Verify that an implementation is complete, deliverables are present, and TESTING.md checks pass"
scripts:
  sh: scripts/bash/check-prerequisites.sh --json --require-tasks --include-tasks
  ps: scripts/powershell/check-prerequisites.ps1 -Json -RequireTasks -IncludeTasks
handoffs:
  - label: Fix incomplete tasks
    agent: speckit.implement
    prompt: "Complete the remaining tasks identified by speckit.verify"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Goal

Verify that the implementation is complete by checking task completion, deliverable presence, and — when a TESTING.md file exists — executing its CLI and browser test tiers. Produce a structured VERIFICATION_REPORT.md in the feature directory.

## Outline

### 1. Prerequisites

Run `{SCRIPT}` from repo root and parse FEATURE_DIR and AVAILABLE_DOCS list. All paths must be absolute. Requires tasks.md to exist.
For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

### 2. Load Artifacts

Read from FEATURE_DIR:

- **REQUIRED**: `tasks.md` — task completion baseline
- **IF EXISTS**: `TESTING.md` — enables CLI and browser verification tiers
- **IF EXISTS**: `spec.md` — user story coverage baseline
- **IF EXISTS**: `plan.md` — service URLs used in the browser tier

### 3. Task Completion Check *(always runs)*

Count checkbox states in tasks.md:

- Completed: lines matching `- [X]` or `- [x]`
- Incomplete: lines matching `- [ ]`

**Evaluation:**

- 100% complete = **PASS**
- Any `- [ ]` remaining = **FAIL** — list every incomplete task (ID and description)

On failure, output the handoff prompt for `speckit.implement`:

> Some tasks are incomplete. Run `/speckit.implement` to complete the remaining tasks identified above.

**Continue to Steps 4-7 even if this step fails** to give a complete picture.

### 4. Deliverable Presence Check *(always runs)*

Check for common artifacts in FEATURE_DIR and report each as PRESENT or MISSING:

| Artifact | Path | Status | Severity |
|----------|------|--------|----------|
| TESTING.md | FEATURE_DIR/TESTING.md | PRESENT / MISSING | WARN |
| research.md | FEATURE_DIR/research.md | PRESENT / MISSING | INFO |
| data-model.md | FEATURE_DIR/data-model.md | PRESENT / MISSING | INFO |
| contracts/ | FEATURE_DIR/contracts/ | PRESENT / MISSING | INFO |
| quickstart.md | FEATURE_DIR/quickstart.md | PRESENT / MISSING | INFO |

- TESTING.md absence is **WARN** (not FAIL) — backwards compatible with older features that predate the testing template
- Other artifacts are **INFO** only (their presence depends on the feature scope)

### 5. CLI Verification Tier *(only if TESTING.md exists)*

Parse TESTING.md and identify bash/shell code blocks with their surrounding section context.

For each command block:

1. **Service dependency check**: If the command depends on external services (database URLs, network calls, running servers), ping the service first. If unreachable, mark the check **SKIP** with the reason.
2. **Execute**: Run the command and capture stdout, stderr, and exit code.
3. **Semantic comparison**: Compare output against any expected-output description in TESTING.md. Judge whether the output **demonstrates the expected behavior** — not a literal string match. Handle formatting differences (ANSI colors, timestamps, absolute paths, platform-specific line endings) gracefully.

Per-section result:

- ✅ **PASS** — output demonstrates expected behavior
- ❌ **FAIL** — actual output contradicts expected behavior (include actual vs expected)
- ⏭ **SKIP** — dependency unreachable or precondition unmet (include reason)

### 6. Browser Verification Tier *(only if TESTING.md has UI sections AND agent has browser automation tools)*

**Detection**: Scan TESTING.md for UI/browser sections by looking for keywords: "browser", "UI", "navigate to", "open", "click", "page", or URLs starting with `http`.

- **No UI sections found**: Skip this tier entirely. Note "No UI sections in TESTING.md".
- **UI sections found**:

  1. **Service reachability**: Extract URLs from TESTING.md or plan.md. If services are down, mark all UI steps **SKIP**.
  2. **Browser tools available**: Execute UI steps described in TESTING.md.
     - If the user passed `--record-gif` in arguments AND the agent has GIF recording tools: record a GIF of the session, name it meaningfully (e.g., `verify-{feature-name}.gif`).
  3. **Browser tools unavailable**: Mark all UI steps as requiring manual verification and provide exact steps for the user to follow.

Per-step result:

- ✅ **PASS** — UI behavior matches expected outcome
- ❌ **FAIL** — UI behavior contradicts expected outcome (describe actual vs expected)
- ⏭ **SKIP** — service unreachable or precondition unmet (include reason)
- 📋 **MANUAL** — browser tools unavailable (provide exact manual steps)

### 7. Generate VERIFICATION_REPORT.md

Write `{FEATURE_DIR}/VERIFICATION_REPORT.md` with this structure:

```markdown
## Verification Report — {feature-name}

**Date**: {date}  **Branch**: {git-branch}  **Feature**: {FEATURE_DIR name}

### Summary

| Tier         | Checks | Pass | Fail | Skip |
|--------------|--------|------|------|------|
| Completion   | N      | N    | 0    | 0    |
| Deliverables | N      | N    | N    | 0    |
| CLI          | N      | N    | N    | N    |
| Browser      | N      | N    | N    | N    |

**Verdict**: ✅ PASS / ⚠️ WARN / ❌ FAIL
```

**Verdict rules:**

- ✅ **PASS**: Completion tier passes AND no FAIL in any tier
- ⚠️ **WARN**: Completion tier passes but TESTING.md is missing OR all non-completion checks were skipped
- ❌ **FAIL**: Any FAIL in the completion tier, OR any FAIL in the CLI or browser tier

**Report sections (after summary):**

#### Failed Checks

For each failure: tier, check ID, actual output, expected behavior.

#### Skipped Checks

For each skip: tier, check ID, reason, manual action if needed.

#### Warnings

TESTING.md absent, services unreachable, browser tools unavailable, etc.

#### Next Actions

Concrete next steps based on results:

- Completion failures → handoff to `speckit.implement`
- CLI failures → specific commands to run or code to fix
- Browser failures → manual steps or service startup instructions
- All pass → "Implementation verified. Ready for review."

Report the final verdict in the conversation **and** write the VERIFICATION_REPORT.md file.

## Operating Principles

- **TESTING.md absence is WARN, not FAIL** — maintains backwards compatibility with features that predate the testing template
- **Semantic output comparison** — judge whether output demonstrates the expected behavior; handle formatting differences (ANSI colors, timestamps, absolute paths) gracefully
- **Always complete all tiers** — if Step 3 (completion) fails, still run Steps 4-7 to give a full picture
- **Service health first** — always check service reachability before attempting CLI or browser commands that depend on external services
- **Minimal side effects** — this command reads tasks.md and TESTING.md, executes test commands, and writes one file (VERIFICATION_REPORT.md). It does not modify implementation code or task status
