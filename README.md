# TrackitNow

TrackitNow is a full-stack task and habit tracking app. Users complete tasks (fitness, learning, coding, health), and the backend turns that activity into streaks, points, badges, and a GitHub-style contribution graph. It also has a friend system with direct messaging.

Live demo: https://trackitnow.vercel.app

## Stack

**Frontend:** React 18, TypeScript, Vite, React Router, Tailwind CSS, Chart.js (via react-chartjs-2), Framer Motion, Axios

**Backend:** FastAPI (Python), SQLAlchemy ORM, PostgreSQL, JWT auth (python-jose), Argon2 password hashing (passlib), Cloudinary for image storage

The frontend and backend are two separate apps in one repo (`frontend/`, `backend/`), talking over a REST API. There is no server-side rendering or shared runtime — the frontend is a static SPA build, the backend is a stateless API process.

## How it works

### Auth

Signup hashes the password with Argon2 and stores the user row. Signin checks the password and returns a JWT (`sub` = user id, 30 minute expiry) via the OAuth2 password flow. Every protected route depends on `get_current_user`, which decodes the token and loads the user from the database — there's no server-side session store, the token is the only state.

### Tasks, sessions, and streaks

Tasks are a single table shared by everyone. A task with `user_id = NULL` is a built-in default task; a task with a `user_id` set is a custom task created by that user. Whether a *specific* user has started or finished a task is tracked separately in `user_tasks`, so the same task can be `pending` for one user and `done` for another.

When a task is marked `done`, the API doesn't just flip a status flag — it also upserts a `sessions` row for that user and date, incrementing a `task_count`. The activity heatmap and the streak calculation are both derived from these session rows rather than from the tasks table directly: the streak walks backward day-by-day from today counting consecutive session dates, and the 12-month activity endpoint just returns a date → task_count map for the frontend to render as a grid.

### Points, badges, rank

Points are computed on the fly as `completed_tasks * 10` rather than stored. Badges are a fixed list (bronze through diamond) matched against the earned badge types on the user; the response for each badge includes whether it's been earned. Rank in the profile response is currently a static value, and weekly goals are a placeholder list — both are stubbed pending a real leaderboard and goals system.

### Friends and chat

Friendship is one row per pair with a `status` of `pending` or `accepted`; there's no separate "requests" table, a pending friendship *is* the request. Chat reuses this: a chat's identity is the friendship id, and messages between two users are queried directly by sender/receiver pair rather than through a chat/thread table. Marking a chat as read is a bulk update on unread messages from that friend.

### Images

Profile photo uploads go straight to Cloudinary from the backend (not stored locally or in Postgres); the returned secure URL is saved on the user row.

## Project structure

```
TrackitNow/
├── backend/
│   ├── main.py          # FastAPI app: all routes, auth, business logic
│   ├── models.py        # SQLAlchemy models (User, Task, UserTask, Friendship, Message, Session, UserBadge)
│   ├── schemas.py        # Pydantic request/response schemas
│   ├── database.py       # Engine, session factory, DATABASE_URL config
│   └── requirements.txt
│
└── frontend/
    ├── src/
    │   ├── components/
    │   ├── pages/
    │   ├── services/      # Axios API client
    │   ├── types/
    │   └── utils/
    ├── package.json
    └── vite.config.ts
```

## Data model

| Table | Purpose |
|---|---|
| `users` | Account, profile fields, social links |
| `tasks` | Default tasks (`user_id` null) and custom tasks (`user_id` set) |
| `user_tasks` | Per-user status of a task: pending, progress, done |
| `sessions` | One row per user per active day, with a completed-task count |
| `friendships` | One row per pair; status is pending or accepted |
| `messages` | Direct messages between two users |
| `user_badges` | Badges a user has earned |

## API

```
Auth
POST   /api/auth/signup
POST   /api/auth/signin
POST   /api/auth/signout

Profile
GET    /api/user/profile
PUT    /api/user/profile
POST   /api/user/profile/photo
DELETE /api/user/account

Tasks
GET    /api/tasks
POST   /api/tasks
PUT    /api/tasks/:id/status

Friends
GET    /api/friends
GET    /api/friends/search
POST   /api/friends/request
GET    /api/friends/requests
PUT    /api/friends/requests/:id/accept
PUT    /api/friends/requests/:id/decline

Chat
GET    /api/chats
GET    /api/chats/:id/messages
POST   /api/chats/:id/messages
PUT    /api/chats/:id/read

Progress
GET    /api/progress/activity
GET    /api/progress/badges
GET    /api/progress/goals
```

Interactive docs once the backend is running: `http://localhost:8000/docs`

## Setup

### Requirements

- Python 3.9+
- Node.js 18+
- PostgreSQL (or a Supabase project)
- Cloudinary account (free tier is enough)

### Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate     # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env         # fill in DATABASE_URL, SECRET_KEY, CLOUDINARY_*
python main.py
```

Tables are created automatically on startup via `models.Base.metadata.create_all`. The API runs at `http://localhost:8000`.

### Frontend

```bash
cd frontend
npm install
cp .env.example .env         # set VITE_API_URL and Cloudinary vars
npm run dev
```

Runs at `http://localhost:5173`.

## Known limitations

- Leaderboard rank is hardcoded rather than computed from real user data.
- Weekly goals are placeholder data, not backed by a table yet.
- Chat is functional but rough — message delivery isn't real-time (no websockets), it's poll/fetch based.
- No automated test suite yet.

## Deployment

- Frontend: Vercel (`vercel --prod`) or any static host after `npm run build`.
- Backend: any host that can run a long-lived FastAPI/uvicorn process (Railway, Render, etc.), pointed at a Postgres instance.

## License

MIT
