# StaySync — MongoDB Schema Design

> **Database**: MongoDB 7 · **ODM**: Mongoose 8 · **Collections**: 5

---

## 1. Mongoose Schema Definitions

### 1.1 Hotel Schema

```javascript
// models/Hotel.js
const mongoose = require('mongoose');

const priceEntrySchema = new mongoose.Schema({
  platform:      { type: String, required: true, enum: ['Booking.com', 'Expedia', 'Agoda', 'MakeMyTrip', 'Goibibo'] },
  pricePerNight: { type: Number, required: true, min: 0 },
  currency:      { type: String, default: 'INR', enum: ['INR', 'USD', 'EUR', 'GBP'] },
  roomType:      { type: String, default: 'Standard' },
  fetchedAt:     { type: Date, default: Date.now },
}, { _id: false });

const priceHistorySchema = new mongoose.Schema({
  date:   { type: Date, required: true },
  prices: [priceEntrySchema],                    // All platforms on that date
}, { _id: false });

const hotelSchema = new mongoose.Schema({
  // ── Identity ──
  name: {
    type: String,
    required: [true, 'Hotel name is required'],
    trim: true,
    index: 'text',                                // Text search index
  },
  slug: {
    type: String,
    unique: true,
    lowercase: true,
  },
  externalIds: {                                  // Map original platform IDs
    booking:    String,
    expedia:    String,
    agoda:      String,
    makemytrip: String,
    goibibo:    String,
  },

  // ── Location (GeoJSON) ──
  location: {
    type: {
      type: String,
      enum: ['Point'],
      default: 'Point',
      required: true,
    },
    coordinates: {
      type: [Number],                             // [longitude, latitude]
      required: [true, 'Coordinates are required'],
      validate: {
        validator: function (coords) {
          return coords.length === 2
            && coords[0] >= -180 && coords[0] <= 180   // lng
            && coords[1] >= -90  && coords[1] <= 90;   // lat
        },
        message: 'Invalid coordinates: [lng, lat] required',
      },
    },
    address: { type: String, trim: true },
    city:    { type: String, required: true, trim: true, index: true },
    state:   { type: String, trim: true },
    country: { type: String, required: true, trim: true },
    pincode: { type: String, trim: true },
  },

  // ── Pricing ──
  currentPrices: [priceEntrySchema],              // Latest price per platform
  bestPrice: {
    platform:      String,
    pricePerNight:  Number,
    currency:       { type: String, default: 'INR' },
  },
  priceHistory: [priceHistorySchema],             // Historical for ML training

  // ── Details ──
  starRating: {
    type: Number,
    min: 1,
    max: 5,
    required: true,
  },
  userRating: {
    type: Number,
    min: 0,
    max: 10,
    default: 0,
  },
  amenities: {
    type: [String],
    enum: [
      'wifi', 'parking', 'pool', 'gym', 'spa', 'restaurant',
      'bar', 'room-service', 'air-conditioning', 'pet-friendly',
      'breakfast-included', 'airport-shuttle', 'ev-charging',
    ],
    default: [],
  },
  images: [{ type: String }],                     // CDN URLs
  description: { type: String, trim: true },

  // ── Aggregated Sentiment ──
  sentimentScore: {
    average:     { type: Number, min: -1, max: 1, default: 0 },
    totalReviews: { type: Number, default: 0 },
    lastComputed: { type: Date },
  },

  // ── ML Prediction ──
  predictedPrice: {
    value:      Number,
    confidence: { type: Number, min: 0, max: 1 },
    targetDate: Date,
    computedAt: Date,
  },

  // ── Metadata ──
  sourcePlatforms: [{ type: String }],            // Which APIs contributed data
  lastFetchedAt:   { type: Date, default: Date.now },
  isActive:        { type: Boolean, default: true, index: true },

}, {
  timestamps: true,                               // createdAt, updatedAt
  toJSON: { virtuals: true },
  toObject: { virtuals: true },
});

// ── Indexes ──
hotelSchema.index({ location: '2dsphere' });                         // Geo queries
hotelSchema.index({ 'location.city': 1, 'bestPrice.pricePerNight': 1 }); // City + price filter
hotelSchema.index({ 'location.city': 1, starRating: -1 });           // City + star sort
hotelSchema.index({ 'bestPrice.pricePerNight': 1, starRating: -1 }); // Price + rating combo
hotelSchema.index({ amenities: 1 });                                  // Amenity filter
hotelSchema.index({ 'externalIds.booking': 1 }, { sparse: true });   // Dedup on upsert
hotelSchema.index({ 'externalIds.expedia': 1 }, { sparse: true });
hotelSchema.index({ name: 'text', description: 'text' });            // Full-text search

// ── Pre-save: compute bestPrice + slug ──
hotelSchema.pre('save', function (next) {
  // Auto-compute best price
  if (this.currentPrices?.length) {
    const lowest = this.currentPrices.reduce((min, p) =>
      p.pricePerNight < min.pricePerNight ? p : min
    );
    this.bestPrice = {
      platform: lowest.platform,
      pricePerNight: lowest.pricePerNight,
      currency: lowest.currency,
    };
  }
  // Auto-generate slug
  if (this.isModified('name')) {
    this.slug = this.name.toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/(^-|-$)/g, '');
  }
  next();
});

// ── Virtual: reviews (populated from Review collection) ──
hotelSchema.virtual('reviews', {
  ref: 'Review',
  localField: '_id',
  foreignField: 'hotel',
});

module.exports = mongoose.model('Hotel', hotelSchema);
```

