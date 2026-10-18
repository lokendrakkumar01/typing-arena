# Local run, smoke checks, and Render deployment

## Run locally

Requirements: Node.js 20+, npm, and a MongoDB Atlas database. From the project folder:

```powershell
Copy-Item .env.example .env
```

Edit `.env` with your Atlas connection string, a long random `JWT_SECRET`, a separate random `INITIAL_ADMIN_KEY`, and Cloudinary values. Do not commit `.env`. In Atlas, allow the machine running this app to connect and ensure the database user has the required database permissions.

Then run:

```powershell
npm install
npm run dev
```

Open `http://localhost:5173`. The API listens at `http://localhost:4000`; check `http://localhost:4000/api/health` for Mongo connectivity. Keep both terminal processes alive while using the app. `npm run build` builds the frontend and checks TypeScript.

## Local API smoke check

Use a REST client or PowerShell. First bootstrap one superadmin account (only works once, while no admin account exists):

```powershell
$base = 'http://localhost:4000'
$admin = Invoke-RestMethod -Method Post -Uri "$base/api/auth/bootstrap" -ContentType 'application/json' -Body (@{name='Organizer';email='organizer@example.edu';password='Choose-A-Long-Unique-Password';key='the-same-value-as-INITIAL_ADMIN_KEY'} | ConvertTo-Json)
$headers = @{ Authorization = "Bearer $($admin.token)" }
$event = Invoke-RestMethod -Method Post -Uri "$base/api/events" -Headers $headers -ContentType 'application/json' -Body (@{title='Local typing event';capacity=20;rounds=@(@{name='Qualifier';durationSec=60;passage='The quick brown fox jumps over the lazy dog. Consistent practice improves typing speed.'})} | ConvertTo-Json -Depth 5)
$event
```

The API surface includes student signup/login, registration, host event creation and round actions, result submission, leaderboard reads, health, and Cloudinary upload signature generation. A full browser flow for official student participation has not yet been connected to the dashboard; the typing arena and overview stats are demo UI. Do not use the UI demo as an official competition.

## Deploy on Render

This repository includes a `render.yaml` Blueprint with a Node API web service and a static frontend service.

1. Push this repository to GitHub and choose **New → Blueprint** in Render. Connect the repository and apply the Blueprint.
2. On the API service, set `MONGODB_URI` to the rotated Atlas URI. Set `CLIENT_ORIGIN` to the exact deployed frontend origin (for example `https://typearena-frontend.onrender.com`, without a trailing slash).
3. Set the Cloudinary cloud name, API key, and API secret in the API service environment. Keep these values out of frontend environment variables.
4. Render generates `JWT_SECRET` and `INITIAL_ADMIN_KEY`. Copy the initial admin key privately; use it once to call `POST https://<api-host>/api/auth/bootstrap`, then remove `INITIAL_ADMIN_KEY` from the API environment and redeploy.
5. Wait for `/api/health` to report `database: "connected"`. If it does not, check the Atlas database user, network access list, and URI encoding for special characters.
6. The static frontend currently serves the dashboard demo. The browser UI has not yet been wired to the API, so event creation, official registration, live scoring, and results are not end-to-end browser features yet.

Render free services can sleep when idle, so the first request after inactivity may take longer. For production competitions, choose a persistent paid instance and verify WebSocket support and database/network settings before announcing an event.

## Deployment boundary

The API enforces server-side event limits, role checks, round timing, and submission scoring. Authentication and official participation workflows are not yet fully integrated into the browser UI; there is no certificate generation, complete audit workflow, qualification engine, or multi-instance Redis setup. The sample leaderboard and charts are illustrative only.
