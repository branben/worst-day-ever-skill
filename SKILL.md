---
name: worst-day-ever
description: Simulate worst-case user flows through a project — without breaking it or exposing secrets. Generates adversarial edge-case scenarios from project spec or traffic patterns, runs them in sandbox, produces a disaster report. Use when user says "stress test this", "what could go wrong", "torture test", "edge case simulation", or asks about robustness of user flows.
---

# Worst-Day-Ever

A disciplined adversarial user simulation: systematically generate the worst things a user could do, execute them safely, and report how the system responds. Never touches real data, secrets, or shared state.

## Core Philosophy

Every system has undiscovered user flows — combinations of inputs, timings, state transitions that no one designed for but users will find. This skill exists to surface them *before* users do, in a controlled environment.

The goal is not to be destructive. The goal is to be *honest* about how the system behaves under stress.

## Greenfield vs Brownfield

The workflow diverges based on whether the project has real traffic data:

### Greenfield (no production traffic / pre-launch)

Input sources: API specs (OpenAPI/GraphQL schema), route definitions, state machine docs, README, type definitions.

Attack surface is **derived from contracts**:
- What types does this endpoint accept? → violate them
- What states exist? → traverse them out of order
- What are the rate limits? → exceed them
- What fields are optional? → omit them. What fields are required? → send garbage.

### Brownfield (has production traffic / live system)

Input sources: production logs (sanitized), error tracking, access logs, DB schema, APM traces.

Attack surface is **mined from real patterns**:
- Which endpoints have highest 5xx rate? → amplify the malformed input that triggered it
- Which user flows have highest drop-off? → the abandoned state is probably an orphaned record
- What's the 99th percentile latency? → send a request that will time out, then check recovery
- Which records are referenced but deleted? → follow the dangling pointer

**NEVER use real PII.** Anonymize all IDs before analysis. If mining logs, replace every real ID with a synthetic one of the same format. Log structures matter; content does not.

## Safeguards

These are non-negotiable. If any cannot be met, abort and tell the user why.

1. **No real data** — all inputs synthesized. Real schemas used only for type/format inference
2. **No secrets** — if the system requires auth, create a test token. Never read `.env` or keyrings
3. **Sandboxed execution** — run in isolated subprocess / DB fake / HTTP stub. No writes to shared state
4. **No external calls** — if the system hits external services (payment gateway, email API), stub them with failure modes
5. **Bounded duration** — each flow runs max 30 seconds. If it hangs, kill it and report "hang detected"
6. **No mutation side effects** — if the system writes to a screen/file, capture the write. If it writes to a DB, roll back or use a throwaway

## Attack Surface Matrix

Execute dimensions in order. For each dimension, generate 3–5 scenarios of escalating severity. Document the result.

