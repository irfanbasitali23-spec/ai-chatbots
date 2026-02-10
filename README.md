# AI Chatbot Appointment Booking System

A full-stack AI-powered appointment booking platform with a conversational chatbot interface built using **React**, **Node.js/Express**, **FastAPI + LangChain**, and **PostgreSQL**.

---

## Architecture Overview

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   React Frontend │────▶│  Node.js/Express │────▶│  FastAPI + LangChain│
│   (Vite + TW)   │◀────│  Backend API     │◀────│  AI Microservice    │
│   Port: 5173     │     │  Port: 3001      │     │  Port: 8000         │
└─────────────────┘     └──────┬───────────┘     └──────┬──────────────┘
                               │                        │
                               ▼                        ▼
                        ┌──────────────────────────────────┐
                        │         PostgreSQL Database       │
                        │    ai_chatbot_appointments        │
                        └──────────────────────────────────┘
```

### Service Responsibilities

| Service | Tech | Role |
|---------|------|------|
| **Frontend** | React 18, Vite, TailwindCSS, Zustand | Chat UI, auth pages, appointment management |
| **Backend API** | Node.js, Express | Auth (JWT), session management, API gateway, rate limiting |
| **AI Microservice** | FastAPI, LangChain, OpenAI | Conversational AI, appointment tool-calling, interaction logging |
| **Database** | PostgreSQL | Users, appointments, chat sessions, messages, interaction logs |

---

## Prerequisites

- **Node.js** >= 18
- **Python** >= 3.10
- **PostgreSQL** >= 14 (running locally)
- **OpenAI API Key** (for the LLM)

---

## Quick Start Guide

### 1. Create the PostgreSQL Database

Open a terminal and run:

```bash
psql -U postgres
```

Then in the psql shell:

```sql
CREATE DATABASE ai_chatbot_appointments;
\q
```

### 2. Initialize the Database Schema

```bash
psql -U postgres -d ai_chatbot_appointments -f database/schema.sql
```

This creates all tables (users, appointments, chat_sessions, chat_messages, interaction_logs), indexes, triggers, and sample data.

### 3. Setup & Start the Backend API (Node.js)

```bash
cd backend
npm install
```

Edit `.env` if your PostgreSQL credentials differ:
```
DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@localhost:5432/ai_chatbot_appointments
```

Start the server:
```bash
npm run dev
```

The backend runs on **http://localhost:3001**.

### 4. Setup & Start the AI Microservice (FastAPI)

```bash
cd ai-service
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # macOS/Linux

pip install -r requirements.txt
```

Edit `.env` with your OpenAI API key:
```
OPENAI_API_KEY=sk-your-actual-key-here
```

Start the service:
```bash
uvicorn app.main:app --reload --port 8000
```

The AI service runs on **http://localhost:8000**.

### 5. Setup & Start the Frontend (React)

```bash
cd frontend
npm install
npm run dev
```

The frontend runs on **http://localhost:5173**.

### 6. Test the Application

1. Open **http://localhost:5173** in your browser
2. **Sign up** with a new account (or use sample: `admin@example.com` / `password123`)
3. Start a **new chat** and try:
   - "I'd like to book an appointment"
   - "What providers are available?"
   - "Check availability for Dr. Sarah Wilson next Monday"
   - "Book a general checkup with Dr. Sarah Wilson"
   - "Show my appointments"
   - "Cancel my appointment"

---

## Key Design Decisions & Tradeoffs

### Architecture
- **Microservice separation**: The AI service is decoupled from the main API. This allows independent scaling, different deployment strategies, and isolates LLM latency from the main API.
- **API Gateway pattern**: The Node.js backend acts as a gateway — it handles auth, validation, rate limiting, and proxies chat requests to the AI service. The frontend never talks directly to the AI service.

### AI & LangChain
- **Tool-calling agent**: Uses LangChain's OpenAI tools agent pattern. The LLM decides which tools to call (check availability, book, cancel, list providers) based on conversation context.
- **Multi-turn memory**: In-memory conversation manager with DB fallback. On first message, history is loaded from PostgreSQL; subsequent messages use in-memory cache for speed.
- **Prompt engineering**: System prompt defines the assistant's persona, capabilities, and constraints. It guides the LLM to collect required information before booking.

### Authentication
- **JWT-based**: Stateless auth with short-lived chatbot tokens (15min) for the AI service. Main tokens last 24h.
- **Separate token endpoint**: `POST /api/chatbot/token` generates scoped tokens specifically for chatbot interactions.

### Database
- **UUID primary keys**: Globally unique, no sequential ID leakage.
- **JSONB metadata columns**: Flexible storage for tool call results, LLM metadata without schema changes.
- **Partial indexes**: Conditional indexes on `business_id` (WHERE NOT NULL) for multi-tenancy efficiency.
- **Unique constraint on provider slots**: Prevents double-booking at the database level.

### Frontend
- **Zustand for state**: Lightweight, no boilerplate compared to Redux. Perfect for this scale.
- **Optimistic updates**: User messages appear instantly; AI response streams in after.
- **Vite proxy**: Dev server proxies `/api` to the backend, avoiding CORS issues.

---

## Assumptions & Known Limitations

1. **LLM dependency**: Requires a valid OpenAI API key. Without it, the chatbot won't respond.
2. **No WebSocket**: Uses HTTP polling (request/response). WebSocket would improve real-time feel but adds complexity.
3. **In-memory conversation cache**: Lost on AI service restart. DB fallback ensures no data loss.
4. **Simulated provider catalog**: Providers are hardcoded in the AI service. In production, this would be a database table.
5. **No email verification**: Signup doesn't verify email. Production would need email confirmation.
6. **Single-tenant by default**: Multi-tenancy schema is prepared (`business_id` columns) but not enforced in application logic.

---

## API Endpoints

### Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/signup` | Register new user |
| POST | `/api/auth/login` | Login, returns JWT |
| GET | `/api/auth/me` | Get current user profile |

