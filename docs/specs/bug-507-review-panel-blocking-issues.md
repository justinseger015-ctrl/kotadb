# Bug: Review Panel Blocking Issues for v0.1.2 (Issue #507)

## User Story / Problem Statement

The Review Panel analysis of PR #505 (Release v0.1.2) identified **4 blocking issues** that must be resolved before merge. These issues affect MCP tool functionality, API correctness, and naming consistency:

1. **search_code MCP Tool Missing project Parameter** - Users cannot filter searches by project via MCP
2. **Project-Scoped Search Filter Not Applied** - Security/data isolation bug where project filter is fetched but never applied
3. **MCP Server Version Mismatch** - Reports `0.1.0` instead of `0.1.2`
4. **Parameter Naming Inconsistency (runId -> run_id)** - Breaks snake_case convention used by all other MCP tools

## Expert Analysis Summary

### Architecture Perspective
- **Issue 1**: Pattern for `project` parameter exists in `SEARCH_DEPENDENCIES_TOOL` (line 150) - follow same pattern
- **Issue 2**: Filter pattern exists at line 360 (`options.repositoryId`) - add `else if` clause for `repositoryIds`
- **Issue 3**: Version is hardcoded at `server.ts:63` - update to `0.1.2`
- **Issue 4**: All other tools use snake_case (`repository_id`, `file_path`, etc.) - rename `runId` to `run_id`

### Testing Strategy
- Existing test at `app/tests/mcp/get-index-job-status.test.ts` uses `runId` - must update to `run_id`
- No new tests required for version update
- Project-scoped search test coverage exists at `app/tests/integration/projects.test.ts`
- All tests must continue passing after changes

### Security Considerations
- **CRITICAL**: Issue 2 is a data isolation bug - project-scoped search currently returns ALL user files instead of just project files
- Fix ensures defense-in-depth layer is properly applied
- RLS is already enforced via `setUserContext()` - this adds application-level filtering

### Integration Requirements
- MCP tool schema changes are additive (Issue 1) or internal (Issues 2, 3, 4)
- No breaking API changes for external consumers
- `executeSearchCode()` must resolve projectId using existing `resolveProjectId()` helper

### UX & Accessibility
- Consistent naming convention (`run_id` vs `runId`) improves developer experience
- Correct version reporting helps clients with compatibility detection
- Project-scoped search working as expected improves data discovery UX

### Hook & Automation Considerations
- Pre-commit hooks will run `tsc`, `biome lint` - no issues expected
- CI validation will run full test suite
- Version update should be automated in future via CI/CD

### Claude Configuration
- No CLAUDE.md changes required
- No settings.json changes required

## Synthesized Recommendations

### Priority Actions
1. **Fix Issue 2 FIRST** (Critical Security Bug) - Add `else if (repositoryIds?.length > 0)` clause
2. **Fix Issue 1** - Add `project` parameter to `SEARCH_CODE_TOOL` and wire it through `executeSearchCode()`
3. **Fix Issue 4** - Rename `runId` to `run_id` in tool schema, executor, response, and tests
4. **Fix Issue 3** - Update version string to `0.1.2`

### Risk Assessment
- **High Risk Areas**: Issue 2 query modification - ensure filter logic is correct
- **Mitigation Strategies**: Run full test suite, verify with manual testing

## Implementation Plan

### Phase 1: Fix Project-Scoped Search Filter (Issue 2) - CRITICAL
**File**: `app/src/api/queries.ts:360-362`

Current code:
```typescript
if (options.repositoryId) {
  query = query.eq("repository_id", options.repositoryId);
}
```

Fix:
```typescript
if (options.repositoryId) {
  query = query.eq("repository_id", options.repositoryId);
} else if (repositoryIds && repositoryIds.length > 0) {
  query = query.in("repository_id", repositoryIds);
}
```

- [ ] Add `else if` clause to apply project filter when `repositoryIds` is populated
- [ ] Verify existing tests pass

### Phase 2: Add project Parameter to search_code Tool (Issue 1)
**Files**: `app/src/mcp/tools.ts`