### Dimension 1 — Input Boundary
- Max-length strings (2× declared max)
- Null bytes embedded in text fields
- Unicode edge cases: ZWNJ (`U+200C`), RTL override (`U+202E`), 4-byte emoji (`�` / `&#x1D573;), BOM at string start
- Type confusion: `{"age": "not_a_number"}`, `{"tags": "should_be_array"}`
- Deeply nested payloads: JSON depth 50+, circular `$ref` in schemas
- Empty/whitespace-only to required fields
- Arithmetic edge cases: `0`, `-1`, `MAX_INT+1`, `NaN`, `Infinity`

### Dimension 2 — State Machine Torture
- Double-submit: replay the final mutation N times (idempotency test)
- Back-button resurrection: create → confirm → nav back to confirm → submit confirm again
- Out-of-order: try confirm before create, delete after refund, approve after reject
- Rapid transitions: submit state changes faster than processing time
- Concurrent state mutations: two clients both "claim" the same resource
- Zombie state: act on resource whose state is already terminal (shipped → try cancel)

### Dimension 3 — Temporal / Timing
- Replay valid-but-expired tokens (session history hijack — use synthetic sessions)
- Clock skew payloads: `If-Modified-Since` = `2099-01-01`, `exp` = `2000-01-01`
- Counter race: 100 simultaneous requests that each claim the "last available" slot
- Connection pool depletion: send slow requests that starve the pool, then hit critical path
- Polling abuse: hammer an endpoint that triggers expensive work on each call (e.g. report generation)

### Dimension 4 — AuthN/Z Shadow
- Horizontal escalation: increment resource IDs (user_1's order → user_1/order_2, /3, /4...)
- Vertical escalation: submit `is_admin=true` or `role=admin` in payload when token says `user`
- Scope mismatch: token with `read:orders` hitting `POST /orders` or `DELETE`
- Session fixation: reuse a pre-auth token after login (server should rotate)
- Cross-tenant: if multi-tenant, hit another tenant's resources with wrong `X-Tenant-Id`
- Missing token: drop `Authorization` entirely on protected endpoints

### Dimension 5 — Data Integrity Cascade
- Delete referenced entity: user deleted → do their scheduled jobs still run? Their orders ship?
- Encoding roundtrips: submit Latin-1 bytes, store as UTF-8, request with `Accept-Charset: ISO-8859-1`
- Pagination edge cases: `?cursor=-1`, `?offset=999999999`, `?limit=0`, `?limit=-1`
- Constraint bypass: submit invalid enum that passes validation but hits DB constraint
- Schema drift: send payload from an older API version to a newer endpoint (missing required fields)
- Reference integrity: create resource pointing to non-existent parent (should 404, may 500)

### Dimension 6 — Concurrent Load Concentration
- Thundering herd: 1000 simultaneous reads on same record at exact moment it's being mutated
- Cache death spiral: expire a hot cache entry → read misses → DB stampede
- Write skew: two transactions both read overlapping data → both commit → invariant violated
- Lock escalation: acquire many row locks → escalates to table lock → blocks readers
- Hotspot abuse: hammer a cheap-to-validate, expensive-to-execute endpoint (search with sort on unindexed column)

### Dimension 7 — External Dependency Failure
- Payment gateway down → does the order get created anyway? In what state?
- Email service timeout → does the user know their action succeeded? Can they retry safely?
- CDN serving stale asset → does the app break silently or show error?
- DNS failure on external service → does it hang or fail fast?
- Third-party returns malformed response → does validation catch it or does it crash upstream?

### Dimension 8 — Brownfield-Only (Log Mining)
- Parse production logs (sanitized): find the 5 inputs that caused the worst errors → replay with variations
- Find unused enum values in schema → submit them to see if they were removed from validation
- Soft-delete audit: find records marked deleted but still referenced by active queries
- Slow query correlation: identify the slowest 1% of requests → what input pattern caused them?
- Error message quality: does the error message leak stack traces, internal paths, or schema names?

## Execution Protocol

For each session running this skill:

```
1. MAP: Discover the interface (read specs, routes, type defs, logs)
2. GENERATE: For each dimension, produce scenario set
3. SANDBOX: Set up isolated execution environment
4. EXECUTE: Run scenarios one at a time, capture result
5. REPORT: Generate disaster report
```

### Design-First for Visual Artifacts

When the user asks for any visual output (TUI splash screen, report format, dashboard), **load the `claude-design` skill first** and use `popular-web-designs` for the visual vocabulary. Do not invent color palettes or typography systems from scratch — borrow from real design systems (Linear for terminal-native, Stripe for marketing, etc.).

### Sandbox Setup

For each project type:

| Project Type | Sandbox Method |
|---|---|
| Python/Flask/Django | In-memory SQLite, `responses` for HTTP stubs, `unittest.mock` for external services |
| Node/Express | Better-sqlite3 in-memory, `nock` for HTTP, `sinon` for timers |
| Rails | Test-mode DB, WebMock for HTTP |
| Go | `testing` package + httptest server |
| Static site | Headless browser (Playwright) in ephemeral container |
| CLI tool | tmpfs for filesystem, stubbed stdin/stdout |

If no sandbox can be constructed (hardware-embedded, proprietary runtime, no test framework), **do not execute** — proceed to dry-run mode only.

### Dry-Run Mode

When live execution isn't possible:
1. Trace the code path analytically: "This input would hit `validate_order()` at line 47, which would call `user.credits` — and since the user was deleted, this returns NULL, and `NULL - 100` evaluates to NULL which is not `< 0`, so the credit check is **bypassed**"
2. Report the hypothetical failure with confidence level: **certain** (pure logic), **likely** (depends on runtime state), **uncertain** (needs live test)
3. Mark as "recommended manual test" in the report

## Output: Disaster Report

Structure:

```markdown
# Worst-Day-Ever Report: <project>

**Date:** ...
**Mode:** [live-sandbox | dry-run | brownfield-mine]
**Dimensions tested:** N
**Total scenarios:** N

## Summary

| Severity | Count | Description |
|---|---|---|
| Critical | N | Data corruption, security bypass, unhandled crash |
| High | N | Silent wrong answer, orphan records, auth bypass |
| Medium | N | Confusing error, unexpected behavior, no crash |
| Low | N | Cosmetic, uninformative error, edge case with no impact |
| Pass | N | Graceful handling, correct error response |

## Findings

### [CRITICAL] <Title>
- **Dimension:** <which Dimension>
- **Flow:** <the pathological user flow, step by step>
- **What happened:** <system behavior observed>
- **Expected:** <what should happen>
- **Remediation:** <specific fix suggestion, file/line if known>

...repeat per finding...

## Resilience Score

**Overall:** X/10

Breakdown by dimension:
- Input Boundary: X/10
- State Machine: X/10
- Temporal/Timing: X/10
- AuthN/Z: X/10
- Data Integrity: X/10
- Concurrency: X/10
- External Deps: X/10
- Brownfield: X/10

## Recommended Tests

Auto-generated test skeletons (pytest/rspec/etc.) based on critical findings.
```

## Scoring Guidelines

When rating severity:

- **Critical**: Security breach, data loss, unhandled exception that propagates to user as 500
- **High**: Silent data corruption, auth bypass that doesn't require privilege to exploit, cascade failures
- **Medium**: Confusing UX, uninformative error messages, behavior that surprises but doesn't break
- **Low**: Error handling works but the error message is poor, edge case that's technically handled but ugly
- **Pass**: Correct response with informative message, graceful degradation, proper circuit-breaking

## Checklist

```
[ ] Identified project type and interface boundaries
[ ] Determined greenfield vs brownfield mode
[ ] Set up sandbox (or declared dry-run)
[ ] Generated scenarios for applicable dimensions
[ ] Executed / analyzed each scenario
[ ] Classified severity of each result
[ ] Generated remediation suggestions
[ ] Verified no secrets, real data, or shared state were touched
[ ] Produced disaster report
```

## Pitfalls

1. **"It worked in dev" is not proof.** The sandbox is deliberately harder than dev. If sandbox passes, the real system may still fail (different data volume, real hardware, real network).

2. **Don't confuse coverage with resilience.** A system that handles 6 out of 7 dependency failures is not "mostly resilient" — the 7th is the one that costs you at 3 AM.

3. **Null bytes in strings are not dead.** Many systems validate length but not content. A `\x00` mid-string crashes C-backed libraries downstream.

4. **Brownfield mining can leak PII if not careful.** Always sanitize logs *before* analysis. Strip emails, JWTs, credit card numbers. If you find one in a log, flag it as a separate incident.

5. **Dry-run is a lower bound, not an upper bound.** "The code path looks safe" does not mean it's safe. Flags are pessimistic — use dry-run to prioritize what to actually run.

6. **Don't skip Dimension 1 because "we validate inputs."** Input validation is the most common lie in software. The validator was written by someone who didn't anticipate Unicode RTL overrides in a username field.

7. **If the system requires auth and you can't create a test user safely, skip AuthN/Z dimension** rather than using a real user's session. Report why it was skipped.

8. **Browser tools block `file://` URLs.** When opening local HTML assets (like the TUI splash), the browser tool rejects private/internal addresses. Serve the directory with `python3 -m http.server <port>` and navigate to `http://localhost:<port>/` instead.

## TUI Splash Screen

A self-contained terminal-style animation is available at `assets/tui.html`. Open it in a browser to show users a visual "worst day ever" splash while the skill runs.

**Design:** Built using the [Linear](https://linear.app) design system (via `popular-web-designs`). Near-black `#08090a` canvas, Inter + JetBrains Mono typography, luminance-based elevation, hostile-red accents for critical findings. Avoids CRT gimmicks (scanlines, glow) in favor of precise dark-mode design.

**Features:**
- Large, prominent "WORST DAY EVER" title (42px Inter weight-600)
- ASCII frown face with JS-driven eye blink (`◉`→─, every 2.5-5.5s)
- 3×2 status grid: uptime, critical count, resilience score, dimensions, mode, last event
- Auto-scrolling event stream with timestamp + OK/WARN/CRITICAL tagged entries
- Glitch animation triggered on critical findings
- Blinking cursor on prompt line

**Usage:** `open assets/tui.html` (macOS) / `xdg-open assets/tui.html` (Linux). The browser blocks `file://` URLs for local assets — serve with `python3 -m http.server` in the skill directory if needed.

**Design note:** When building terminal-style splash screens, prefer Linear's dark-mode system over retro-CRT aesthetics. See `references/tui-design-notes.md` for the full design rationale and what to avoid.

## Relationship to Other Skills

- **`diagnose`** — if a finding from this skill reproduces in production, use `diagnose` to root-cause
- **`tdd`** — recommended tests in the report become TDD test cases for the next session
- **`improve-codebase-architecture`** — if systemic patterns emerge (no rate limiting anywhere, all auth in one middleware), hand off architectural recommendations
- **`request-refactor-plan`** — if findings require significant code changes, this skill produces the plan
- **`p5js`** — if the findings warrant an interactive visualization (e.g., resilience radar chart, exploration of attack paths), delegate to p5js

## When to Use This Skill

- Pre-launch review of a new project
- After major refactor, before merge
- When onboarding to a new codebase and wanting to understand failure modes
- After a production incident ("how could this have happened? what else?")
- Quarterly resilience review of a live system
- User says "show me the TUI" or "worst day ever" — launch `assets/tui.html` in browser as a visual splash before running the scan
