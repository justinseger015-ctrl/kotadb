# Bug: Review Panel Blocking Issues for v0.1.2 Release (Issue #506)

## User Story / Problem Statement

PR #505 (Release v0.1.2) received a comprehensive Review Panel evaluation from 7 domain experts. This issue consolidates the **4 blocking issues** that must be resolved before the release can merge to main.

**Review Panel Status:** CHANGES_REQUESTED (3 APPROVE, 3 CHANGES_REQUESTED, 1 APPROVE)

## Expert Analysis Summary

### Architecture Perspective

**Issue #1 (Projects RLS):** Architectural inconsistency - Projects API violates established RLS context pattern used in `job-tracker.ts`. Data flow violation: expected middleware sets context before RLS queries, actual flow skips context entirely.

**Recommendation:** Centralize context management with a wrapper function or apply pattern consistently to all 6 project endpoints.

### Testing Strategy

**Antimocking Compliance Required:**
- All tests must use real Supabase Local
- RLS enforcement tests needed for Projects API (6 endpoints)
- Job status cross-user access test needed
- MCP schema validation tests for localPath rejection

**Test Pattern:**
```typescript
// Create User A data
// Attempt User B access
// Expect 404/403
```

### Security Considerations

**Issue #1 (CRITICAL):** Without `setUserContext()`, RLS policies cannot enforce `current_setting('app.user_id', true)::uuid`. If service client is used, RLS bypassed entirely.

**Issue #2 (HIGH):** `localPath` parameter could enable path traversal attacks, sensitive file exposure, resource exhaustion.

**Issue #3 (MEDIUM):** Any authenticated user can query ANY job status if they know the UUID - violates principle of least privilege.

### Integration Requirements

**Issue #2 (CRITICAL Contract Violation):** MCP schema still contains `localPath` despite commit daa8ee2 claiming removal. Schema-implementation mismatch breaks MCP client expectations.

**Issue #3:** Tool description claims "RLS enforced" but implementation doesn't enforce at query level.

### UX & Accessibility

**Issue #4:** Developers exploring expert system find incomplete documentation:
- 7 experts exist but README.md documents only 4
- CLAUDE.md has no reference to expert system
- Reduced trust in documentation accuracy

### Hook & Automation Considerations

Pre-commit hooks should catch documentation drift. Consider adding validation script to verify expert documentation completeness.

### Claude Configuration

**Issue #4:** CLAUDE.md needs Experts section added to Command Navigation tables. All 7 experts and orchestrator commands should be documented for discoverability.

## Synthesized Recommendations

### Priority Actions

