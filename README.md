# TypeArena Pro

Dark, esports inspired typing competition dashboard with an Express/Mongoose API and Socket.IO event rooms.

## Architecture

- **Web:** React, TypeScript and Vite. The client displays event state and typing metrics; official time and scoring belong on the server.
- **API:** Express with Zod validation and rate limits. Host-only event creation and lifecycle operations use signed JWTs with `admin`, `host` or `superadmin` roles.
- **Realtime:** Socket.IO rooms are scoped to event IDs. Lifecycle updates emit after the event is saved.
- **Persistence:** MongoDB stores events and up to four rounds per event. Result documents have a unique event/round/participant index to prevent duplicate results.
- **Accounts:** Passwords are bcrypt hashed. Student self-registration cannot choose a privileged role. An administrator can be bootstrapped once with a separate one-time key.
- **Uploads:** Hosts can request a short-lived Cloudinary upload signature. The API secret remains on the server.

## State flow

Events use `registration → live → completed`. Rounds use `scheduled → live ↔ paused → completed`. Authorized server actions set the official deadline and pause accounting. Invalid transitions return HTTP 409.

## Local setup

1. Install Node.js 20 or newer.
2. Copy `.env.example` to `.env` and set `MONGODB_URI` and a random `JWT_SECRET`.
3. Run `npm install`, then `npm run dev`.
4. Vite serves the UI at `http://localhost:5173`; API health is at `http://localhost:4000/api/health`.

The host API requires `Authorization: Bearer <JWT>` where the JWT role is `admin`, `host`, or `superadmin`. No production administrator or password is seeded.

## API

- `GET /api/health` — service and MongoDB connectivity.
- `POST /api/auth/register` and `/api/auth/login` — student account creation and login.
- `POST /api/auth/bootstrap` — one-time initial superadmin setup using `INITIAL_ADMIN_KEY`.
- `GET /api/events?q=...` — public registration/live event discovery.
- `POST /api/events/:id/register` — authenticated student registration.
- `POST /api/events` — host-authenticated event creation; `rounds` is optional and limited to four.
- `POST /api/events/:id/action` — host-authenticated `open`, `start`, `pause`, `resume`, or `end` transition.
- `POST /api/events/:id/submit` — authenticated registered participant score submission, checked against the persisted round passage and server deadline.
- `GET /api/events/:id/leaderboard` — persisted official results for a round.
- `POST /api/uploads/signature` — host-authenticated Cloudinary signature generation.
- Socket.IO: emit `event:join` with an event ObjectId; receive `event:state` and `leaderboard:update` updates.

The UI now has student signup/login, one-time administrator setup, role-aware navigation, event create/register flows, host round controls, timed typing submissions, result leaderboards, administrator player/role management, and database-backed analytics. Admins can assign `host`, `admin`, or `student` roles. Redis scale-out, full audit history, qualification/advancement workflows, and deployment-specific hardening remain before a large tournament. Remove the initial-admin setup key after bootstrap. See `LOCAL_AND_RENDER.md` for local testing and deployment.

Never commit `.env`, expose server credentials in frontend variables, or paste active credentials into source control. Rotate credentials shared outside their secret manager.
