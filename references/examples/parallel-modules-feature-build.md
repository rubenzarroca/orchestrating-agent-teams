# Example: Parallel Module Development — "Build Notification Preferences" (Throughput Paradigm)

A 3-teammate feature build where each agent owns a layer of the stack. No cross-
communication needed — each works in isolation against a shared contract. The Lead
coordinates handoff and integration. This is Throughput at its simplest: divide the
work, go faster, assemble the result.

---

## Context

Project: B2B SaaS platform. Adding a notification preferences feature: users can
choose which notifications they receive (email, in-app, SMS) and set quiet hours.
Stack: Next.js frontend, Express API, Jest tests. The feature touches three
independent directories with a clear API contract between them.

## Step 1: Planning (Single Agent, Plan Mode)

Before spawning, the developer defines the contract and splits the work:

```
I need a notification preferences feature. Users should be able to toggle
notification channels (email, in-app, SMS) per event type, and set quiet hours.
Use plan mode. Define the API contract and split into parallel tracks.
```

**Lead output (plan):**

API contract (shared by all teammates):
- `GET /api/preferences` → returns `{ channels: { email: bool, inApp: bool, sms: bool }, quietHours: { enabled: bool, start: string, end: string }, eventTypes: [...] }`
- `PUT /api/preferences` → accepts same shape, returns updated preferences
- `GET /api/preferences/events` → returns available event types

Tracks:
- Track 1 — API: Express routes + Postgres queries in `src/api/preferences/`
- Track 2 — Frontend: React components in `src/frontend/preferences/`
- Track 3 — Tests: Unit + integration tests in `tests/preferences/`

All three tracks can run in parallel. No file overlap. Frontend mocks the API
using the contract until integration.

## Step 2: Team Spawn

```
Build a notification preferences feature. Users toggle notification channels
(email, in-app, SMS) per event type and set quiet hours.

API contract:
- GET /api/preferences → { channels: { email, inApp, sms }, quietHours: { enabled, start, end }, eventTypes: [...] }
- PUT /api/preferences → accepts same shape, returns updated
- GET /api/preferences/events → returns available event types

Stack: Express API, Next.js frontend, Jest tests. Postgres for persistence.

Spawn 3 teammates:

1. API teammate: Implement the three endpoints in src/api/preferences/.
   Create the Postgres migration for the preferences table. Use the existing
   db connection pool in src/db/pool.ts. Validate input (quiet hours must be
   valid HH:MM, channels must be boolean). Return proper HTTP status codes.
   You own src/api/preferences/ and src/db/migrations/ exclusively.
   Do not touch any other directory.
   Done when: all three endpoints work, migration is ready, input validation
   is in place.

2. Frontend teammate: Build the preferences page in src/frontend/preferences/.
   Components needed: PreferencesPage (container), ChannelToggles (per event type),
   QuietHoursForm (time picker with enable/disable). Use the existing design
   system in src/frontend/components/ui/. Mock API calls using the contract above
   until integration. Follow existing patterns in src/frontend/settings/ for layout.
   You own src/frontend/preferences/ exclusively.
   Do not touch any other directory.
   Done when: all components render correctly with mock data, form validation
   works, loading and error states are handled.

3. Test teammate: Write tests in tests/preferences/. Cover:
   - API unit tests: valid CRUD, invalid input rejection, quiet hours validation
   - Frontend unit tests: toggle behavior, form submission, error states
   - Integration test: full flow from UI toggle → API call → DB update → UI confirmation
   You own tests/preferences/ exclusively.
   Do not touch any other directory.
   Done when: all critical paths have coverage, edge cases documented.

Each teammate owns their directory exclusively. No cross-directory edits.
Use Sonnet for all teammates.
Activate delegate mode. Do not write any code yourself.
Wait for all teammates to complete their tasks before proceeding.
```

## Step 3: Parallel Execution

Each teammate works in isolation against the shared contract:

**API teammate** (working ~8 min):
- Creates Postgres migration: `preferences` table with user_id, channel_settings (JSONB),
  quiet_hours (JSONB), event_overrides (JSONB)
- Implements GET /api/preferences with user auth middleware
- Implements PUT /api/preferences with Joi validation
- Implements GET /api/preferences/events returning hardcoded event catalog
- Tests manually with curl
- Marks task complete

**Frontend teammate** (working ~10 min):
- Scaffolds PreferencesPage using existing SettingsPage layout as reference
- Builds ChannelToggles component: grid of event types × channels with toggle switches
- Builds QuietHoursForm: enable checkbox + two time pickers (start/end)
- Mocks API calls using the contract (returns hardcoded data matching contract shape)
- Adds loading spinner, error toast, success confirmation
- Marks task complete

**Test teammate** (working ~7 min):
- Writes API tests: happy path CRUD, invalid time format rejection, missing fields,
  unauthorized access
- Writes frontend tests: toggle state changes, form submission calls correct endpoint,
  error state renders message, quiet hours validation (end > start)
- Writes integration test skeleton (requires both API and frontend, will need wiring)
- Marks task complete

## Step 4: Lead Synthesis

After all three teammates report, the Lead verifies and integrates:

```markdown
## Notification Preferences — Build Summary

### Completed
- API: 3 endpoints implemented in src/api/preferences/, migration ready.
  Input validation covers channel booleans and quiet hours format.
- Frontend: 3 components in src/frontend/preferences/. Mock API in place.
  Loading, error, and success states handled.
- Tests: 12 unit tests (API), 8 unit tests (frontend), 1 integration skeleton.

### Integration Steps (remaining)
1. Replace frontend API mocks with real endpoint calls
2. Run migration on dev database
3. Wire integration test to use test database
4. Verify end-to-end flow: UI → API → DB → UI confirmation

### Issues Found
None. All teammates delivered within scope and contract.
```

## Why This Worked

- **Shared contract up front**: The API shape was defined before spawning. All three
  teammates built against the same interface, so integration is mechanical, not creative.
- **Strict directory ownership**: API owns src/api/preferences/, Frontend owns
  src/frontend/preferences/, Tests own tests/preferences/. Zero conflict risk.
- **No cross-communication needed**: This is pure Throughput — each teammate works in
  isolation. The mailbox was never used. The task list tracked completion, nothing more.
- **Parallel time savings**: Sequentially this would take ~25 minutes. In parallel, the
  wall-clock time was ~10 minutes (the longest single track). That's the Throughput payoff.
- **Clear integration path**: The Lead's synthesis isn't just "concatenate reports" — it
  identifies the remaining integration steps that require sequential work after the parallel
  phase completes.