---

### 1.2 User Schema

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt   = require('bcryptjs');

const bookmarkSchema = new mongoose.Schema({
  hotel:     { type: mongoose.Schema.Types.ObjectId, ref: 'Hotel', required: true },
  savedAt:   { type: Date, default: Date.now },
  notes:     { type: String, maxlength: 500 },
  priceAtSave: { type: Number },                  // Snapshot price when bookmarked
}, { _id: false });

const bookingHistorySchema = new mongoose.Schema({
  hotel:         { type: mongoose.Schema.Types.ObjectId, ref: 'Hotel', required: true },
  platform:      { type: String, required: true },
  redirectUrl:   { type: String, required: true },  // External booking link
  checkinDate:   { type: Date },
  checkoutDate:  { type: Date },
  priceAtRedirect: { type: Number },
  redirectedAt:  { type: Date, default: Date.now },
}, { _id: false });

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Name is required'],
    trim: true,
    minlength: [2, 'Name must be at least 2 characters'],
    maxlength: [50, 'Name cannot exceed 50 characters'],
  },
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\S+@\S+\.\S+$/, 'Please enter a valid email'],
  },
  password: {
    type: String,
    required: [true, 'Password is required'],
    minlength: [8, 'Password must be at least 8 characters'],
    select: false,                                // Never return in queries by default
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user',
  },
  avatar: { type: String },

  // ── Favourites / Bookmarks ──
  bookmarks: [bookmarkSchema],

  // ── Redirect History ──
  bookingHistory: [bookingHistorySchema],

  // ── Search Preferences ──
  preferences: {
    defaultCity:     { type: String, default: 'Mumbai' },
    defaultCurrency: { type: String, default: 'INR', enum: ['INR', 'USD', 'EUR', 'GBP'] },
    priceAlerts:     { type: Boolean, default: false },
  },

  // ── Auth ──
  lastLoginAt:     { type: Date },
  isActive:        { type: Boolean, default: true },

}, {
  timestamps: true,
});

// ── Indexes ──
userSchema.index({ email: 1 }, { unique: true });
userSchema.index({ 'bookmarks.hotel': 1 });          // Fast bookmark lookup
userSchema.index({ 'bookingHistory.hotel': 1 });

// ── Pre-save: hash password ──
userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 12);
  next();
});

// ── Instance method: compare password ──
userSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

// ── Strip sensitive fields from JSON ──
userSchema.methods.toJSON = function () {
  const obj = this.toObject();
  delete obj.password;
  delete obj.__v;
  return obj;
};

module.exports = mongoose.model('User', userSchema);
```

---

### 1.3 Review Schema

```javascript
// models/Review.js
const mongoose = require('mongoose');

const reviewSchema = new mongoose.Schema({
  hotel: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Hotel',
    required: true,
    index: true,
  },

  // ── Source ──
  source: {
    type: String,
    required: true,
    enum: ['booking', 'expedia', 'agoda', 'google', 'user'],
  },
  externalReviewId: { type: String },             // Dedup across API fetches
  author: {
    name:   { type: String, default: 'Anonymous' },
    userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }, // null if from API
  },

  // ── Content ──
  title:  { type: String, trim: true, maxlength: 200 },
  text:   { type: String, required: true, trim: true, maxlength: 5000 },
  rating: { type: Number, min: 1, max: 10 },
  stayDate: { type: Date },

  // ── VADER Sentiment ──
  sentiment: {
    compound: { type: Number, min: -1, max: 1 },  // Overall score
    positive: { type: Number, min: 0, max: 1 },
    negative: { type: Number, min: 0, max: 1 },
    neutral:  { type: Number, min: 0, max: 1 },
    analyzedAt: { type: Date },
  },

  // ── Metadata ──
  language:  { type: String, default: 'en' },
  isVisible: { type: Boolean, default: true },
  fetchedAt: { type: Date, default: Date.now },

}, {
  timestamps: true,
});

