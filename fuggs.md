# Bug Report

This document details the bugs identified during testing.

## How I found these
I used an AI tool to help draft the initial test suite, then ran the tests and read each failure, got help, and ultimately worked out what was wrong in the code. 
For each bug below I checked the source myself and confirmed the cause before fixing it.

---
### Bug 1: Pagination Skips the First Page of Results (Off-by-One)
- **File & Function:** `src/services/taskService.js` -> `getPaginated`
- **How I Found It:** Discovered via unit test `getPaginated › should return the first page starting at index 0` and route integration test `GET /tasks › should paginate tasks with page and limit query params`.
- **Expected Behavior:** Requesting page 1 with limit 10 (`/tasks?page=1&limit=10`) should calculate offset `0` and return records 1 through 10.
- **Actual Behavior:** Computes offset as `page * limit` (`1 * 10 = 10`), skipping the first 10 items.
- **Root Cause:** Standard human-facing pagination is 1-indexed, but zero-based indexing requires subtracting 1 before multiplying by the limit.
- **Fix (Applied):** Changed `const offset = page * limit;` to `const offset = (page - 1) * limit;`.

---

### Bug 2: Status Filter Uses Substring Matching
- **File & Function:** `src/services/taskService.js` -> `getByStatus`
- **How I Found It:** Discovered via unit test `getByStatus › should not return tasks on partial substring matches`.
- **Expected Behavior:** Querying `GET /tasks?status=do` should return no items because `do` is not a valid task status.
- **Actual Behavior:** Returns both `todo` and `done` tasks because `String.prototype.includes('do')` evaluates to true for both strings.
- **Root Cause:** Using `.includes()` rather than exact string equality against the status enum.
- **Fix (Applied):** Changed `t.status.includes(status)` to `t.status === status`.

---

### Bug 3: Completing a Task Silently Overwrites Priority (Data Loss)
- **File & Function:** `src/services/taskService.js` -> `completeTask`
- **How I Found It:** Discovered via unit test `completeTask › should preserve original priority when marking a task complete`.
- **Expected Behavior:** Marking a task as complete (`PATCH /tasks/:id/complete`) should update `status` to `done` and set `completedAt`, preserving existing task properties.
- **Actual Behavior:** The task's `priority` is quietly forced back to `'medium'`, destroying any previously configured priority (`'high'` or `'low'`).
- **Root Cause:** An explicit `priority: 'medium'` property is hardcoded in the update spread.
- **Fix (Applied):** Removed the `priority: 'medium'` line from `completeTask`.

---

### Bug 4: Status and Pagination Parameters Cannot Be Combined (Documented / Unfixed)
- **File & Function:** `src/routes/tasks.js` -> `GET /tasks`
- **How I Found It:** Discovered while reviewing the route logic and testing queries with multiple parameters.
- **Expected Behavior:** Calling `/tasks?status=todo&page=1&limit=5` should return the first 5 tasks that have the status `todo`.
- **Actual Behavior:** The handler contains `if (status) return res.json(...)` which executes an early return, completely ignoring `page` and `limit`.
- **Suggested Fix:** Rather than mutually exclusive branching, filter the list by status first (if provided), and then slice the resulting array using the pagination helper.