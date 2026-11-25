# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

Record your plan here before editing code.

- Presentation tier decision: Keep API URL construction in the shared frontend API client and keep route views focused on rendering. Every capability view will expose accessible loading, success, empty, and error states.
- Logic tier decision: Keep request validation, business rules, status codes, and response shaping in the Express API. Reject malformed or missing resources with deliberate 4xx responses; do not add authentication.
- Data tier decision: Use Mongoose schemas to enforce required fields and relationships, with defaults for collection fields. Keep seeding idempotent and preserve records that are not exercise fixtures.
- Activity tracking approach: Validate activity input and both referenced records before creating an activity. Compute points on the server and set the occurrence date when creating it; return stored activities from the list endpoint.
- Team management approach: List and create teams while retaining their name, mascot, and member references. Return `400` when a required team name is missing.
- Leaderboard approach: Sum activity points per team, sort by total points descending, and return each team name, total, and 1-based rank.
- Workout suggestions approach: Find the user by email, use their fitness goal and recent activity to select matching workouts, and include a human-readable reason. Return a deliberate 4xx when no user matches.

## Test plan

Baseline recorded on 2026-11-25. The installed dependencies were already available, so no dependency restore was needed.

- Backend baseline: `npm test --prefix octofit-tracker/backend` — 8 failed, 1 passed. The seed test passes; four data-contract failures indicate missing schema validation, and four API-contract failures indicate incomplete endpoint behavior (invalid POSTs return `501`, leaderboard is empty, and suggestions omit `reason`).
- Frontend baseline: `npm test --prefix octofit-tracker/frontend` — 15 failed. The views do not yet satisfy API rendering, per-user suggestion requests, or accessible loading, empty, and error states.
- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — use after model and seed changes; failures indicate data-contract or seed regressions.
- API command: `npm run test:api --prefix octofit-tracker/backend` — use after route/service changes; failures indicate request validation, status/response, or business-rule regressions.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — use after UI changes; failures indicate API-client integration, rendering, or state/accessibility regressions.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — not run in the baseline; failures indicate backend TypeScript compilation errors.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — not run in the baseline; failures indicate frontend TypeScript or Vite build errors.