// ── Indexes ──
reviewSchema.index({ hotel: 1, source: 1 });                    // Filter by hotel + source
reviewSchema.index({ hotel: 1, 'sentiment.compound': -1 });     // Sort by sentiment
reviewSchema.index({ hotel: 1, createdAt: -1 });                 // Latest reviews first
reviewSchema.index({ source: 1, externalReviewId: 1 }, { unique: true, sparse: true }); // Dedup

// ── Post-save: update Hotel sentiment aggregate ──
reviewSchema.post('save', async function () {
  const Review = this.constructor;
  const Hotel  = mongoose.model('Hotel');

  const stats = await Review.aggregate([
    { $match: { hotel: this.hotel, 'sentiment.compound': { $exists: true } } },
    { $group: {
        _id: '$hotel',
        avgSentiment: { $avg: '$sentiment.compound' },
        count:        { $sum: 1 },
    }},
  ]);

  if (stats.length) {
    await Hotel.findByIdAndUpdate(this.hotel, {
      'sentimentScore.average':     Math.round(stats[0].avgSentiment * 100) / 100,
      'sentimentScore.totalReviews': stats[0].count,
      'sentimentScore.lastComputed': new Date(),
    });
  }
});

module.exports = mongoose.model('Review', reviewSchema);
```

---

### 1.4 Refresh Token Schema

```javascript
// models/RefreshToken.js
const mongoose = require('mongoose');

const refreshTokenSchema = new mongoose.Schema({
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true,
    index: true,
  },
  tokenHash: {
    type: String,
    required: true,
    unique: true,
  },
  family: {
    type: String,                                   // Token rotation family ID
    required: true,
    index: true,
  },
  isUsed: {
    type: Boolean,
    default: false,
  },
  isRevoked: {
    type: Boolean,
    default: false,
  },
  expiresAt: {
    type: Date,
    required: true,
    index: { expireAfterSeconds: 0 },              // MongoDB TTL — auto-delete expired
  },
  userAgent: String,
  ipAddress: String,
}, {
  timestamps: true,
});

// ── Indexes ──
refreshTokenSchema.index({ user: 1, family: 1 });
refreshTokenSchema.index({ user: 1, isRevoked: false });

// ── Static: revoke entire family (theft detection) ──
refreshTokenSchema.statics.revokeFamily = async function (family) {
  return this.updateMany({ family }, { isRevoked: true });
};

// ── Static: revoke all for a user (logout all devices) ──
refreshTokenSchema.statics.revokeAllForUser = async function (userId) {
  return this.updateMany({ user: userId }, { isRevoked: true });
};

module.exports = mongoose.model('RefreshToken', refreshTokenSchema);
```

---

### 1.5 Cache Metadata Schema

```javascript
// models/CacheMeta.js
const mongoose = require('mongoose');

const cacheMetaSchema = new mongoose.Schema({
  key: {
    type: String,
    required: true,
    unique: true,                                  // Redis key mirror
  },
  queryType: {
    type: String,
    required: true,
    enum: ['hotel-search', 'hotel-detail', 'price-fetch', 'ml-prediction', 'ml-sentiment'],
  },
  queryParams: {
    type: mongoose.Schema.Types.Mixed,             // Original search params
  },
  source: {
    type: String,                                  // Which RapidAPI source
  },

  // ── TTL Tracking ──
  cachedAt:  { type: Date, default: Date.now },
  expiresAt: {
    type: Date,
    required: true,
    index: { expireAfterSeconds: 0 },              // Auto-purge old records
  },
  ttlSeconds: { type: Number },

  // ── Stats ──
  hitCount:  { type: Number, default: 0 },
  lastHitAt: { type: Date },
  sizeBytes: { type: Number },                     // Approximate cached payload size

}, {
  timestamps: true,
});

// ── Indexes ──
cacheMetaSchema.index({ queryType: 1, cachedAt: -1 });
cacheMetaSchema.index({ source: 1 });

// ── Static: record a cache hit ──
cacheMetaSchema.statics.recordHit = async function (key) {
  return this.findOneAndUpdate(
    { key },
    { $inc: { hitCount: 1 }, lastHitAt: new Date() },
  );
};