1. **CRITICAL - Block Release:** Remove `localPath` from MCP schema (Issue #2) - 30 min
2. **CRITICAL - Block Release:** Add RLS context to Projects API (Issue #1) - 2 hours
3. **HIGH - Recommended:** Fix get_index_job_status RLS (Issue #3) - 1 hour
4. **MEDIUM - Post-Release:** Complete expert documentation (Issue #4) - 1 hour

### Risk Assessment

| Issue | Risk Level | Blocker |
|-------|------------|---------|
| #1 Projects RLS | CRITICAL | YES |
| #2 localPath | CRITICAL | YES |
| #3 Job Status RLS | MEDIUM | Recommended |
| #4 Documentation | LOW | No |

## Implementation Plan

### Phase 1: Remove localPath from MCP Schema (Issue #2)

**File:** `app/src/mcp/tools.ts`

- [ ] Remove `localPath` from `INDEX_REPOSITORY_TOOL.inputSchema.properties` (lines 96-99)
- [ ] Remove `localPath` validation from type guard `isIndexParams` (around line 512)
- [ ] Remove `localPath` validation from `executeIndexRepository` (around lines 676-689)
- [ ] Add explicit rejection with clear error message for backward compatibility
- [ ] Update `IndexRequest` type if `localPath` field exists

### Phase 2: Add RLS Context to Projects API (Issue #1)

**File:** `app/src/api/routes.ts`

For each of the 6 project endpoints (lines 1088-1258):

- [ ] POST /api/projects - Add `setUserContext`/`clearUserContext` wrapper
- [ ] GET /api/projects - Add `setUserContext`/`clearUserContext` wrapper
- [ ] GET /api/projects/:id - Add `setUserContext`/`clearUserContext` wrapper
- [ ] PATCH /api/projects/:id - Add `setUserContext`/`clearUserContext` wrapper
- [ ] DELETE /api/projects/:id - Add `setUserContext`/`clearUserContext` wrapper
- [ ] POST /api/projects/:id/repositories/:repoId - Add wrapper
- [ ] DELETE /api/projects/:id/repositories/:repoId - Add wrapper

**Pattern to follow (from `job-tracker.ts`):**
```typescript
import { setUserContext, clearUserContext } from "@db/client";

// In each endpoint:
try {
    await setUserContext(supabase, context.userId);
    // ... existing query code ...
} finally {
    await clearUserContext(supabase);
}
```

### Phase 3: Fix get_index_job_status RLS (Issue #3)

**File:** `app/src/mcp/tools.ts` (lines 1341-1387)

- [ ] Remove underscore prefix from `_userId` parameter
- [ ] Add `setUserContext(supabase, userId)` before query
- [ ] Add `clearUserContext(supabase)` in finally block

**File:** `app/src/api/queries.ts` (lines 829-846)

- [ ] Verify RLS policy on `index_jobs` table enforces user access
- [ ] Update function signature to accept `userId` if needed for defense-in-depth

### Phase 4: Complete Expert Documentation (Issue #4)

**File:** `.claude/commands/README.md`

- [ ] Update Expert Types table (lines 215-222) to include all 7 experts:
  - Architecture, Testing, Security, Integration (existing)
  - UX Expert (add)
  - CC Hook Expert (add)
  - Claude Config Expert (add)

**File:** `CLAUDE.md`

- [ ] Add new "Experts" section to Command Navigation
- [ ] List all expert commands with purposes
- [ ] Include orchestrator commands (planning_council, review_panel, orchestrator)

## Validation Requirements

- [ ] Core gates: `cd app && bun run lint`, `cd app && bunx tsc --noEmit`
- [ ] Tests: `cd app && bun test`
- [ ] Build: `cd app && bun run build`
- [ ] Integration: `cd app && bun test --filter "projects|mcp|job.*status"`

## Acceptance Criteria

- [ ] All Projects API endpoints call `setUserContext()` before database operations
- [ ] `localPath` removed from INDEX_REPOSITORY_TOOL schema with rejection logic
- [ ] `getIndexJobStatus` enforces user RLS via userId parameter
- [ ] CLAUDE.md includes Experts section with all 7 domain experts
- [ ] README.md expert table updated to show all 7 experts
- [ ] All existing tests pass
- [ ] New tests added for RLS enforcement on job status

## Notes

### Files Modified

| File | Changes |
|------|---------|
| `app/src/mcp/tools.ts` | Remove localPath (lines 96-99, ~512, ~676-689), fix job status RLS (lines 1341-1387) |
| `app/src/api/routes.ts` | Add RLS context to 7 project endpoints (lines 1088-1258) |
| `app/src/api/queries.ts` | Verify/update getIndexJobStatus (lines 829-846) |
| `.claude/commands/README.md` | Add 3 missing experts to table |
| `CLAUDE.md` | Add Experts section to navigation |

### Related Issues

- **Blocks:** PR #505 (Release v0.1.2)
- **Related:** Issue #412 (localPath removal)
- **Related:** Issue #413 (get_index_job_status feature)

### Expert Attribution

| Finding | Domain Expert |
|---------|---------------|
| RLS Context Missing | Security Expert |
| localPath Schema | Integration Expert |
| Job Status RLS | Integration + Security |
| Expert Docs | Claude Config Expert |
