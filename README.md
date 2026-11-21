# Campus Safe

Campus Emergency Assistance and Incident Coordination Platform.
React + Vite + Tailwind CSS (client) and Node.js + Express + Socket.IO + PostgreSQL (server).

There are two experiences on one backend:

- **Public SOS page** (`/`): no login. Four big buttons: Medical, Fire, Accident, Harassment / Women Safety.
- **Campus Portal** (`/login`, `/portal/...`): students, security, wardens, medical staff and super admin.

No map service is used anywhere. GPS coordinates are only shown as text.

## 1. What you need

- Node.js 20 or newer (`node -v`)
- A PostgreSQL database. Easiest is a free **Neon** project (https://neon.tech). A local Postgres also works.

## 2. Setup (about 5 minutes)

```bash
# from the campus-safe folder
npm run install:all

# server settings
cp server/.env.example server/.env
```

Open `server/.env` and set:

- `DATABASE_URL`: the connection string from the Neon dashboard (keep `?sslmode=require`).
- `JWT_SECRET`: any long random text (16+ characters).
- For a local non-SSL Postgres, set `DATABASE_SSL=false`.

Create the tables and demo data:

```bash
npm run db:setup
```

Run it (two terminals):

```bash
npm run dev:server     # API on http://localhost:5000
npm run dev:client     # website on http://localhost:5173
```

Open http://localhost:5173. The SOS page shows first.

### Demo accounts (created by the seed)

Password for all: `Password@123`

| Role | Email |
|---|---|
| Super admin | admin@campus.test |
| Security (coordinator + responder) | security@campus.test, security2@campus.test |
| Warden | warden@campus.test |
| Medical staff | medical@campus.test, medical2@campus.test |
| Student | student@campus.test |

Change these passwords (Users > Reset password) before any real use. Students can also register themselves.

**First thing as admin:** go to Configuration > Emergency contacts and type your real campus numbers. They are empty on purpose. The SOS page shows "Number not set yet" until you do.

## 3. Try the six workflows

1. **Public emergency.** Open `/`, press a button. Allow location when asked. You get a reference like `CS-EM-20261117-A1B2C`, a 5-second cancel button and live status.
2. **Complaint.** Log in as the student, Report an incident, attach a PDF or image, submit, then track it.
3. **Security coordination.** Log in as security in another browser. The SOS appears instantly (with a beep). Acknowledge, assign a responder, then log in as that responder to Accept, update status, message the team and Resolve.
4. **Warden and medical.** They only see incidents routed to their department (see the routing table in Configuration).
5. **Admin.** Analytics with date filter, CSV/PDF export, users, departments, routing rules, escalation times, audit log.
6. **Simulation.** Simulation centre > Start. Nothing real is created or sent. You get a scorecard.

## 4. Automated API test

With the server running against a seeded database:

```bash
npm run test:api
```

It runs about 70 checks covering all six workflows (permissions, confidentiality, uploads, cancellation, escalation endpoint, simulations, exports). Login and alert endpoints are rate limited, so wait a minute between runs.

## 5. Project layout

```
server/
  db/schema.sql        tables, indexes and constraints
  db/migrate.js        creates the tables
  db/seed.js           departments, routing rules, demo users, scenarios
  src/index.js         Express + Socket.IO + auto-escalation job
  src/auth.js          JWT login check and role guard
  src/services.js      who may see an incident, status changes
  src/notify.js        save notification, then push by socket
  src/routes/          auth, public (SOS), incidents, misc, admin, simulations
client/src/
  pages/Home.jsx       public SOS page
  pages/               portal pages (dashboards, incident detail, admin ...)
  components/          layout, tables, progress tracker
```

## 6. How the main rules are enforced

- **Role comes from the account**, never from the login form. Every API route checks it again on the server.
- **Who sees what:** a staff member sees an incident only if their department is its primary or supporting department. Admins see all. Students see their own. Medical staff never see harassment or ragging reports.
- **Internal notes and internal messages** are filtered out on the server for students.
- **SOS alert is saved and sent first.** Cancelling afterwards marks it CANCELLED (never resolved) and tells the team.
- **GPS never blocks an alert.** The alert waits at most 2.5 seconds for GPS, then goes out. Location is attached when it arrives, or the sender can type it.
- **Notifications are saved in PostgreSQL first**, then pushed. They are reloaded on reconnect.
- **Sockets:** a valid token is required, and rooms are chosen by the server from the account.
- **Auto-escalation:** every 30 seconds the server flags unacknowledged reports past their priority's time limit, notifies the department and admins, and suggests an available responder. It never resolves anything.
- **Uploads:** JPG, PNG, WebP, PDF, Word; 5 MB; 5 files; random stored names; only downloadable by people allowed to see that incident.
- **Abuse protection:** rate limits on SOS and login, duplicate-alert protection, input validation, parameterised SQL, Helmet headers, CSV formula protection.

## 7. Going live

```bash
npm run build          # builds client/dist
npm start              # Express serves the API and the built website on one port
```

- Set `NODE_ENV=production`, a strong `JWT_SECRET`, `CLIENT_ORIGIN` (your site URL) and `TRUST_PROXY=true` if behind a proxy.
- **Use HTTPS.** Browsers only allow GPS on HTTPS (or localhost).
- If the website and API are on different domains, set `VITE_API_URL` and `VITE_SOCKET_URL` in `client/.env` before building.
- Uploaded files are saved in `server/uploads`. Many hosts wipe local disk on deploy, so use a persistent disk or move uploads to cloud storage.

## 8. Honest limits (please read)

- **Not tested in a real browser by the author.** The API is tested end to end and the client builds without errors, but the screens were not visually checked. Open every page once and report anything odd.
- **ShadCN UI is not used.** The UI uses Tailwind and small hand-written components instead.
- **No ORM.** There was no existing project, so it uses plain SQL with `pg`. Tables are created from `db/schema.sql`.
- **Uploads go to local disk**, not Cloudinary.
- **Password reset email is not sent.** There is no email service. In development the reset code is returned on screen. For production, send the link by email in `routes/auth.js` (marked).
- **The guidance assistant is a fixed, reviewable FAQ**, not an LLM, so nothing is invented. Have your medical and safety teams approve the text in `client/src/pages/Safety.jsx`.
- **Not included:** notification-settings configuration, a campus location directory table, and hiding medical details from admins (admins see everything routed to the system).
- **Location updates** only run while the confirmation page stays open. Nothing tracks after the browser closes.
- **Browser storage:** the login token is kept in localStorage.
- **Emergency services are not contacted.** The app says so on screen. Never rely on this software alone for real emergencies.
