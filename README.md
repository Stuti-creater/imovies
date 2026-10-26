# IMovies: movie ticket booking (demo MVP)

Cinematic movie-booking web app: browse, search, filter, pick a showtime, choose seats, get a booking reference, see booking history. Payment is **simulated**: every booking is labelled a demonstration reservation. Showtimes, theatres and prices are **sample data**, not real cinema schedules.

**Stack:** React 18 + Vite, React Router, Axios, Lucide, Bootstrap 5 utilities + custom CSS · Node.js, Express, Mongoose, JWT, bcryptjs · MongoDB Atlas.

```
IMovies/
  client/   React app (Vite)        → deploy to Vercel
  server/   Express REST API        → deploy to Render
```

## Run locally

Prerequisites: Node 18+ and a MongoDB (Atlas free cluster or local `mongod`).

```bash
# 1) API
cd server
cp .env.example .env            # then edit MONGO_URI and JWT_SECRET
npm install
npm run seed                    # demo movies + shows (safe to re-run)
npm run dev                     # http://localhost:5000/api/health

# 2) Web app (new terminal)
cd client
cp .env.example .env            # VITE_API_URL=http://localhost:5000/api
npm install
npm run dev                     # http://localhost:5173
```

Generate a JWT secret: `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"`

### Create the first admin
Admin rights live in the database (`role: 'admin'`). Public sign-up can never set a role.
```bash
cd server
ADMIN_EMAIL=you@example.com ADMIN_PASSWORD='StrongPass123' npm run create-admin
```
If the user already exists it is promoted (password unchanged). Log in, and an **Admin** link appears.

## Environment variables

| Where | Name | Purpose |
|---|---|---|
| server | `PORT` | API port (Render sets this itself) |
| server | `MONGO_URI` | MongoDB connection string |
| server | `JWT_SECRET` | Token signing secret. **Required in production**; the server refuses to start without a real one |
| server | `JWT_EXPIRES_IN` | Token lifetime, default `7d` |
| server | `CLIENT_URL` | Allowed frontend origin(s), comma-separated, no trailing slash |
| server | `ADMIN_*` | Only for `npm run create-admin` |
| client | `VITE_API_URL` | API base URL including `/api` |

`.env` files are git-ignored. Never commit secrets.

## How double-booking is prevented
Seat reservation is one atomic MongoDB update on the `Show` document: it only matches if **none** of the requested seats are already in `bookedSeats`, and then pushes them in the same operation. When two users race for a seat, exactly one update matches; the other gets `409`. This needs **no transaction and no replica set**. If creating the `Booking` fails afterwards, the seats are released again. Price is always computed from the database, never from the client.
Known limit: a hard crash between the two steps could leave seats reserved; a periodic reconciliation job (compare `Show.bookedSeats` with confirmed bookings) is the standard fix.

## API

Base: `/api`. Errors are `{ message, details? }`. 🔒 = Bearer token, 👑 = admin role.

| Method | Path | Notes |
|---|---|---|
| GET | `/health` | status + DB state |
| POST | `/auth/register` | `{name,email,password}` → `{token,user}` (password 8+ chars, letter + number) |
| POST | `/auth/login` | `{email,password}` → `{token,user}` |
| GET | `/auth/me` 🔒 | current user |
| GET | `/movies?q=&genre=` | search by title, filter by genre |
| GET | `/movies/:id` | movie details |
| GET | `/movies/:id/shows` | upcoming shows (with `bookedCount`) |
| GET | `/shows/:id` | one show incl. `bookedSeats` for the seat map |
| POST | `/bookings` 🔒 | `{showId, seats:["A1","A2"]}` (max 6) → booking + reference |
| GET | `/bookings/my` 🔒 | booking history |
| POST/PUT/DELETE | `/admin/movies[/:id]` 👑 | manage movies |
| GET/POST/PUT/DELETE | `/admin/shows[/:id]` 👑 | manage shows |
| GET | `/admin/bookings` 👑 | all bookings |

Status codes: 400 validation · 401 auth · 403 role · 404 · 409 seats taken/duplicate · 429 rate-limited auth.

## Tests
```bash
cd server && npm test          # unit tests (validators, seat parsing)
cd client && npm run build     # production build check
```

## Deployment checklist

1. **GitHub:** `git init && git add . && git commit -m "IMovies MVP"`, create a repo, push. Confirm no `.env` is tracked (`git ls-files | grep .env` should only show `.env.example`).
2. **MongoDB Atlas:** create a free cluster → Database Access: add a user → Network Access: allow `0.0.0.0/0` (Render free IPs change) → copy the `mongodb+srv://…` string, add a database name (e.g. `/imovies`).
3. **Render (API):** New Web Service → repo → *Root Directory* `server` → Build `npm install` → Start `npm start`. Env vars: `NODE_ENV=production`, `MONGO_URI`, `JWT_SECRET`, `CLIENT_URL` (your Vercel URL, set after step 4). Check `https://<service>.onrender.com/api/health`.
4. **Vercel (web):** Import repo → *Root Directory* `client` → framework Vite. Env var `VITE_API_URL=https://<service>.onrender.com/api`. `vercel.json` already rewrites routes to `index.html`. Then put the Vercel URL into Render's `CLIENT_URL` and redeploy.
5. **Seed demo data (optional):** from your machine with the Atlas URI in `server/.env`: `ALLOW_SEED=true NODE_ENV=production npm run seed`. Seeding never deletes real bookings. Then `npm run create-admin`.

## Troubleshooting
- **CORS error in browser:** `CLIENT_URL` must exactly match the frontend origin (scheme + host, no trailing slash). Redeploy the API after changing it.
- **"Cannot reach the server":** wrong `VITE_API_URL`, or the free Render service is waking up (first request can take ~50 s).
- **Atlas connection timeout:** add `0.0.0.0/0` to Network Access; check the password is URL-encoded.
- **Server exits with JWT_SECRET error in production:** set a real secret.
- **Posters blank:** artwork is hot-linked from the TMDB image CDN; broken links fall back to a lettered card. Replace URLs in `server/src/seed.js` or via Admin.
- **Everyone logged out after redeploy:** you changed `JWT_SECRET`.

## Security notes
bcrypt (cost 12) password hashing, hashes never returned · JWT verified on every protected call and the user/role re-read from the DB · admin routes enforced server-side · helmet, 20 kb body limit, auth rate limiting, CORS allow-list · input validation + whitelisted fields on admin writes (no mass assignment) · generic 500 messages, no stack traces to clients. The JWT is kept in `localStorage` for simplicity; for stricter setups use an httpOnly cookie.
