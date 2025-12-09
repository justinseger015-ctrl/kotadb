# Bug: Review Panel Final Blocking Issues for v0.1.2 (Issue #508)

## User Story / Problem Statement

Follow-up to #506 and #507, capturing 5 additional critical fixes identified by the Review Panel for PR #505 (Release v0.1.2). These issues span security (RLS context), UX (web documentation), and documentation (CLAUDE.md) domains.

**Review Panel Status:** CHANGES_REQUESTED
**Expert Consensus:** Architecture, Security, UX, and Claude Config experts all identified blocking issues.

## Expert Analysis Summary

### Architecture Perspective

**Priority Actions:**
1. **CRITICAL**: Add `setUserContext()` to 7 MCP project tool handlers
2. **HIGH**: Verify RLS policy enforcement for project-related tables
3. **MEDIUM**: Validate architecture consistency across MCP/REST interfaces

**Risk Assessment:**
- Security Risk: HIGH - Missing RLS context in MCP handlers bypasses data isolation
- Consistency Risk: MEDIUM - Architectural inconsistency between MCP and REST handlers
- Regression Risk: LOW - Changes are additive (no breaking changes)

**Key Finding:** The `setUserContext()` import already exists at line 27 (`import { setUserContext } from "@db/client.js";`) but is only called in `executeGetIndexJobStatus()` at line 1380, creating inconsistency.

### Testing Strategy

**Required Tests:**
1. Unit tests verifying `setUserContext()` is called before database operations
2. Integration tests for RLS isolation between users via MCP interface
3. Regression tests ensuring existing REST API behavior unchanged

**Anti-mock Compliance:** All tests must use real Supabase Local connections per project testing philosophy.

### Security Considerations

**Current Vulnerability:**
- 7 project MCP handlers do NOT call `setUserContext()` before database operations
- Without RLS context, `current_setting('app.user_id')` returns NULL
- Defense-in-depth failure - only application-level filtering, not database-level enforcement

**Severity:** MEDIUM (not HIGH) because explicit `.eq("user_id", userId)` filters provide application-level protection, but RLS enforcement is compromised.

**After Fix:** Defense-in-depth restored - both application AND database enforce isolation.

### Integration Requirements

**MCP-REST Consistency:**
- REST API routes correctly call `setUserContext()` (confirmed in #506)
- MCP tool handlers must follow identical patterns

**Web UI Integration:**
- `ToolReference.tsx` hardcodes tool list instead of fetching from source
- 10 of 14 tools (71%) missing from web documentation

### UX & Accessibility

**User Impact:**
- Issue 2: Users following web docs will use removed `localPath` parameter and receive errors
- Issue 3: 10 of 14 tools (71%) undiscoverable via web UI
- Issue 5: Expert `_improve` commands hidden from users

### Hook & Automation Considerations

No pre-commit hook changes required. Future enhancement: ESLint rule to detect missing `setUserContext()` calls.

### Claude Configuration

**Issue 4 Investigation:** No "4 experts" text found in current CLAUDE.md - may have been fixed or line numbers incorrect.

**Issue 5 Confirmed:** Expert table documents 14 commands (`_plan` + `_review` for 7 experts) but 7 `_improve` commands exist and are NOT documented.

## Synthesized Recommendations

### Priority Actions

1. **CRITICAL (Block Release):** Add `setUserContext()` to 7 MCP project handlers
2. **HIGH (Should Fix Before Release):** Remove `localPath` from web UI, add 10 missing tools
3. **MEDIUM (Can Fix After Release):** Add 7 `_improve` commands to CLAUDE.md Expert table

### Risk Assessment

- **High Risk Areas:** MCP RLS context (security), web documentation accuracy (UX)
- **Mitigation Strategies:** Add RLS isolation tests, verify web UI against live MCP server

## Implementation Plan

### Phase 1: Security Fix (MCP setUserContext)

**File:** `app/src/mcp/tools.ts`

- [ ] Add `await setUserContext(supabase, userId);` to `executeCreateProject()` (line 1073, before `createProject()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeListProjects()` (line 1099, before `listProjects()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeGetProject()` (line 1140, before `resolveProjectId()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeUpdateProject()` (line 1193, before `resolveProjectId()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeDeleteProject()` (line 1233, before `resolveProjectId()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeAddRepositoryToProject()` (line 1275, before `resolveProjectId()` call)
- [ ] Add `await setUserContext(supabase, userId);` to `executeRemoveRepositoryFromProject()` (line 1338, before `resolveProjectId()` call)

### Phase 2: Web UI Fixes

**File:** `web/components/mcp/ToolReference.tsx`

- [ ] Remove `localPath (optional)` from line 12 params
- [ ] Add 10 missing tools to the `tools` array:
  - `analyze_change_impact`
  - `validate_implementation_spec`
  - `create_project`
  - `list_projects`
  - `get_project`
  - `update_project`
  - `delete_project`
  - `add_repository_to_project`
  - `remove_repository_from_project`
  - `get_index_job_status`

### Phase 3: Documentation Fixes

**File:** `CLAUDE.md`

- [ ] Investigate and fix expert count references (Issue 4 - may already be fixed)
- [ ] Add 7 `_improve` commands to Expert table after existing entries:
  - `/experts:architecture-expert:architecture_expert_improve`
  - `/experts:security-expert:security_expert_improve`
  - `/experts:testing-expert:testing_expert_improve`
  - `/experts:integration-expert:integration_expert_improve`
  - `/experts:ux-expert:ux_expert_improve`
  - `/experts:cc_hook_expert:cc_hook_expert_improve`
  - `/experts:claude-config:claude_config_improve`

## Validation Requirements

- [ ] Core gates: `cd app && bun run lint`, `cd app && bunx tsc --noEmit`
- [ ] Tests: `cd app && bun test`
- [ ] Build: `cd app && bun run build`
- [ ] Manual verification: All 14 MCP tools visible in web UI
- [ ] Manual verification: `localPath` not shown in web UI

## Files to Modify

| File | Changes |
|------|---------|
| `app/src/mcp/tools.ts` | Add `setUserContext()` to 7 handlers |
| `web/components/mcp/ToolReference.tsx` | Fix params, add 10 tools |
| `CLAUDE.md` | Add 7 `_improve` commands |

## Notes

**Issue Relationships:**
- **Blocks:** PR #505 (Release v0.1.2)
- **Follows:** #506 (RLS context in routes.ts, localPath schema removal)
- **Follows:** #507 (search_code project filter, version, naming)

**Expert Attribution:**
| Finding | Domain Expert(s) |
|---------|--------------------|
| MCP setUserContext missing | Architecture, Security |
| Web UI localPath outdated | UX |
| Web UI missing tools | UX |
| Expert count incorrect | Claude Config |
| _improve commands missing | Claude Config |