### Chatbot
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/chatbot/token` | Generate short-lived chatbot token |
| POST | `/api/chatbot/sessions` | Create new chat session |
| GET | `/api/chatbot/sessions` | List user's chat sessions |
| GET | `/api/chatbot/sessions/:id/messages` | Get session messages |
| POST | `/api/chatbot/sessions/:id/messages` | Send message (proxied to AI) |
| PATCH | `/api/chatbot/sessions/:id/close` | Close a session |

### Appointments
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/appointments` | Create appointment |
| GET | `/api/appointments` | List user appointments |
| GET | `/api/appointments/availability` | Check provider availability |
| GET | `/api/appointments/:id` | Get appointment details |
| PATCH | `/api/appointments/:id` | Update appointment |
| DELETE | `/api/appointments/:id` | Cancel appointment |

### AI Service
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/chat` | Process chat message |
| GET | `/health` | Health check |

---

## Project Structure

```
ai-chatbot-appointment/
├── README.md
├── database/
│   └── schema.sql              # PostgreSQL DDL + sample data
├── backend/                    # Node.js/Express API
│   ├── package.json
│   ├── .env
│   └── src/
│       ├── server.js           # Express app entry
│       ├── config/             # App config, logger
│       ├── db/                 # DB pool, init script
│       ├── middleware/         # Auth, validation, rate limiting, errors
│       ├── routes/             # Auth, chatbot, appointments
│       └── services/           # Business logic
├── ai-service/                 # FastAPI + LangChain
│   ├── requirements.txt
│   ├── .env
│   └── app/
│       ├── main.py             # FastAPI app entry
│       ├── config.py           # Settings (pydantic-settings)
│       ├── database.py         # SQLAlchemy engine
│       ├── models.py           # ORM models
│       ├── schemas.py          # Pydantic request/response
│       ├── routes/
│       │   └── chat.py         # Chat endpoint
│       └── services/
│           ├── chatbot.py      # LangChain agent + conversation manager
│           └── appointment_tools.py  # LangChain tools
└── frontend/                   # React + Vite
    ├── package.json
    ├── vite.config.js
    ├── tailwind.config.js
    ├── index.html
    └── src/
        ├── main.jsx
        ├── App.jsx             # Router
        ├── index.css           # Tailwind + custom styles
        ├── lib/
        │   └── api.js          # API client
        ├── store/
        │   ├── authStore.js    # Auth state (Zustand)
        │   └── chatStore.js    # Chat state (Zustand)
        ├── components/
        │   ├── Layout.jsx      # App shell with nav
        │   ├── ChatSidebar.jsx # Session list
        │   ├── ChatMessage.jsx # Message bubble
        │   └── TypingIndicator.jsx
        └── pages/
            ├── Login.jsx
            ├── Signup.jsx
            ├── Chat.jsx        # Main chat interface
            └── Appointments.jsx
```

## Submission Details

See the prepared submission draft: submission.md

Instructions: copy the contents of `submission.md` into a Google Doc, add your name and email on the first page (already included), set the doc share settings to "Anyone with the link" (Viewer), and submit the shareable link to the required submission URL.

