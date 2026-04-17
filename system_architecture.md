# StaySync — System Architecture Document

> **Project**: StaySync — Hotel/Room Price Aggregator  
> **Author**: System Architect  
> **Date**: April 2026  

---

## 1. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENT TIER                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                     Next.js 15 (App Router) — Port 3000                       │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐     │  │
│  │  │ Map View     │  │  List View   │  │  Auth Pages  │  │  Dashboard     │     │  │
│  │  │ (Google Maps │  │  (Filters,   │  │  (Login,     │  │  (Bookmarks,   │     │  │
│  │  │  Price Pins) │  │   Sort,      │  │   Register,  │  │   Price        │     │  │
│  │  │              │  │   Paginate)  │  │   Profile)   │  │   Forecast)    │     │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └───────┬────────┘     │  │
│  │         │                 │                  │                 │              │  │
│  │         └────────────┬────┴──────────────────┴─────────────────┘              │  │
│  │                      │  HTTP/REST (Axios/Fetch)                               │  │
│  └──────────────────────┼───────────────────────────────────────────────────────┘   │
│                         │                                                           │
└─────────────────────────┼───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              API GATEWAY TIER                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                  Node.js + Express.js — Port 5000                             │  │
│  │                                                                               │  │
│  │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                      │  │
│  │   │  Auth        │   │  Hotel       │   │  ML          │                      │  │
│  │   │  Module      │   │  Module      │   │  Module      │                      │  │
│  │   │  /api/auth/* │   │  /api/hotels │   │  /api/ml/*   │                      │  │
│  │   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                      │  │
│  │          │                  │                   │                             │  │
│  │   ┌──────┴──────────────────┴───────────────────┴──────────┐                  │  │
│  │   │          Shared Middleware Layer                       │                  │  │
│  │   │  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐   │                  │  │
│  │   │  │ JWT     │ │ Rate     │ │ Error    │ │ Request   │   │                  │  │
│  │   │  │ Verify  │ │ Limiter  │ │ Handler  │ │ Validator │   │                  │  │
│  │   │  └─────────┘ └──────────┘ └──────────┘ └───────────┘   │                  │  │
│  │   └────────────────────────────────────────────────────────┘                  │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                     │
└──────────┬──────────────────────────┬──────────────────────────┬────────────────────┘
           │                          │                          │
           ▼                          ▼                          ▼
┌──────────────────┐   ┌────────────────────┐   ┌────────────────────────────────┐
│   DATA TIER      │   │   CACHE TIER       │   │   ML TIER                      │
│                  │   │                    │   │                                │
│  ┌────────────┐  │   │  ┌──────────────┐  │   │  ┌──────────────────────────┐  │
│  │  MongoDB   │  │   │  │    Redis     │  │   │  │  Python ML Service       │  │
│  │  Port 27017│  │   │  │  Port 6379   │  │   │  │  Port 8000 (Flask)       │  │
│  │            │  │   │  │              │  │   │  │                          │  │
│  │  • users   │  │   │  │  • hotel:    │  │   │  │  • /predict-price        │  │
│  │  • hotels  │  │   │  │    search    │  │   │  │    (Linear Regression)   │  │
│  │  • reviews │  │   │  │    results   │  │   │  │                          │  │
│  │  • bookmarks│ │   │  │  • session   │  │   │  │  • /sentiment-score      │  │
│  │  • price   │  │   │  │    tokens    │  │   │  │    (VADER Analysis)      │  │
│  │    history │  │   │  │  • rate      │  │   │  │                          │  │
│  └────────────┘  │   │  │    limits    │  │   │  └──────────────────────────┘  │
│                  │   │  └──────────────┘  │   │                                │
└──────────────────┘   └────────────────────┘   └────────────────────────────────┘
                                                            │
                                                            │ HTTP
                                                            ▼
                                                ┌────────────────────────┐
                                                │   EXTERNAL APIS        │
                                                │                        │
                                                │   • RapidAPI Hotels    │
                                                │   • Google Maps API    │
                                                └────────────────────────┘
```

---

## 2. Microservices / Modules & Ownership

StaySync follows a **modular monolith** pattern (not full microservices) — all Node.js logic lives inside one Express server, organized into clean module boundaries. The Python ML service is the only separate process.

### Module Breakdown

| Module | Route Prefix | Responsibility | Key Files |
|--------|-------------|----------------|-----------|
| **Auth Module** | `/api/auth/*` | User registration, login, JWT token issue/refresh, password hashing, refresh token rotation | `routes/auth.js`, `controllers/authController.js`, `middleware/authMiddleware.js`, `models/User.js` |
| **Hotel Module** | `/api/hotels/*` | Aggregate hotel data from RapidAPI, normalize responses, search/filter, bookmarks, price history storage | `routes/hotels.js`, `controllers/hotelController.js`, `services/rapidApiService.js`, `models/Hotel.js` |
| **Review Module** | `/api/reviews/*` | CRUD on user reviews, trigger sentiment analysis via ML service, store sentiment scores | `routes/reviews.js`, `controllers/reviewController.js`, `models/Review.js` |
| **ML Module** | `/api/ml/*` | Proxy layer to Python ML service — price forecasting requests, sentiment analysis requests | `routes/ml.js`, `controllers/mlController.js`, `services/mlService.js` |
| **Cache Layer** | (internal) | Redis read-through cache with TTL, cache invalidation helpers | `services/cacheService.js`, `config/redis.js` |
| **Python ML Service** | `:8000/*` | Standalone Flask/FastAPI server running scikit-learn price prediction and VADER sentiment | `ml-service/app.py`, `ml-service/models/`, `ml-service/requirements.txt` |
| **Next.js Frontend** | `/` | SSR/CSR pages, Google Maps integration, auth state management, API consumption | `frontend/app/`, `frontend/components/`, `frontend/lib/` |

### Why "Modular Monolith" over Microservices?

> [!TIP]
> For a B.Tech project, a modular monolith gives you **the structure of microservices** (clear boundaries, separate concerns) without the **operational overhead** (service discovery, inter-service auth, distributed tracing). You can split into true microservices later if needed — the module boundaries are already clean.

---

## 3. Data Flow: RapidAPI → Node → Redis → MongoDB → Next.js

### 3.1 Search Request Flow (Sequence)

```
  Next.js Client                 Express Server              Redis              RapidAPI            MongoDB
       │                              │                        │                    │                   │
       │  GET /api/hotels/search      │                        │                    │                   │
       │  ?city=Mumbai&checkin=...     │                        │                    │                   │
       │─────────────────────────────▶│                        │                    │                   │
       │                              │                        │                    │                   │
       │                              │  CACHE CHECK           │                    │                   │
       │                              │  GET hotel:search:     │                    │                   │
       │                              │  {hash(query_params)}  │                    │                   │
       │                              │───────────────────────▶│                    │                   │
       │                              │                        │                    │                   │
       │                     ┌────────┤  HIT? ◀────────────────│                    │                   │
       │                     │ YES    │                        │                    │                   │
       │  ◀──────────────────┘        │  Return cached JSON    │                    │                   │
       │  200 OK (cached)             │                        │                    │                   │
       │                              │                        │                    │                   │
       │                     ┌────────┤  MISS?                 │                    │                   │
       │                     │ NO     │                        │                    │                   │
       │                     │        │  Fetch from RapidAPI   │                    │                   │
       │                     │        │────────────────────────┼───────────────────▶│                   │
       │                     │        │                        │                    │                   │
       │                     │        │  ◀─────────────────────┼────────────────────│                   │
       │                     │        │  Raw hotel data        │                    │                   │
       │                     │        │                        │                    │                   │
       │                     │        │  ── NORMALIZE ──       │                    │                   │
       │                     │        │  Map to unified        │                    │                   │
       │                     │        │  Hotel schema          │                    │                   │
       │                     │        │                        │                    │                   │
       │                     │        │  CACHE SET (TTL=15min) │                    │                   │
       │                     │        │───────────────────────▶│                    │                   │
       │                     │        │                        │                    │                   │
       │                     │        │  PERSIST (async)       │                    │                   │
       │                     │        │────────────────────────┼───────────────────▶│                   │
       │                     │        │                        │                    │  upsert hotels    │
       │                     │        │                        │                    │  + price_history   │
       │                     │        │                        │                    │                   │
       │  ◀──────────────────┘        │                        │                    │                   │
       │  200 OK (fresh)              │                        │                    │                   │
       │                              │                        │                    │                   │
```

### 3.2 Data Normalization

RapidAPI returns heterogeneous data. The Node backend normalizes it into a unified schema:

```javascript
// Unified Hotel Schema (what reaches the frontend)
{
  hotelId: String,           // Internal unique ID
  externalIds: {             // Original platform IDs
    booking: String,
    expedia: String,
  },
  name: String,
  location: {
    address: String,
    city: String,
    country: String,
    coordinates: {           // For Google Maps pins
      lat: Number,
      lng: Number,
    },
  },
  prices: [                  // Aggregated across platforms
    {
      platform: "Booking.com",
      pricePerNight: 4500,
      currency: "INR",
      lastUpdated: ISODate,
    },
    {
      platform: "Expedia",
      pricePerNight: 4200,
      currency: "INR",
      lastUpdated: ISODate,
    },
  ],
  bestPrice: {               // Pre-computed lowest
    platform: String,
    pricePerNight: Number,
  },
  rating: Number,
  amenities: [String],
  images: [String],
  sentimentScore: Number,    // From VADER analysis
  predictedPrice: Number,    // From ML forecasting
}
```

### 3.3 Cache Strategy

| Cache Key Pattern | TTL | Invalidation |
|---|---|---|
| `hotel:search:{hash(city+dates+filters)}` | 15 min | On new search with force-refresh |
| `hotel:detail:{hotelId}` | 30 min | On review submission |
| `hotel:prices:{hotelId}` | 10 min | Shortest TTL — prices change fast |
| `ml:price-prediction:{hotelId}` | 60 min | On new price data ingestion |
| `ml:sentiment:{hotelId}` | 24 hours | On new review submission |

---

## 4. JWT Auth Flow

### 4.1 Token Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   TOKEN PAIR                             │
│                                                         │
│   Access Token (AT)              Refresh Token (RT)     │
│   ─────────────────              ──────────────────     │
│   • Short-lived: 15 min         • Long-lived: 7 days   │
│   • Sent: Authorization          • Sent: httpOnly       │
│     header (Bearer)                 cookie               │
│   • Contains: userId,            • Contains: userId,    │
│     email, role                     tokenFamily          │
│   • Stateless validation         • Stored in MongoDB    │
│                                    (for rotation)       │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Registration Flow

```
Client                           Server                          MongoDB
  │                                │                                │
  │  POST /api/auth/register       │                                │
  │  { name, email, password }     │                                │
  │───────────────────────────────▶│                                │
  │                                │                                │
  │                                │  1. Validate input (Joi)       │
  │                                │  2. Check email not taken      │
  │                                │────────────────────────────────▶│
  │                                │  ◀─ No duplicate ──────────────│
  │                                │                                │
  │                                │  3. bcrypt.hash(password, 12)  │
  │                                │  4. Create User document       │
  │                                │────────────────────────────────▶│
  │                                │  ◀─ User saved ────────────────│
  │                                │                                │
  │                                │  5. Sign AT (15min) + RT (7d)  │
  │                                │  6. Store RT hash in DB        │
  │                                │────────────────────────────────▶│
  │                                │                                │
  │  ◀────────────────────────────│                                │
  │  201 Created                   │                                │
  │  Body: { accessToken, user }   │                                │
  │  Cookie: refreshToken (httpOnly, secure, sameSite=strict)       │
  │                                │                                │
```

### 4.3 Login Flow

```
Client                           Server                          MongoDB
  │                                │                                │
  │  POST /api/auth/login          │                                │
  │  { email, password }           │                                │
  │───────────────────────────────▶│                                │
  │                                │  1. Find user by email         │
  │                                │────────────────────────────────▶│
  │                                │  ◀─ User doc ──────────────────│
  │                                │                                │
  │                                │  2. bcrypt.compare(password,   │
  │                                │     user.hashedPassword)       │
  │                                │                                │
  │                                │  3. Invalidate ALL existing    │
  │                                │     refresh tokens (family)    │
  │                                │────────────────────────────────▶│
  │                                │                                │
  │                                │  4. Sign new AT + RT           │
  │                                │  5. Store new RT hash          │
  │                                │────────────────────────────────▶│
  │                                │                                │
  │  ◀────────────────────────────│                                │
  │  200 OK                        │                                │
  │  Body: { accessToken, user }   │                                │
  │  Cookie: refreshToken          │                                │
```

### 4.4 Refresh Token Rotation

> [!IMPORTANT]
> **Rotation** means every time a refresh token is used, it is **invalidated** and a **new pair** (AT + RT) is issued. If a previously-used RT is presented again, the entire token family is revoked — this detects token theft.

```
Client                           Server                          MongoDB
  │                                │                                │
  │  POST /api/auth/refresh        │                                │
  │  Cookie: refreshToken (old)    │                                │
  │───────────────────────────────▶│                                │
  │                                │  1. Verify RT signature        │
  │                                │  2. Find RT hash in DB         │
  │                                │────────────────────────────────▶│
  │                                │                                │
  │                         ┌──────┤  RT found & unused?            │
  │                         │ YES  │                                │
  │                         │      │  3. Mark old RT as "used"      │
  │                         │      │  4. Sign new AT + new RT       │
  │                         │      │  5. Store new RT hash          │
  │                         │      │────────────────────────────────▶│
  │  ◀──────────────────────┘      │                                │
  │  200 OK { accessToken }        │                                │
  │  Cookie: new refreshToken      │                                │
  │                                │                                │
  │                         ┌──────┤  RT already used (reuse!)      │
  │                         │THEFT │                                │
  │                         │      │  REVOKE entire token family    │
  │                         │      │────────────────────────────────▶│
  │  ◀──────────────────────┘      │                                │
  │  401 Unauthorized              │                                │
  │  (force re-login)              │                                │
```

### 4.5 Protected Route Middleware

```javascript
// middleware/authMiddleware.js
const verifyAccessToken = (req, res, next) => {
  const authHeader = req.headers.authorization;       // "Bearer <token>"
  if (!authHeader?.startsWith('Bearer '))
    return res.status(401).json({ message: 'No token provided' });

  const token = authHeader.split(' ')[1];

  try {
    const decoded = jwt.verify(token, process.env.ACCESS_TOKEN_SECRET);
    req.user = decoded;                               // { userId, email, role }
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError')
      return res.status(401).json({ message: 'Token expired', code: 'TOKEN_EXPIRED' });
    return res.status(403).json({ message: 'Invalid token' });
  }
};
```

**Frontend flow**: Next.js intercepts `TOKEN_EXPIRED`, calls `/api/auth/refresh` automatically via an Axios interceptor, retries the original request.

---

## 5. Python ML Integration

### Architecture Decision: **REST API (Flask/FastAPI micro-service)** — NOT subprocess

> [!CAUTION]
> **Never use `child_process.exec('python script.py')`** in production. It creates a new Python process per request, has no connection pooling, no error handling parity, and blocks the Node event loop while waiting.

### Why REST over Subprocess?

| Criteria | Subprocess (`child_process`) | REST Service (Flask) |
|----------|------------------------------|----------------------|
| Startup overhead | New Python process per call (~300ms) | Pre-loaded models, instant (~5ms) |
| Memory | Loads scikit-learn on every call | Models loaded once at boot |
| Error handling | Parsing stderr strings | Proper HTTP status codes + JSON |
| Scalability | Bound to Node process | Scale independently via Docker |
| Testing | Difficult to mock | Standard API mocking |
| Docker | Shared container (messy deps) | Clean separate container |

### ML Service Architecture

```
ml-service/
├── app.py                    # Flask/FastAPI entry point
├── requirements.txt          # Python dependencies
├── Dockerfile
├── models/
│   ├── price_predictor.py    # Linear Regression (scikit-learn)
│   └── sentiment_analyzer.py # VADER (nltk.sentiment)
├── data/
│   └── trained_model.pkl     # Pre-trained model (persisted)
└── tests/
    └── test_predictions.py
```

### API Endpoints

```
ML Service (Port 8000)

POST /predict-price
  Body: { hotelId, historicalPrices: [...], targetDate }
  Response: { predictedPrice: 4350, confidence: 0.82 }

POST /sentiment-score
  Body: { reviews: ["Great hotel...", "Terrible service..."] }
  Response: { averageScore: 0.65, breakdown: [...] }

GET /health
  Response: { status: "ok", modelsLoaded: true }
```

### Integration in Node Backend

```javascript
// services/mlService.js
const ML_SERVICE_URL = process.env.ML_SERVICE_URL || 'http://ml-service:8000';

async function getPricePrediction(hotelId, historicalPrices, targetDate) {
  // Check Redis cache first
  const cacheKey = `ml:price-prediction:${hotelId}:${targetDate}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Call ML service
  const { data } = await axios.post(`${ML_SERVICE_URL}/predict-price`, {
    hotelId, historicalPrices, targetDate,
  });

  // Cache result (1 hour TTL)
  await redis.setex(cacheKey, 3600, JSON.stringify(data));
  return data;
}

async function getSentimentScore(reviews) {
  const { data } = await axios.post(`${ML_SERVICE_URL}/sentiment-score`, {
    reviews: reviews.map(r => r.text),
  });
  return data;
}
```

### When ML is Triggered

```
┌──────────────────────────────────────────────────────────────────┐
│                  ML TRIGGER POINTS                               │
│                                                                  │
│  Price Prediction:                                               │
│    • When user views hotel detail page                           │
│    • Nightly cron job for popular hotels (pre-compute)           │
│    • On-demand via /api/ml/predict-price                         │
│                                                                  │
│  Sentiment Analysis:                                             │
│    • When a new review is submitted  (async, post-response)      │
│    • Batch job on initial hotel data import                      │
│    • On-demand via /api/ml/sentiment                             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Docker Compose Service Breakdown

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  # ─────────────────────────────── FRONTEND ──────────────────────────
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: production           # Multi-stage: deps → build → runtime
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:5000/api
      - NEXT_PUBLIC_GOOGLE_MAPS_KEY=${GOOGLE_MAPS_KEY}
    depends_on:
      - backend
    networks:
      - staysync-network
    restart: unless-stopped

  # ─────────────────────────────── BACKEND ───────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: production
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - MONGO_URI=mongodb://mongo:27017/staysync
      - REDIS_URL=redis://redis:6379
      - ML_SERVICE_URL=http://ml-service:8000
      - ACCESS_TOKEN_SECRET=${ACCESS_TOKEN_SECRET}
      - REFRESH_TOKEN_SECRET=${REFRESH_TOKEN_SECRET}
      - RAPIDAPI_KEY=${RAPIDAPI_KEY}
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - staysync-network
    restart: unless-stopped

  # ─────────────────────────────── ML SERVICE ────────────────────────
  ml-service:
    build:
      context: ./ml-service
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - FLASK_ENV=production
      - MODEL_PATH=/app/data/trained_model.pkl
    volumes:
      - ml-models:/app/data        # Persist trained models
    networks:
      - staysync-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ─────────────────────────────── MONGODB ───────────────────────────
  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
      - ./backend/mongo-init:/docker-entrypoint-initdb.d  # Seed scripts
    networks:
      - staysync-network
    restart: unless-stopped
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  # ─────────────────────────────── REDIS ─────────────────────────────
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - staysync-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

# ──────────────────────────────── VOLUMES ──────────────────────────────
volumes:
  mongo-data:
    driver: local
  redis-data:
    driver: local
  ml-models:
    driver: local

# ──────────────────────────────── NETWORKS ─────────────────────────────
networks:
  staysync-network:
    driver: bridge
```

### Container Map

```
┌────────────────────────────────────────────────────────────────┐
│                  staysync-network (bridge)                      │
│                                                                │
│   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐ │
│   │ frontend │   │ backend  │   │ml-service │   │  mongo   │ │
│   │ :3000    │──▶│ :5000    │──▶│ :8000     │   │  :27017  │ │
│   │ Next.js  │   │ Express  │   │ Flask     │   │  MongoDB │ │
│   └──────────┘   └────┬─────┘   └───────────┘   └────▲─────┘ │
│                       │                               │       │
│                       │         ┌──────────┐          │       │
│                       └────────▶│  redis   │──────────┘       │
│                                 │  :6379   │  (backend        │
│                                 │  Redis   │   connects       │
│                                 └──────────┘   to both)       │
│                                                               │
└────────────────────────────────────────────────────────────────┘
```

### Multi-Stage Dockerfiles

**Backend Dockerfile** (example):

```dockerfile
# ── Stage 1: Dependencies ──
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# ── Stage 2: Build ──
FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# ── Stage 3: Production ──
FROM node:20-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S nodejs && adduser -S staysync -u 1001
COPY --from=build --chown=staysync:nodejs /app .
USER staysync
EXPOSE 5000
CMD ["node", "server.js"]
```

**ML Service Dockerfile**:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Download NLTK data for VADER
RUN python -c "import nltk; nltk.download('vader_lexicon')"

COPY . .
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "app:app"]
```

---

## Recommended Project Structure

```
StaySync/
├── docker-compose.yml
├── .github/
│   └── workflows/
│       └── ci-cd.yml              # GitHub Actions pipeline
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── next.config.js
│   ├── tailwind.config.js
│   ├── app/
│   │   ├── layout.js
│   │   ├── page.js                # Landing page
│   │   ├── search/
│   │   │   └── page.js            # Map + List view
│   │   ├── hotel/
│   │   │   └── [id]/page.js       # Hotel detail
│   │   ├── auth/
│   │   │   ├── login/page.js
│   │   │   └── register/page.js
│   │   └── dashboard/
│   │       └── page.js
│   ├── components/
│   │   ├── MapView.jsx
│   │   ├── ListView.jsx
│   │   ├── PricePin.jsx
│   │   ├── HotelCard.jsx
│   │   ├── FilterSidebar.jsx
│   │   └── Navbar.jsx
│   └── lib/
│       ├── api.js                 # Axios instance + interceptors
│       └── auth.js                # Token management
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── server.js                  # Express entry point
│   ├── config/
│   │   ├── db.js                  # Mongoose connection
│   │   └── redis.js               # Redis client
│   ├── models/
│   │   ├── User.js
│   │   ├── Hotel.js
│   │   ├── Review.js
│   │   └── RefreshToken.js
│   ├── routes/
│   │   ├── auth.js
│   │   ├── hotels.js
│   │   ├── reviews.js
│   │   └── ml.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── hotelController.js
│   │   ├── reviewController.js
│   │   └── mlController.js
│   ├── middleware/
│   │   ├── authMiddleware.js
│   │   ├── rateLimiter.js
│   │   └── errorHandler.js
│   ├── services/
│   │   ├── rapidApiService.js
│   │   ├── cacheService.js
│   │   └── mlService.js
│   └── mongo-init/
│       └── init.js                # DB seed script
└── ml-service/
    ├── Dockerfile
    ├── requirements.txt
    ├── app.py
    ├── models/
    │   ├── price_predictor.py
    │   └── sentiment_analyzer.py
    ├── data/
    │   └── trained_model.pkl
    └── tests/
        └── test_predictions.py
```

---

> [!NOTE]
> **Next steps**: Once you're comfortable with the architecture, I can scaffold the actual project files — starting with the backend Express server, Mongoose models, and auth module. Let me know which component you want to build first.
