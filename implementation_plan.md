# StaySync — Team Development Plan (3 Members)

> **Goal**: Build StaySync (Hotel Price Aggregator) as a team of 3, following professional practices, while **actually learning** the tech — not just copy-pasting AI-generated code.

---

## 🧭 Before You Write a Single Line of Code

### Step 0: Understand What You're Building

StaySync is a **hotel price aggregator** — think of it like Google Flights, but for hotel rooms. It:
1. Pulls hotel prices from multiple platforms (Booking, Expedia, etc.) via RapidAPI
2. Shows them on a map + list view so users can compare
3. Uses ML to predict future prices and analyze review sentiment
4. Lets users bookmark hotels, write reviews, and get the best deal

> [!TIP]
> **Before coding, each team member should spend ~1 hour** using Google Flights, Trivago, or Kayak. Notice how they handle search, filters, map views, and price comparisons. That's the UX you're targeting.

---

## 🏗️ System Architecture — Explained for Beginners

Here's how all the pieces connect. Read this section together as a team.

```
┌──────────────────────────────────────────────────────────┐
│  YOUR LAPTOP (Development)                                │
│                                                          │
│  ┌──────────┐  HTTP    ┌──────────┐  DB     ┌─────────┐ │
│  │ Frontend │ ───────▶ │ Backend  │ ──────▶ │ MongoDB │ │
│  │ (Next.js │ ◀─────── │ (Express)│ ◀────── │         │ │
│  │  :3000)  │  JSON    │  (:5000) │         └─────────┘ │
│  └──────────┘          └────┬─────┘                      │
│                              │         ┌─────────┐       │
│                              │ cache   │  Redis  │       │
│                              ├───────▶ │ (:6379) │       │
│                              │         └─────────┘       │
│                              │                            │
│                              │ HTTP    ┌──────────┐      │
│                              └───────▶ │ML Service│      │
│                                        │ (Flask   │      │
│                                        │  :8000)  │      │
│                                        └──────────┘      │
└──────────────────────────────────────────────────────────┘
```

### What Each Piece Does (Plain English)

| Component | What it is | Analogy |
|-----------|-----------|---------|
| **Next.js Frontend** | The website users see and click on. Renders pages, shows maps, handles forms. | The restaurant's dining area — what customers interact with |
| **Express Backend** | The brain. Receives requests from the frontend, talks to databases, fetches hotel data from external APIs. | The kitchen — processes orders, prepares food |
| **MongoDB** | Stores all persistent data — users, hotels, reviews, tokens. | The pantry — stores all ingredients |
| **Redis** | A super-fast temporary store. Caches frequent queries so you don't hit the external API every time. | A prep counter — keeps frequently used items handy |
| **ML Service (Flask)** | A separate Python app that runs machine learning models — price prediction and sentiment analysis. | A specialized chef — only handles desserts (ML tasks) |
| **RapidAPI** | External service that gives you hotel data from Booking.com, Expedia, etc. You pay per request. | The food supplier — you order ingredients from them |

### How a User Search Actually Works (Step by Step)

```
1. User types "Mumbai" and clicks Search on the website
2. Frontend sends GET /api/hotels/search?city=Mumbai to Backend
3. Backend checks Redis: "Do I already have recent results for Mumbai?"
   → YES (cache hit): Return cached data instantly
   → NO  (cache miss): Continue to step 4
4. Backend calls RapidAPI: "Give me hotels in Mumbai"
5. RapidAPI returns raw data (different format per platform)
6. Backend NORMALIZES data into one consistent shape
7. Backend saves normalized data to MongoDB (for history)
8. Backend saves to Redis with 15-min TTL (for caching)
9. Backend sends clean JSON response to Frontend
10. Frontend renders hotel cards + map pins
```

> [!IMPORTANT]
> **Every team member must understand this flow.** Draw it on paper. Discuss it. If someone asks "what happens when a user searches?", everyone should be able to explain it.

---

## 📂 Folder Structure — What Goes Where

