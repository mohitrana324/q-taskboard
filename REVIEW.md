# TaskBoard Code Review

Prioritized by business impact against the assignment requirements. The review is based on the current application code; line references point to the checked-in files.

| # | File and lines | Category | Severity | Issue and recommended fix |
|---|---|---|---|---|
| 1 | `backend/projects/views.py:165-184` | Security | Critical | `TaskDetailView.patch` loads a task by ID and updates it without checking that the requester belongs to its project or has an editing role. Any authenticated user, including a viewer or a member of another project, can modify any task whose UUID they obtain. Check project membership and require the `admin` or `member` role before applying changes, as the delete path already does. |
| 2 | `backend/projects/views.py:112-120` | Security | High | The task-search query interpolates the user-controlled `q` parameter directly into SQL. A search expression can change the query predicate; the read-only reproduction below made a search for a non-matching value return every task in the project. Replace the raw SQL with ORM filters (or parameterized SQL) and preserve the membership check. |
| 3 | `backend/projects/views.py:234-243` | Architecture | High | The export endpoint checks membership and role, but it does not export anything: it returns `exported: 0` and serialized tasks without making an Airtable API call. This leaves the assignment’s mandatory Part 3c unusable. Implement the real Airtable integration, repeat-run handling, transient-error retries, and per-record failure handling. |
| 4 | `backend/projects/views.py:156, 180-181` | Data Integrity | Medium | Task creation and updates accept an `assigneeId` without confirming that the user is a member of the same project. A task can therefore be assigned to an unrelated account, and the task response includes that account’s user details. Validate that the assignee has a membership in the task’s project (or reject the assignment) before saving. |

## Bug reproduction: injectable task search

This read-only request was run against the local backend using the seeded Meera account. The search term does not match the task titles, but the injected condition returns all seven tasks in the project.

```bash
TOKEN=$(curl -sS -X POST http://localhost:8000/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"meera@taskboard.dev","password":"password123"}' \
  | python -c "import json,sys; print(json.load(sys.stdin)['token'])")

PROJECT_ID=$(curl -sS http://localhost:8000/api/projects \
  -H "Authorization: Bearer $TOKEN" \
  | python -c "import json,sys; print(next(p['id'] for p in json.load(sys.stdin)['projects'] if p['name']=='Q3 Launch'))")

curl -sS -G "http://localhost:8000/api/projects/$PROJECT_ID/tasks" \
  --data-urlencode "q=x%' OR 1=1) --" \
  -H "Authorization: Bearer $TOKEN"
```

Actual response summary from the local API: **HTTP 200**, with all seven `Q3 Launch` tasks:

```text
Finalize launch date with marketing
Draft press release
Record demo video
Set up analytics dashboards
Prepare customer email blast
Update pricing page copy
QA the new signup flow end-to-end
```

The response body also includes each task’s selected fields (including description, status, and IDs). The request changed no data.