2.1. Update `SEARCH_CODE_TOOL` inputSchema (lines 55-77):
- [ ] Add `project` property to `inputSchema.properties`:
```typescript
project: {
  type: "string",
  description: "Optional: Filter results to a specific project (by name or UUID)",
},
```

2.2. Update `executeSearchCode()` (lines 591-640):
- [ ] Add validation for `project` parameter (optional string)
- [ ] Resolve project identifier to UUID using `resolveProjectId()` helper
- [ ] Pass `projectId` to `searchFiles()` call in options

### Phase 3: Fix runId Naming Inconsistency (Issue 4)
**Files**: `app/src/mcp/tools.ts`, `app/tests/mcp/get-index-job-status.test.ts`

3.1. Update `GET_INDEX_JOB_STATUS_TOOL` (lines 446-468):
- [ ] Rename `runId` to `run_id` in inputSchema.properties
- [ ] Update description to reference `run_id`
- [ ] Update `required` array to use `run_id`

3.2. Update `executeGetIndexJobStatus()` (lines 1337-1386):
- [ ] Change parameter validation from `p.runId` to `p.run_id`
- [ ] Update error messages from `runId` to `run_id`
- [ ] Change response field from `runId` to `run_id`

3.3. Update test file `app/tests/mcp/get-index-job-status.test.ts`:
- [ ] Line 51: Update test to use `run_id` instead of empty object
- [ ] Line 77: Change `runId: 12345` to `run_id: 12345`
- [ ] Line 103: Change `runId: "not-a-valid-uuid"` to `run_id: "not-a-valid-uuid"`
- [ ] Line 130: Change `runId: nonExistentId` to `run_id: nonExistentId`
- [ ] Lines 161, 171, 181, 221, 223: Update `extractToolResult` expectations from `runId` to `run_id`

### Phase 4: Fix MCP Server Version (Issue 3)
**File**: `app/src/mcp/server.ts:63`

- [ ] Change `version: "0.1.0"` to `version: "0.1.2"`

## Validation Requirements

- [ ] Core gates: `cd app && bun run lint`, `cd app && bunx tsc --noEmit`
- [ ] Tests: `cd app && bun test`
- [ ] Build: `cd app && bun run build`
- [ ] Specific test validation: `cd app && bun test tests/mcp/get-index-job-status.test.ts`
- [ ] Project search test: `cd app && bun test tests/integration/projects.test.ts`

## Acceptance Criteria (from Issue #507)

- [ ] `search_code` MCP tool accepts optional `project` parameter
- [ ] Project-scoped search returns only files from project's repositories
- [ ] MCP server version reports `0.1.2`
- [ ] `get_index_job_status` uses `run_id` (snake_case) consistently
- [ ] All existing tests pass
- [ ] New test for project-scoped search via MCP (optional - covered by existing integration test)

## Notes

### Files to Modify
| File | Lines | Change |
|------|-------|--------|
| `app/src/api/queries.ts` | 360-362 | Add `else if` for `repositoryIds` filter |
| `app/src/mcp/tools.ts` | 55-77 | Add `project` param to `SEARCH_CODE_TOOL` |
| `app/src/mcp/tools.ts` | 591-640 | Wire `project` param in `executeSearchCode()` |
| `app/src/mcp/tools.ts` | 446-468 | Rename `runId` to `run_id` in `GET_INDEX_JOB_STATUS_TOOL` |
| `app/src/mcp/tools.ts` | 1337-1386 | Update `executeGetIndexJobStatus()` for `run_id` |
| `app/src/mcp/server.ts` | 63 | Update version to `0.1.2` |
| `app/tests/mcp/get-index-job-status.test.ts` | Multiple | Update all `runId` references to `run_id` |

### References
- Issue #507: https://github.com/jayminwest/kota-db-ts/issues/507
- PR #505: Release v0.1.2
- Issue #506: Previous Review Panel blocking issues (resolved)
- Issue #469: MCP project CRUD tools
- Issue #413: get_index_job_status implementation

### Estimated Impact
- **Low Risk**: Changes are localized to 4 files
- **No Breaking Changes**: `search_code` addition is backward compatible, `run_id` rename affects MCP only
- **No Database Migrations**: Pure code changes
- **Rollback Strategy**: Single git revert sufficient