```
StaySync/
├── docker-compose.yml          ← Runs everything together (Phase 4)
├── .github/workflows/          ← CI/CD pipeline (Phase 4)
├── .env.example                ← Template for environment variables
├── README.md                   ← Project documentation
│
├── backend/                    ← Nikhil's primary territory
│   ├── package.json
│   ├── server.js               ← Entry point: creates Express app, connects DB
│   ├── config/
│   │   ├── db.js               ← MongoDB connection logic
│   │   └── redis.js            ← Redis connection logic
│   ├── models/                 ← Mongoose schemas (data shape definitions)
│   │   ├── User.js
│   │   ├── Hotel.js
│   │   ├── Review.js
│   │   └── RefreshToken.js
│   ├── routes/                 ← URL → Controller mapping
│   │   ├── auth.js             ← /api/auth/login, /register, /refresh
│   │   ├── hotels.js           ← /api/hotels/search, /:id
│   │   ├── reviews.js          ← /api/reviews (CRUD)
│   │   └── ml.js               ← /api/ml/predict, /sentiment
│   ├── controllers/            ← Business logic for each route
│   │   ├── authController.js
│   │   ├── hotelController.js
│   │   ├── reviewController.js
│   │   └── mlController.js
│   ├── middleware/              ← Code that runs BEFORE your controller
│   │   ├── authMiddleware.js   ← Checks JWT token validity
│   │   ├── rateLimiter.js      ← Prevents spam/abuse
│   │   └── errorHandler.js     ← Catches errors, sends clean responses
│   └── services/               ← Reusable business logic
│       ├── rapidApiService.js  ← Talks to RapidAPI
│       ├── cacheService.js     ← Redis read/write helpers
│       └── mlService.js        ← Talks to Python ML service
│
├── frontend/                   ← Shalu's primary territory
│   ├── package.json
│   ├── next.config.js
│   ├── app/                    ← Next.js App Router pages
│   │   ├── layout.js           ← Shared layout (navbar, footer)
│   │   ├── page.js             ← Landing page (/)
│   │   ├── search/page.js      ← Search results (/search)
│   │   ├── hotel/[id]/page.js  ← Hotel detail (/hotel/abc123)
│   │   ├── auth/
│   │   │   ├── login/page.js
│   │   │   └── register/page.js
│   │   └── dashboard/page.js   ← User dashboard
│   ├── components/             ← Reusable UI building blocks
│   │   ├── MapView.jsx
│   │   ├── HotelCard.jsx
│   │   ├── FilterSidebar.jsx
│   │   ├── PricePin.jsx
│   │   └── Navbar.jsx
│   └── lib/
│       ├── api.js              ← Axios instance with interceptors
│       └── auth.js             ← Token storage + refresh logic
│
└── ml-service/                 ← Diksha's primary territory
    ├── requirements.txt
    ├── Dockerfile
    ├── app.py                  ← Flask entry point
    ├── models/
    │   ├── price_predictor.py  ← scikit-learn Linear Regression
    │   └── sentiment_analyzer.py ← VADER sentiment
    ├── data/
    │   └── trained_model.pkl   ← Saved trained model
    └── tests/
        └── test_predictions.py
```

### Key Concept: Routes → Controllers → Services

This is the most important pattern in the backend. Here's how a request flows:

```
Request hits /api/auth/login
        │
        ▼
   routes/auth.js          ← "Which controller function handles this URL?"
        │                     router.post('/login', authController.login)
        ▼
   controllers/authController.js  ← "What's the business logic?"
        │                            Find user, compare password, create token
        ▼
   services/ + models/     ← "How do I talk to the database/external APIs?"
        │                     User.findOne({email}), bcrypt.compare()
        ▼
   Response sent back       ← { accessToken, user }
```

> **Route** = Traffic cop (directs requests)
> **Controller** = Manager (decides what to do)
> **Service** = Worker (does the actual work)
> **Model** = Blueprint (defines data shape)

---

## 🗄️ Database Schema — What's Stored and Why

