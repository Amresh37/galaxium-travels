# 🚀 Galaxium Travels — Interplanetary Booking System

A full-stack space-flight booking system with a React 19 frontend and a Python FastAPI backend that exposes the same business logic over both REST and MCP (Model Context Protocol).

## 🌟 Features

- **Space-Themed UI** — Responsive interface with animated starfield
- **Full Booking System** — Browse flights, make bookings, manage reservations
- **Dual Protocol Backend** — REST API and MCP support on the same server
- **Type-Safe** — TypeScript frontend with strict mode; Python type hints throughout
- **User Management** — Name/email authentication with session context
- **Demo Data** — Auto-seeded users, flights, and bookings on every server start

## 🏗️ Architecture

```
galaxium-travels/
├── booking_system_backend/       # FastAPI + FastMCP server (Python)
│   ├── server.py                 # REST & MCP routes, app lifecycle
│   ├── db.py                     # SQLAlchemy session and engine setup
│   ├── models.py                 # SQLAlchemy ORM models
│   ├── schemas.py                # Pydantic request/response schemas
│   ├── seed.py                   # Demo data seeding
│   ├── services/                 # Business logic (user, flight, booking)
│   │   ├── user.py
│   │   ├── flight.py
│   │   └── booking.py
│   ├── tests/                    # pytest test suite
│   ├── requirements.txt
│   └── Dockerfile
│
├── booking_system_frontend/      # React 19 + TypeScript + Vite
│   ├── src/
│   │   ├── components/           # Reusable UI components
│   │   ├── pages/                # Route-level pages
│   │   ├── hooks/                # Custom React hooks (UserProvider, etc.)
│   │   ├── services/             # API layer (axios instance + helpers)
│   │   ├── types/                # TypeScript type definitions
│   │   └── utils/                # Utility functions
│   └── package.json
│
└── start.sh                      # macOS/Linux one-command startup
```

## 🚀 Quick Start

### Prerequisites

- **Python 3.8+** — [Download](https://www.python.org/downloads/)
- **Node.js 18+** — [Download](https://nodejs.org/)

### Option 1: One-Command Start (macOS/Linux)

```bash
./start.sh
```

This will:
- Install all dependencies
- Start the backend on port **8080**
- Start the frontend dev server on port **5173**

### Option 2: Manual Start

#### Backend
```bash
cd booking_system_backend
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python server.py
```

#### Frontend (new terminal)
```bash
cd booking_system_frontend
npm install
npm run dev
```

## 🌐 Endpoints

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend REST API | http://localhost:8080 |
| Swagger / OpenAPI docs | http://localhost:8080/docs |
| MCP endpoint | http://localhost:8080/mcp |

## 🎯 User Guide

### Booking a Flight

1. **Browse Flights** — Navigate to the Flights page to see available routes
2. **Search & Filter** — Use the search bar to find specific destinations
3. **Sign In / Register** — Click "Book Now" and enter your name and email
4. **Confirm Booking** — Review flight details and confirm your reservation
5. **Manage Bookings** — View and cancel bookings from the "My Bookings" page

### Demo Data

The database is wiped and re-seeded on every server start with:
- **10 Users** — Alice, Bob, Charlie, Diana, Eve, Frank, Grace, Heidi, Ivan, Judy
- **10 Flights** — Routes between Earth, Mars, Moon, Venus, Jupiter, Europa, Pluto
- **20 Sample Bookings** — Mix of `booked`, `cancelled`, and `completed` statuses

## 🛠️ Technology Stack

### Backend
| Package | Purpose |
|---|---|
| FastAPI | Web framework |
| FastMCP | MCP protocol layer |
| SQLAlchemy | ORM |
| Pydantic v2 | Data validation & serialisation |
| SQLite | Database (file: `booking.db`) |
| Uvicorn | ASGI server |

### Frontend
| Package | Purpose |
|---|---|
| React 19 | UI library |
| TypeScript 5 | Type safety |
| Vite 7 | Build tool |
| Tailwind CSS 3 | Styling |
| React Router 7 | Routing |
| Framer Motion | Animations |
| Axios | HTTP client |
| Lucide React | Icons |
| React Hot Toast | Notifications |

## 🧪 Testing

```bash
cd booking_system_backend
pytest                             # all tests
pytest tests/test_services.py     # service layer only
pytest tests/test_rest.py         # REST endpoints only
```

Tests use an in-memory SQLite database; each test function gets a fresh schema. `seed()` is patched to a no-op.

## 📦 Production Deployment

### Backend
```bash
cd booking_system_backend
pip install -r requirements.txt
uvicorn server:app --host 0.0.0.0 --port 8080
```

### Frontend
```bash
cd booking_system_frontend
npm run build
# Deploy the generated 'dist/' folder to your static hosting service
```

Both `booking_system_backend/` and `booking_system_frontend/` include a `Dockerfile` for containerised deployment.

## 🎨 Configuration

### Change the API URL
Create or edit `booking_system_frontend/.env`:
```env
VITE_API_URL=https://your-api-url.com
```

### Modify Theme Colors
Edit `booking_system_frontend/tailwind.config.js`:
```js
colors: {
  'cosmic-purple': '#6366F1',
  'nebula-pink': '#EC4899',
}
```

## 🐛 Troubleshooting

| Problem | Fix |
|---|---|
| Backend won't start | Check Python version (`python --version`, needs 3.8+) and that port 8080 is free |
| Frontend won't start | Check Node version (`node --version`, needs 18+) and that port 5173 is free |
| `npm install` errors | Delete `node_modules/` and rerun `npm install` |
| API connection refused | Verify backend is running and `VITE_API_URL` in `.env` is correct |

## 📄 License

MIT — see [LICENSE](LICENSE).

---

*Explore the cosmos, one booking at a time.*
