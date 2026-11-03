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

- Presentation tier decision: Keep React components responsible for displaying API data and accessible loading, empty, and error states. Use the shared API client with relative `/api` paths so Vite proxies requests to the Express API; keep business rules and persistence out of the frontend.
- Logic tier decision: Keep Express routes thin and delegate validation and capability behavior to backend services. Return deliberate 4xx responses for invalid input, malformed or missing references, and unknown users; do not expose cast errors as 500 responses.
- Data tier decision: Use Mongoose models for users, teams, activities, and workouts, with required fields and defaults that match the API and data contracts. Keep seed data idempotent and preserve unrelated application records. Keep MongoDB, Express, and Vite on ports 27017, 8000, and 5173 respectively, and do not add authentication in this exercise.
- Activity tracking approach: Validate the request and confirm `userId` and `teamId` are well-formed IDs for existing records before creating an activity. Store the corresponding user and team references, submitted type and minutes, computed points, and occurrence time. Return a deliberate 4xx for invalid input or missing references, without creating an activity.
- Team management approach: Reject a team without a name with `400`, and preserve the submitted name and mascot in create and list responses. Support the members field as team member references.
- Leaderboard approach: Aggregate activity points by team, order teams by total points from highest to lowest, and include each team's name, total, and 1-based rank.
- Workout suggestions approach: Find the requested user by email and use their goal or recent activity to choose a matching workout, with the first suggestion's focus matching that goal or activity and a reason explaining the recommendation. Return a deliberate 4xx response if no user matches; do not add authentication.

## Test plan

Record the commands you run and what each failure means.

| Command | What a failure means |
| --- | --- |
| `npm run test:data --prefix octofit-tracker/backend` | Model validation, defaults, or seed behavior violates the data contract. |
| `npm run test:api --prefix octofit-tracker/backend` | An API route, request validation, status code, or response violates its contract. |
| `npm test --prefix octofit-tracker/frontend` | A frontend component or user-facing state violates its contract. |
| `npm run build --prefix octofit-tracker/backend` | Backend TypeScript has a compile-time error. |
| `npm run build --prefix octofit-tracker/frontend` | The frontend has a TypeScript or Vite build error. |

### Execution record

- `npm run test:data --prefix octofit-tracker/backend`: initially failed because required model fields were not validated; the seed idempotency test passed. After adding the schema validations, all 5 tests passed.
- `npm run build --prefix octofit-tracker/backend`: passed after the model changes.