Refer to the [database_schema.md](file:///home/nikhil/Documents/Coding/antigravity/StaySync/database_schema.md.resolved) for full Mongoose schemas. Here's the beginner summary:

### Collections (Tables in MongoDB language)

| Collection | What it stores | Key fields |
|-----------|---------------|------------|
| `users` | Registered users | name, email, password (hashed!), bookmarks[], bookingHistory[] |
| `hotels` | Hotel data aggregated from APIs | name, location (GeoJSON), currentPrices[], bestPrice, priceHistory[], amenities[] |
| `reviews` | User + scraped reviews | hotel (reference), text, rating, sentiment scores |
| `refreshtokens` | JWT refresh tokens for auth | user (reference), tokenHash, family, expiresAt (auto-deletes!) |
| `cachemetas` | Tracks what's cached in Redis | key, queryType, hitCount, expiresAt |

### Relationships Between Collections

```mermaid
erDiagram
    USER ||--o{ REVIEW : "writes"
    USER ||--o{ REFRESH_TOKEN : "has auth tokens"
    HOTEL ||--o{ REVIEW : "receives"
    USER }o--o{ HOTEL : "bookmarks (embedded in User)"
```

**Key concept — References vs Embedding:**
- **Embedded** (inside the document): `bookmarks[]` lives inside User — because a user won't have 10,000 bookmarks
- **Referenced** (separate collection): Reviews are in their own collection — because a hotel could have thousands of reviews (embedding would hit MongoDB's 16MB doc limit)

---

## 🛣️ API Routes — The Full Map

Every URL your frontend can call:

### Auth Routes (`/api/auth`)
| Method | Route | What it does | Auth needed? |
|--------|-------|-------------|-------------|
| POST | `/register` | Create new user account | ❌ |
| POST | `/login` | Login, get tokens | ❌ |
| POST | `/refresh` | Get new access token using refresh token | ❌ (uses cookie) |
| POST | `/logout` | Invalidate refresh token | ✅ |
| GET | `/me` | Get current user profile | ✅ |

### Hotel Routes (`/api/hotels`)
| Method | Route | What it does | Auth needed? |
|--------|-------|-------------|-------------|
| GET | `/search?city=X&checkin=Y&checkout=Z` | Search hotels | ❌ |
| GET | `/:id` | Get single hotel details | ❌ |
| POST | `/:id/bookmark` | Add to bookmarks | ✅ |
| DELETE | `/:id/bookmark` | Remove from bookmarks | ✅ |

### Review Routes (`/api/reviews`)
| Method | Route | What it does | Auth needed? |
|--------|-------|-------------|-------------|
| GET | `/hotel/:hotelId` | Get reviews for a hotel | ❌ |
| POST | `/` | Submit a new review | ✅ |
| PUT | `/:id` | Edit your review | ✅ |
| DELETE | `/:id` | Delete your review | ✅ |

### ML Routes (`/api/ml`)
| Method | Route | What it does | Auth needed? |
|--------|-------|-------------|-------------|
| POST | `/predict-price` | Get price prediction for a hotel | ❌ |
| POST | `/sentiment` | Analyze review sentiment | ❌ |

---

## 👥 Team Structure & Task Split

### Member Roles

| Member | Primary Role | Primary Folder | Key Technologies to Learn |
|--------|-------------|----------------|--------------------------|
| **Nikhil** 🔵 | Backend Developer | `backend/` | Node.js, Express.js, Mongoose, JWT, bcrypt, Redis |
| **Shalu** 🟢 | Frontend Developer | `frontend/` | Next.js 15 (App Router), React, Google Maps API, Axios |
| **Diksha** 🟣 | ML Engineer + DevOps | `ml-service/` + `docker/` | Python, Flask, scikit-learn, VADER, Docker |

> [!IMPORTANT]
> These are **primary** roles, not silos. Everyone should understand the full system. You'll need to coordinate at integration boundaries (e.g., "what JSON shape does the backend return?" — Nikhil and Shalu must agree on this).

---

## 📆 Development Phases

### Phase 1: Foundation (Week 1) — "Make it run"  ⏰ Deadline: end of Week 1

> **Goal**: Each member gets their service running independently, returning hardcoded/mock data.

---

#### 🔵 Nikhil — Backend Foundation

**What to learn first** (in order):
1. What is Node.js and npm? (`node -v`, `npm init -y`)
2. What is Express.js? (Build a "Hello World" server)
3. What is middleware? (A function that runs before your route handler)
4. What is Mongoose? (An ODM — it lets you define schemas for MongoDB)
5. What are environment variables? (`.env` file — stores secrets)

**Tasks:**

- [ ] **A1.1** Initialize the project
  ```bash
  mkdir backend && cd backend
  npm init -y
  npm install express mongoose dotenv cors helmet morgan
  npm install -D nodemon
  ```
  Add to `package.json` scripts: `"dev": "nodemon server.js"`

- [ ] **A1.2** Create `server.js` — Express entry point
  - Import express, set up middleware (cors, helmet, morgan, express.json)
  - Create a test route: `GET /api/health` → `{ status: "ok" }`
  - Listen on port 5000
  - **Checkpoint**: Run `npm run dev`, visit `http://localhost:5000/api/health` in browser

- [ ] **A1.3** Create `config/db.js` — MongoDB connection
  - Use `mongoose.connect(process.env.MONGO_URI)`
  - Handle connection errors and success logging
  - **Checkpoint**: See "MongoDB connected" in terminal

- [ ] **A1.4** Create ALL Mongoose models (copy from `database_schema.md`, but **type them yourself** — don't paste)
  - `models/User.js` — with password hashing pre-save hook
  - `models/Hotel.js` — with GeoJSON location, price arrays
  - `models/Review.js` — with sentiment fields
  - `models/RefreshToken.js` — with TTL index
  - **Checkpoint**: Can import models without errors

- [ ] **A1.5** Create auth routes + controller (the first real feature)
  - `routes/auth.js` + `controllers/authController.js`
  - Implement: `POST /register` and `POST /login`
  - Use bcrypt for password hashing, JWT for tokens
  - **Checkpoint**: Register a user via Postman, login gets a token back

**📚 Resources to study:**
- [Express.js Getting Started](https://expressjs.com/en/starter/installing.html)
- [Mongoose Quick Start](https://mongoosejs.com/docs/index.html)
- [JWT Introduction](https://jwt.io/introduction)
- [REST API Design Best Practices](https://restfulapi.net/)

---

#### 🟢 Shalu — Frontend Foundation

**What to learn first** (in order):
1. What is React? (Components, props, state, hooks)
2. What is Next.js? (React framework with routing, SSR, file-based pages)
3. What is the App Router? (Next.js 13+ — `app/` folder, `layout.js`, `page.js`)
4. What is Axios? (HTTP client — like `fetch()` but better)
5. What is component-based architecture? (Small reusable pieces)

**Tasks:**

- [ ] **B1.1** Initialize Next.js project
  ```bash
  npx -y create-next-app@latest frontend --js --app --no-tailwind --eslint --src-dir --no-import-alias
  ```
  (If the flags differ, run `npx create-next-app --help` first)

- [ ] **B1.2** Set up project structure
  - Create `components/`, `lib/` folders
  - Install dependencies: `npm install axios`
  - Set up a CSS approach (CSS Modules or vanilla CSS with a design system)

- [ ] **B1.3** Build the **Navbar** component
  - Logo, Search bar, Login/Register buttons
  - Responsive (mobile hamburger menu)
  - **Checkpoint**: Navbar renders on all pages

- [ ] **B1.4** Build the **Landing Page** (`app/page.js`)
  - Hero section with search bar (city, check-in, check-out)
  - Popular destinations section (hardcoded for now)
  - **Checkpoint**: Page looks polished, search form submits and navigates to `/search`

- [ ] **B1.5** Build the **Search Results Page** (`app/search/page.js`)
  - Layout: Filters sidebar (left) + Hotel card grid (right)
  - Use **hardcoded mock data** for now — an array of 5-6 fake hotels
  - HotelCard component: image, name, star rating, price, platform
  - **Checkpoint**: Shows list of hotel cards with mock data

- [ ] **B1.6** Build **Auth Pages** (`app/auth/login/page.js`, `register/page.js`)
  - Login form: email + password
  - Register form: name + email + password + confirm password
  - Client-side validation (required fields, email format, password match)
  - **Checkpoint**: Forms render and validate, submit button logs form data to console

**📚 Resources to study:**
- [Next.js App Router Tutorial](https://nextjs.org/learn)
- [React Official Tutorial](https://react.dev/learn)
- [CSS Modules in Next.js](https://nextjs.org/docs/app/getting-started/css)
- [Axios Docs](https://axios-http.com/docs/intro)

---

#### 🟣 Diksha — ML Service + DevOps Foundation

**What to learn first** (in order):
1. What is Flask? (Python web framework — like Express but for Python)
2. What is an API endpoint? (A URL that accepts data and returns a response)
3. What is scikit-learn? (ML library — has pre-built algorithms like Linear Regression)
4. What is VADER? (A tool that scores text as positive/negative/neutral)
5. What is Docker? (Packages your app into a container that runs anywhere)

**Tasks:**

- [ ] **C1.1** Initialize Python ML service
  ```bash
  mkdir ml-service && cd ml-service
  python -m venv venv
  source venv/bin/activate
  pip install flask flask-cors scikit-learn nltk pandas numpy gunicorn
  pip freeze > requirements.txt
  ```

- [ ] **C1.2** Create `app.py` — Flask entry point
  - Set up Flask app with CORS
  - Create `GET /health` → `{ status: "ok", modelsLoaded: false }`
  - **Checkpoint**: Run `python app.py`, visit `http://localhost:8000/health`

- [ ] **C1.3** Build the **Sentiment Analyzer** (`models/sentiment_analyzer.py`)
  - Use VADER from NLTK
  - Create a function: `analyze(text) → { compound, positive, negative, neutral }`
  - Wire it to `POST /sentiment-score` endpoint
  - **Checkpoint**: Send a review text via Postman, get sentiment scores back

- [ ] **C1.4** Build the **Price Predictor** (`models/price_predictor.py`)
  - Use scikit-learn's Linear Regression
  - Start with mock training data (generate fake price history)
  - Create a function: `predict(historical_prices, target_date) → { predicted_price, confidence }`
  - Wire it to `POST /predict-price` endpoint
  - **Checkpoint**: Send historical prices via Postman, get a prediction back

- [ ] **C1.5** Write the **Dockerfile** for ML service
  - Base image: `python:3.11-slim`
  - Install dependencies, download NLTK data
  - Expose port 8000
  - **Checkpoint**: `docker build -t staysync-ml .` succeeds

- [ ] **C1.6** Create `docker-compose.yml` (at project root)
  - Services: mongo, redis (just these two for now)
  - **Checkpoint**: `docker-compose up` starts MongoDB and Redis

**📚 Resources to study:**
- [Flask Quickstart](https://flask.palletsprojects.com/en/3.0.x/quickstart/)
- [VADER Sentiment Analysis](https://github.com/cjhutto/vaderSentiment)
- [scikit-learn Linear Regression](https://scikit-learn.org/stable/modules/linear_model.html)
- [Docker Getting Started](https://docs.docker.com/get-started/)
- [Docker Compose overview](https://docs.docker.com/compose/)

---

### Phase 2: Core Features (Week 2) — "Make it work"  ⏰ Deadline: end of Week 2

> **Goal**: Real data flows through the system. Backend talks to RapidAPI, frontend shows real hotel data, ML models work on actual text.

---

#### 🔵 Nikhil — Backend Features

- [ ] **A2.1** Set up Redis connection (`config/redis.js`)
  - Use `ioredis` or `redis` npm package
  - Create `services/cacheService.js` with `get`, `set`, `delete` helpers

- [ ] **A2.2** Build the **RapidAPI integration** (`services/rapidApiService.js`)
  - Sign up for a hotel API on RapidAPI (e.g., Booking.com or Hotels.com)
  - Build platform adapters (normalize different API response shapes)
  - Implement upsert logic (merge data from multiple platforms)

- [ ] **A2.3** Build hotel routes + controller
  - `GET /api/hotels/search` — with city, checkin/checkout, filters
  - `GET /api/hotels/:id` — single hotel detail
  - Implement cache-first strategy (check Redis → miss → fetch API → cache → respond)

- [ ] **A2.4** Build bookmark routes
  - `POST /api/hotels/:id/bookmark` — add bookmark (needs auth middleware)
  - `DELETE /api/hotels/:id/bookmark` — remove bookmark
  - `GET /api/auth/me` — return user with populated bookmarks

- [ ] **A2.5** Build review routes + controller
  - Full CRUD on reviews
  - On review create: trigger sentiment analysis by calling ML service
  - Update hotel's `sentimentScore` aggregate

- [ ] **A2.6** Build ML proxy routes
  - `POST /api/ml/predict-price` — forward to Python ML service
  - `POST /api/ml/sentiment` — forward to Python ML service
  - Add Redis caching for ML responses

- [ ] **A2.7** Implement **refresh token rotation**
  - `POST /api/auth/refresh` — issue new AT+RT, revoke old RT
  - Detect token reuse (theft) and revoke entire family

- [ ] **A2.8** Add middleware layers
  - Rate limiter (`express-rate-limit`)
  - Global error handler
  - Request validation (`joi` or `express-validator`)

---

#### 🟢 Shalu — Frontend Features

- [ ] **B2.1** Set up **Axios instance** (`lib/api.js`)
  - Base URL pointing to backend (`http://localhost:5000/api`)
  - Request interceptor: attach access token to every request
  - Response interceptor: on 401 TOKEN_EXPIRED, auto-refresh and retry

- [ ] **B2.2** Build **Auth Context/Provider** (`lib/auth.js`)
  - Store user + access token in React context
  - Login, logout, register functions
  - Auto-refresh on app load

- [ ] **B2.3** Connect **Auth Pages** to real backend
  - Register form → `POST /api/auth/register`
  - Login form → `POST /api/auth/login`
  - Store token, redirect to dashboard
  - Show error messages from backend

- [ ] **B2.4** Connect **Search Page** to real backend
  - Replace mock data with `GET /api/hotels/search?city=...`
  - Loading states, error states, empty states
  - Implement filters (price range, star rating, amenities)

- [ ] **B2.5** Build **Hotel Detail Page** (`app/hotel/[id]/page.js`)
  - Hotel header (images, name, rating, best price)
  - Price comparison table (all platforms)
  - Reviews section
  - ML widgets: predicted price, sentiment gauge
  - Bookmark button

- [ ] **B2.6** Integrate **Google Maps**
  - Map view on search page showing hotel locations as pins
  - Price displayed on each pin
  - Click pin → hotel card popup
  - Requires: Google Maps API key

- [ ] **B2.7** Build **User Dashboard** (`app/dashboard/page.js`)
  - Bookmarked hotels list
  - Booking redirect history
  - "Price when you saved" vs "current price" comparison

---

#### 🟣 Diksha — ML + Integration

- [ ] **C2.1** Train the **price predictor on real data**
  - Use historical price data from MongoDB (or generate realistic synthetic data)
  - Feature engineering: day of week, month, days until checkin, holiday flags
  - Train, evaluate (RMSE, R²), save model to `data/trained_model.pkl`

- [ ] **C2.2** Improve sentiment analyzer
  - Handle edge cases: empty reviews, non-English text, very short text
  - Batch processing: accept array of reviews, return array of scores
  - Add `/bulk-sentiment` endpoint

- [ ] **C2.3** Add model loading on startup
  - Load `trained_model.pkl` at Flask app startup (not per request!)
  - Update `/health` to report `modelsLoaded: true`

- [ ] **C2.4** Write comprehensive **ML service tests** (`tests/test_predictions.py`)
  - Test sentiment analyzer with known positive/negative/neutral texts
  - Test price predictor with mock historical data
  - Test edge cases (empty input, invalid data)

- [ ] **C2.5** Expand `docker-compose.yml` — Full stack
  - Add services: backend, frontend, ml-service
  - Configure networking (all services on `staysync-network`)
  - Add health checks for all services
  - **Checkpoint**: `docker-compose up` starts the entire system

- [ ] **C2.6** Set up **environment variables**
  - Create `.env.example` with all required vars (no real secrets)
  - Document each variable in README
  - Ensure docker-compose reads from `.env`

---

### Phase 3: Polish + Integration (Week 3) — "Make it good"  ⏰ Deadline: end of Week 3

> **Goal**: Error handling, testing, UI polish, performance, documentation.

#### All Members (Nikhil + Shalu + Diksha)

- [ ] Add comprehensive error handling in your module
- [ ] Write tests (Backend: Jest, Frontend: React Testing Library, ML: pytest)
- [ ] Mobile responsiveness (Frontend)
- [ ] Loading skeletons and smooth animations (Frontend)
- [ ] API rate limiting and input validation (Backend)
- [ ] Model accuracy improvement (ML)
- [ ] Create a project demo video
- [ ] Write the README with setup instructions

---

### Phase 4: Deployment & Demo (Week 4) — "Ship it"  ⏰ FINAL DEADLINE

#### Diksha leads, Nikhil + Shalu support

- [ ] Finalize docker-compose for production
- [ ] Set up GitHub Actions CI/CD (lint → test → build → deploy)
- [ ] Deploy to a cloud platform (Railway, Render, or AWS EC2)
- [ ] Final integration testing
- [ ] Prepare project presentation/documentation

---

## 🔧 Professional Practices — Follow These from Day 1

### Git Workflow

```
main (protected — never push directly)
  │
  ├── develop (integration branch — merge PRs here)
  │     │
  │     ├── feature/A-auth-module        ← Nikhil
  │     ├── feature/B-landing-page       ← Shalu
  │     ├── feature/C-sentiment-service  ← Diksha
  │     │
  │     ├── fix/A-cors-issue             ← Bug fix branches
  │     └── ...
```

**Rules:**
1. **Never push to `main` or `develop` directly** — always make a Pull Request (PR)
2. **Branch naming**: `feature/A-short-description`, `fix/B-bug-name`
3. **At least 1 team member reviews** every PR before merging
4. **Commit messages**: Use conventional commits — `feat: add login route`, `fix: handle empty search`, `docs: update README`
5. **Pull from `develop` daily** — avoid merge conflicts by staying up to date

### Setting Up the Repo

```bash
# One person does this (then others clone)
git init
git checkout -b main
git add .
git commit -m "chore: initial project structure"

# Create develop branch
git checkout -b develop

# Push to GitHub
git remote add origin <your-github-url>
git push -u origin main
git push -u origin develop

# Protect main and develop branches in GitHub Settings → Branches → Branch Protection Rules
```

### Code Review Checklist (for PR reviews)

- [ ] Does it work? (reviewer should pull the branch and test)
- [ ] Is the code readable? (meaningful variable names, comments for complex logic)
- [ ] Are there error handling cases? (what if the API is down? what if input is wrong?)
- [ ] Are there any hardcoded values that should be in `.env`?
- [ ] Does it follow the project's code style?

### Environment Variables

Create `.env` in both `backend/` and `frontend/`:

```bash
# backend/.env
PORT=5000
MONGO_URI=mongodb://localhost:27017/staysync
REDIS_URL=redis://localhost:6379
ACCESS_TOKEN_SECRET=your-super-secret-key-at-least-32-chars
REFRESH_TOKEN_SECRET=another-secret-key-at-least-32-chars
RAPIDAPI_KEY=your-rapidapi-key-here
ML_SERVICE_URL=http://localhost:8000

# frontend/.env.local
NEXT_PUBLIC_API_URL=http://localhost:5000/api
NEXT_PUBLIC_GOOGLE_MAPS_KEY=your-google-maps-key
```

> [!CAUTION]
> **NEVER commit `.env` files to Git.** Add `.env` to `.gitignore` immediately. Share secrets via a secure channel (DMs, not group chat).

---

## 🤝 Team Coordination

### Daily Sync (10 mins max)

Every day, each member answers three questions:
1. **What did I do yesterday?**
2. **What am I doing today?**
3. **Am I blocked on anything?**

### Integration Points (Must Agree Together)

These are the "contracts" between team members — agree on these **before** coding:

| Contract | Between | What to agree on |
|----------|---------|------------------|
| API Response Shapes | Nikhil ↔ Shalu | What JSON does each endpoint return? |
| Auth Token Flow | Nikhil ↔ Shalu | How is the token sent? What happens on expiry? |
| ML Service API | Nikhil ↔ Diksha | What's the request/response format for `/predict-price` and `/sentiment-score`? |
| Docker Networking | All 3 | What hostnames and ports does each service use inside Docker? |

> [!TIP]
> **Create a shared Postman collection or an API spec (JSON file)** where you document every endpoint's request and response format. This is your team's contract.

### Tools to Use

| Tool | Purpose |
|------|---------|
| **GitHub** | Code hosting, PRs, issue tracking |
| **Postman** | Testing API endpoints |
| **MongoDB Compass** | Visual database explorer |
| **Redis Insight** | Visual Redis explorer |
| **Docker Desktop** | Running containers |
| **VS Code** | Code editor (use Live Share for pair programming) |

---

## 📍 Where to Start RIGHT NOW

1. **All 3**: Read this document top to bottom. Discuss anything unclear.
2. **All 3**: Set up your dev environment (Node 20+, Python 3.11+, MongoDB, Redis, Docker).
3. **One person**: Create the GitHub repo, push initial structure, protect branches.
4. **Each member**: Create your first feature branch and start Phase 1 tasks.
5. **Come back here** anytime you're stuck — I'll help you debug, explain concepts, or review your code.

> [!NOTE]
> **You're building this to learn.** Don't rush to copy code. When you encounter something you don't understand (middleware? JWT? GeoJSON?), spend 10-15 minutes reading about it, then implement it yourself. That's how you actually learn. Use me to clarify concepts, debug issues, and review your approach — not to generate all the code for you.

---

## 🚀 Day 0: Environment Setup Guide (All 3 Members)

Before **any** coding begins, everyone needs these tools installed. Run these on your Linux machine.

### Step 1: Install Node.js 20+ (Nikhil + Shalu)

```bash
# Install via nvm (Node Version Manager) — the professional way
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20

# Verify
node -v   # Should show v20.x.x
npm -v    # Should show 10.x.x
```

### Step 2: Install Python 3.11+ (Diksha, and nice-to-have for all)

```bash
# Most Linux distros have Python pre-installed. Check:
python3 --version

# If not 3.11+, install via deadsnakes PPA (Ubuntu/Debian):
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip

# Verify
python3.11 --version
```

### Step 3: Install MongoDB 7 (Nikhil, but all should have it)

```bash
# Option A: Install locally (Ubuntu)
# Follow: https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/

# Option B: Use Docker (easier — see Step 5)
# Option C: Use MongoDB Atlas (free cloud tier) — https://www.mongodb.com/atlas

# If installed locally, start it:
sudo systemctl start mongod
sudo systemctl enable mongod

# Verify
mongosh --eval "db.runCommand('ping')"
```

### Step 4: Install Redis (Nikhil)

```bash
# Ubuntu/Debian
sudo apt install redis-server
sudo systemctl start redis
sudo systemctl enable redis

# Verify
redis-cli ping   # Should respond: PONG
```

### Step 5: Install Docker + Docker Compose (Diksha, but all should have it)

```bash
# Install Docker Engine
# Follow official docs: https://docs.docker.com/engine/install/ubuntu/

# Or quick install:
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in, then:

# Verify
docker --version
docker compose version
```

### Step 6: Install GUI Tools (All)

| Tool | Install | Purpose |
|------|---------|---------|
| **VS Code** | `sudo snap install code --classic` | Code editor |
| **Postman** | `sudo snap install postman` | API testing |
| **MongoDB Compass** | [Download](https://www.mongodb.com/try/download/compass) | Visual DB browser |
| **Git** | `sudo apt install git` (usually pre-installed) | Version control |

### Step 7: Configure Git (All)

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main

# Set up SSH key for GitHub (so you don't type password every push)
ssh-keygen -t ed25519 -C "your.email@example.com"
cat ~/.ssh/id_ed25519.pub
# Copy the output → GitHub → Settings → SSH Keys → Add
```

---

## 🔑 RapidAPI Recommendation

> [!TIP]
> For a student/B.Tech project, use the **Booking.com API on RapidAPI** (by "Tipsters" or the official listing). It has:
> - A free tier with ~500 requests/month (enough for development)
> - Hotel search, details, reviews, and pricing data
> - Good documentation and response structure
>
> **How to get started:**
> 1. Go to [rapidapi.com](https://rapidapi.com)
> 2. Create a free account
> 3. Search for "Booking.com" or "Hotels"
> 4. Subscribe to the free tier
> 5. Copy your `X-RapidAPI-Key` — this goes in your `.env` file
>
> **Nikhil** should do this first and test the API response in Postman before writing adapter code.

---

## ✅ Ready? Here's Your Literal First Steps

| Who | Right now | Today | This week |
|-----|-----------|-------|-----------|
| **Nikhil** 🔵 | Install Node.js + MongoDB + Redis | Create GitHub repo, push initial folder structure, protect branches | Tasks A1.1 → A1.5 |
| **Shalu** 🟢 | Install Node.js | Clone repo, create `feature/B-setup` branch | Tasks B1.1 → B1.6 |
| **Diksha** 🟣 | Install Python + Docker | Clone repo, create `feature/C-setup` branch | Tasks C1.1 → C1.6 |
| **All 3** | Install VS Code + Postman + Git | Read this plan together, discuss the architecture diagram | Agree on API contracts |
