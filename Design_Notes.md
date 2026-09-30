# Design Decisions: PATCH /tasks/:id/assign

1. **Sanitization:** `assignee` strings are trimmed of leading/trailing whitespace to prevent duplicate identities (e.g., `"Alice"` vs `" Alice "`).
2. **Validation:** Empty strings, whitespace-only strings, and non-string payloads return `400 Bad Request`. A 100-character ceiling prevents buffer/memory bloat.
3. **Reassignment:** Reassignment is allowed without conflict (no 409). In standard task management systems (e.g. Jira, GitHub Issues), tasks are routinely handed off between team members.
4. **Idempotence & Method Choice:** `PATCH` is utilized because it selectively mutates only the `assignee` field while leaving other task metadata untouched.
5. **Missing Resource:** Returns `404 Not Found` if the provided `:id` does not match an existing task record.