module.exports = mongoose.model('CacheMeta', cacheMetaSchema);
```

---

## 2. Compound Index Strategy

### Index Matrix

| Collection | Index | Type | Purpose |
|---|---|---|---|
| `hotels` | `{ location: '2dsphere' }` | Geo | `$near`, `$geoWithin` for Map View |
| `hotels` | `{ 'location.city': 1, 'bestPrice.pricePerNight': 1 }` | Compound | City filter + price sort (**most frequent query**) |
| `hotels` | `{ 'location.city': 1, starRating: -1 }` | Compound | City + star rating sort |
| `hotels` | `{ 'bestPrice.pricePerNight': 1, starRating: -1 }` | Compound | Global price+rating filter |
| `hotels` | `{ amenities: 1 }` | Multikey | `$in` / `$all` amenity filters |
| `hotels` | `{ name: 'text', description: 'text' }` | Text | Full-text search |
| `reviews` | `{ hotel: 1, 'sentiment.compound': -1 }` | Compound | Hotel reviews sorted by sentiment |
| `reviews` | `{ source: 1, externalReviewId: 1 }` | Compound, unique, sparse | API review deduplication |
| `refreshtokens` | `{ expiresAt: 1 }` | TTL | Auto-delete expired tokens |
| `cachemetas` | `{ expiresAt: 1 }` | TTL | Auto-delete stale cache metadata |

### Geo + Price Query Example

```javascript
// "Hotels near me under ₹5000"
Hotel.find({
  location: {
    $near: {
      $geometry: { type: 'Point', coordinates: [72.8777, 19.0760] }, // Mumbai
      $maxDistance: 15000,       // 15 km radius
    },
  },
  'bestPrice.pricePerNight': { $lte: 5000 },
  isActive: true,
}).sort({ 'bestPrice.pricePerNight': 1 });
```

> [!IMPORTANT]
> MongoDB cannot combine a `2dsphere` index with other fields in a single compound index for `$near` queries. The query planner uses the geo index first, then filters the results in-memory for price. For high cardinality, add a post-geo `$match` in an aggregation pipeline instead.

```javascript
// Aggregation alternative for large datasets
Hotel.aggregate([
  {
    $geoNear: {
      near: { type: 'Point', coordinates: [72.8777, 19.0760] },
      distanceField: 'distance',
      maxDistance: 15000,
      query: { isActive: true },         // Pre-filter
      spherical: true,
    },
  },
  { $match: { 'bestPrice.pricePerNight': { $lte: 5000 } } },
  { $sort:  { 'bestPrice.pricePerNight': 1 } },
  { $limit: 50 },
]);
```

---

## 3. Normalisation Strategy

### The Problem

Each RapidAPI source returns a different shape:

```
Booking API:          Expedia API:          Agoda API:
{                     {                     {
  hotel_name: "...",    property_name: "...", hotelName: "...",
  latitude: 19.07,     geo: {                loc: {
  longitude: 72.87,      lat: 19.07,           latitude: 19.07,
  price: {                lng: 72.87            longitude: 72.87
    amount: 4500        },                    },
  },                    rate_per_night: {     cost: 4800
  star_count: 4           value: 4200        starCategory: "4-star"
}                       }                   }
                        stars: 4
```

### The Solution: Platform Adapters

```
RapidAPI ──▶ Platform Adapter ──▶ Unified Hotel Shape ──▶ MongoDB
```

```javascript
// services/adapters/bookingAdapter.js
function normalizeBooking(raw) {
  return {
    externalId:  raw.hotel_id,
    name:        raw.hotel_name,
    coordinates: [raw.longitude, raw.latitude],  // GeoJSON = [lng, lat]
    city:        raw.city_name,
    country:     raw.country,
    address:     raw.address,
    platform:    'Booking.com',
    price:       raw.price?.amount ?? null,
    currency:    raw.price?.currency ?? 'INR',
    starRating:  raw.star_count,
    amenities:   mapBookingAmenities(raw.facilities),
    images:      raw.photos?.map(p => p.url) ?? [],
    rating:      raw.review_score,
  };
}

// services/adapters/expediaAdapter.js
function normalizeExpedia(raw) {
  return {
    externalId:  raw.property_id,
    name:        raw.property_name,
    coordinates: [raw.geo.lng, raw.geo.lat],
    city:        raw.location?.city,
    country:     raw.location?.country,
    address:     raw.location?.address_line,
    platform:    'Expedia',
    price:       raw.rate_per_night?.value ?? null,
    currency:    raw.rate_per_night?.currency ?? 'INR',
    starRating:  raw.stars,
    amenities:   mapExpediaAmenities(raw.amenities),
    images:      raw.images?.map(i => i.highRes) ?? [],
    rating:      raw.guest_rating,
  };
}

// services/adapters/index.js — Factory
const adapters = {
  booking: normalizeBooking,
  expedia: normalizeExpedia,
  agoda:   normalizeAgoda,
};

function normalize(platform, rawData) {
  const adapter = adapters[platform];
  if (!adapter) throw new Error(`No adapter for platform: ${platform}`);
  return adapter(rawData);
}
```

### Upsert Logic (Merge Multi-Platform Data)

```javascript
// services/rapidApiService.js
async function upsertHotel(normalized) {
  const filter = {};
  filter[`externalIds.${normalized.platform.toLowerCase()}`] = normalized.externalId;

  const update = {
    $set: {
      name: normalized.name,
      'location.type': 'Point',
      'location.coordinates': normalized.coordinates,
      'location.city': normalized.city,
      'location.country': normalized.country,
      'location.address': normalized.address,
      [`externalIds.${normalized.platform.toLowerCase()}`]: normalized.externalId,
      starRating: normalized.starRating,
      lastFetchedAt: new Date(),
    },
    $addToSet: {
      sourcePlatforms: normalized.platform,
      amenities: { $each: normalized.amenities },
      images: { $each: normalized.images },
    },
  };

  // Upsert: create if new, merge if exists
  const hotel = await Hotel.findOneAndUpdate(filter, update, {
    upsert: true,
    new: true,
    setDefaultsOnInsert: true,
  });

  // Update currentPrices array — replace entry for this platform
  const priceIdx = hotel.currentPrices.findIndex(
    p => p.platform === normalized.platform
  );
  if (priceIdx >= 0) {
    hotel.currentPrices[priceIdx].pricePerNight = normalized.price;
    hotel.currentPrices[priceIdx].fetchedAt = new Date();
  } else {
    hotel.currentPrices.push({
      platform: normalized.platform,
      pricePerNight: normalized.price,
      currency: normalized.currency,
      fetchedAt: new Date(),
    });
  }

  await hotel.save();  // Triggers pre-save → bestPrice auto-compute
  return hotel;
}
```

---

## 4. Denormalisation Decisions

| What's Denormalised | Where | Why | Trade-off |
|---|---|---|---|
| **`bestPrice`** on Hotel | `hotels.bestPrice` | Avoids computing `Math.min()` across `currentPrices[]` on every search query — reads >>> writes | Updated on every price fetch via pre-save hook |
| **`sentimentScore`** on Hotel | `hotels.sentimentScore` | Avoids `$lookup` + aggregation on every hotel card render | Updated via post-save hook on Review |
| **`predictedPrice`** on Hotel | `hotels.predictedPrice` | Cached ML result — avoids hitting Python service per request | Stale for up to 1 hour (acceptable for forecasts) |
| **`priceAtSave`** on User.bookmarks | `users.bookmarks.priceAtSave` | Shows "price when you saved it" vs current — useful UX | Snapshot, never updated |
| **`priceAtRedirect`** on User.bookingHistory | `users.bookingHistory.priceAtRedirect` | Audit trail of what price the user saw before redirect | Snapshot, never updated |
| **`author.name`** on Review | `reviews.author.name` | Avoids `$lookup` to User on every review render | Could become stale if user renames (acceptable) |

### What's NOT Denormalised (and Why)

| Data | Stored As | Why Not Denormalise |
|---|---|---|
| **Full review list** | Separate `reviews` collection with `hotel` ref | Unbounded growth — embedding would hit 16MB doc limit |
| **User's full profile** in Review | Only `author.userId` + `author.name` | Privacy + size, plus user data changes |
| **Hotel details** in User bookmarks | Only `ObjectId` ref | Hotel data changes frequently (prices), stale embeds would mislead |

---

## Entity Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ REVIEW : "writes"
    USER ||--o{ REFRESH_TOKEN : "owns"
    USER {
        ObjectId _id
        string name
        string email
        string password
        array bookmarks
        array bookingHistory
    }
    HOTEL ||--o{ REVIEW : "has"
    HOTEL {
        ObjectId _id
        string name
        GeoJSON location
        array currentPrices
        object bestPrice
        array priceHistory
        number starRating
        object sentimentScore
    }
    REVIEW {
        ObjectId _id
        ObjectId hotel FK
        string source
        string text
        object sentiment
    }
    REFRESH_TOKEN {
        ObjectId _id
        ObjectId user FK
        string tokenHash
        string family
        Date expiresAt
    }
    CACHE_META {
        ObjectId _id
        string key
        string queryType
        Date expiresAt
    }
```
