# ☕ CoffeeTracker - Project Specification

**A learning project for mastering microservices, event-driven architecture, and DevOps**

---

## Table of Contents

1. [Project Overview](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#project-overview)
2. [Architecture](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#architecture)
3. [Coffee Service](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#coffee-service)
4. [Analytics Service](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#analytics-service)
5. [Database Design](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#database-design)
6. [Event-Driven Flow](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#event-driven-flow)
7. [Real-Time Features](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#real-time-features)
8. [Caching Strategy](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#caching-strategy)
9. [SKIP LOCK Implementation](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#skip-lock-implementation)
10. [Frontend Features](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#frontend-features)
11. [Tech Stack](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#tech-stack)
12. [Implementation Timeline](https://claude.ai/chat/fc94d3ab-85a2-4397-ba18-ada00864f775#implementation-timeline)

---

## Project Overview

### **What is CoffeeTracker?**

A personal coffee tracking application that helps you:

- Track daily caffeine intake against a customizable capacity threshold
- Monitor coffee spending against a monthly budget
- Identify which coffee shops and types keep you most productive
- Log impressions and taste ratings for different coffee shops
- Receive real-time alerts when approaching caffeine limits

### **Why This Project?**

✅ **Microservices**: 2 independent services with clear responsibilities  
✅ **Event-Driven**: Real-time event publishing and consumption  
✅ **Caching**: Multi-layer Redis strategy for performance  
✅ **SKIP LOCK**: Practical concurrent database updates  
✅ **DevOps**: Docker, containers, message queue orchestration  
✅ **Frontend**: Clean Next.js UI with real-time updates  
✅ **Real Problem**: You actually drink coffee and want insights!  
✅ **Portfolio**: Demonstrates distributed systems thinking

### **Key Features**

- Log coffee: shop, type, price, quality, feelings (before/after), productivity duration
- Adjustable caffeine capacity threshold (personalized limit)
- Budget tracking (monthly spending limit)
- Shop comparison: visit frequency, quality/price ratio, productivity scores
- Productivity insights: which shops/types keep you focused longest
- Impression tracking: quality ratings and consistent experiences per shop
- Real-time alerts: caffeine warnings, budget warnings
- Responsive Next.js dashboard

---

## Architecture

### **System Overview**

```
┌─────────────────────────────────────────┐
│        Next.js Frontend                 │
│   (Coffee logging + dashboards)         │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
    ↓                         ↓
┌──────────────────┐   ┌──────────────────┐
│  COFFEE SERVICE  │   │ ANALYTICS SERVICE│
│  (Golang)        │   │  (Python/Node)   │
├──────────────────┤   ├──────────────────┤
│ - Coffee logs    │   │ - Event consumer │
│ - Shops/ratings  │   │ - Aggregations   │
│ - User prefs     │   │ - Insights       │
│ - Auth           │   │ - Real-time      │
└────────┬─────────┘   │   alerts         │
         │             └────────┬─────────┘
         │ CoffeeLogged event   │
         │ (RabbitMQ)           │
         └──────────────────────┘
                 │
         ┌───────┴────────┐
         │                │
         ↓                ↓
    PostgreSQL        Redis Cache
    (All data)    (Hot data, aggregations)
```

### **Service Communication**

```
1. Synchronous: Frontend → Coffee Service (REST/HTTP)
2. Asynchronous: Coffee Service → Analytics Service (RabbitMQ events)
3. Synchronous: Frontend → Analytics Service (REST/HTTP for reads)
4. Real-time: Analytics Service → Frontend (WebSocket or polling)
```

---

## Coffee Service

### **Overview**

The source of truth service. Handles all user data, authentication, coffee log CRUD operations, and shop management. Publishes events when coffee is logged.

### **Responsibilities**

- User authentication and authorization
- User preference management (caffeine capacity, budget limits)
- Coffee shop CRUD operations
- Coffee log creation, reading, updating, deletion
- Event publishing to message queue

### **Technology**

- **Language**: Golang
- **Framework**: Gin or Chi
- **Database**: PostgreSQL (main schema)
- **Message Queue**: RabbitMQ publisher

### **API Endpoints**

#### Authentication

```
POST   /api/v1/auth/register
       Body: { email, password }
       Returns: { user_id, token }

POST   /api/v1/auth/login
       Body: { email, password }
       Returns: { token }

POST   /api/v1/auth/logout
       Headers: Authorization: Bearer {token}
```

#### Coffee Logs

```
POST   /api/v1/logs
       Headers: Authorization: Bearer {token}
       Body: {
         shop_id: int,
         coffee_type: string,           // "espresso", "latte", "cold_brew", etc.
         price_usd: float,
         quality_rating: int,            // 1-5
         before_feeling: string,         // "alert", "focused", "sleepy", etc.
         after_feeling: string,
         felt_better_after_mins: int,    // How long effect lasted
         caffeine_mg: int
       }
       Returns: { log_id, created_at }

GET    /api/v1/logs
       Headers: Authorization: Bearer {token}
       Query: ?limit=50&offset=0&shop_id=1&from_date=2024-04-01
       Returns: [{ id, shop_id, coffee_type, price, ... }, ...]

GET    /api/v1/logs/{id}
       Headers: Authorization: Bearer {token}
       Returns: { id, shop_id, ... full log details ... }

PUT    /api/v1/logs/{id}
       Headers: Authorization: Bearer {token}
       Body: { coffee_type, price, quality_rating, ... }
       Returns: { id, ... updated log ... }

DELETE /api/v1/logs/{id}
       Headers: Authorization: Bearer {token}
       Returns: { success: true }
```

#### Coffee Shops

```
GET    /api/v1/shops
       Headers: Authorization: Bearer {token}
       Query: ?limit=50&offset=0&name=blue
       Returns: [{ id, name, location, created_by, created_at }, ...]

GET    /api/v1/shops/{id}
       Headers: Authorization: Bearer {token}
       Returns: { id, name, location, created_by, created_at }

POST   /api/v1/shops
       Headers: Authorization: Bearer {token}
       Body: { name: string, location: string }
       Returns: { id, name, location, created_at }

PUT    /api/v1/shops/{id}
       Headers: Authorization: Bearer {token}
       Body: { name: string, location: string }
       Returns: { id, name, location, updated_at }

DELETE /api/v1/shops/{id}
       Headers: Authorization: Bearer {token}
       Returns: { success: true }
```

#### User Preferences

```
GET    /api/v1/user/preferences
       Headers: Authorization: Bearer {token}
       Returns: {
         user_id: int,
         caffeine_capacity_mg: int,      // e.g., 400
         monthly_budget_usd: float,      // e.g., 50.00
         alert_enabled: boolean,
         created_at, updated_at
       }

PUT    /api/v1/user/preferences
       Headers: Authorization: Bearer {token}
       Body: {
         caffeine_capacity_mg: int,
         monthly_budget_usd: float,
         alert_enabled: boolean
       }
       Returns: { ... updated preferences ... }
```

### **Database Schema**

```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  hashed_password VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User preferences (thresholds and settings)
CREATE TABLE user_preferences (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  caffeine_capacity_mg INTEGER DEFAULT 400,
  monthly_budget_usd DECIMAL(10, 2) DEFAULT 50.00,
  alert_enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Coffee shops
CREATE TABLE coffee_shops (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  location VARCHAR(255),
  created_by INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Coffee logs (source of truth)
CREATE TABLE coffee_logs (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  shop_id INTEGER NOT NULL REFERENCES coffee_shops(id),
  coffee_type VARCHAR(100) NOT NULL,           -- "espresso", "latte", "cold_brew"
  price_usd DECIMAL(10, 2) NOT NULL,
  quality_rating INTEGER NOT NULL CHECK (quality_rating >= 1 AND quality_rating <= 5),
  before_feeling VARCHAR(100),                 -- "alert", "focused", "sleepy", etc.
  after_feeling VARCHAR(100),
  felt_better_after_mins INTEGER,              -- Duration of effect
  caffeine_mg INTEGER NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_coffee_logs_user_date 
  ON coffee_logs(user_id, created_at DESC);

CREATE INDEX idx_coffee_logs_shop 
  ON coffee_logs(shop_id);

CREATE INDEX idx_coffee_logs_user_type 
  ON coffee_logs(user_id, coffee_type);

CREATE INDEX idx_coffee_shops_user 
  ON coffee_shops(created_by);
```

### **Event Publishing**

When a coffee log is successfully created, publish to RabbitMQ:

```go
// Event: CoffeeLogged
type CoffeeLoggedEvent struct {
  LogID          int       `json:"log_id"`
  UserID         int       `json:"user_id"`
  ShopID         int       `json:"shop_id"`
  CoffeeType     string    `json:"coffee_type"`
  Price          float64   `json:"price"`
  Quality        int       `json:"quality"`
  CaffeineMg     int       `json:"caffeine_mg"`
  BeforeFeeling  string    `json:"before_feeling"`
  AfterFeeling   string    `json:"after_feeling"`
  FocusDuration  int       `json:"focus_duration_mins"`
  Timestamp      time.Time `json:"timestamp"`
}

// RabbitMQ exchange: "coffee.events"
// Routing key: "coffee.logged"
```

---

## Analytics Service

### **Overview**

Consumes `CoffeeLogged` events in real-time and calculates insights, tracks productivity, evaluates alerts, and serves analytics endpoints to the frontend.

### **Responsibilities**

- Real-time event consumption from RabbitMQ
- Daily caffeine/spending aggregation
- Productivity metrics calculation (focus duration per shop/type)
- Impression tracking (quality ratings, before/after feelings)
- Shop comparison metrics (visit count, quality/price ratio)
- Alert evaluation and storage
- Real-time analytics endpoint serving
- Cache management and invalidation

### **Technology**

- **Language**: Python (recommended) or Node.js
- **Framework**: FastAPI/Flask (Python) or Express.js (Node.js)
- **Message Queue**: RabbitMQ consumer
- **Cache**: Redis
- **Database**: PostgreSQL (analytics schema)

### **API Endpoints**

#### Today's Summary (Real-time)

```
GET    /api/v1/analytics/today
       Headers: Authorization: Bearer {token}
       Returns: {
         caffeine: {
           total_mg: 95,
           capacity_mg: 400,
           remaining_mg: 305,
           percentage: 24,
           alert: null
         },
         budget: {
           spent_usd: 5.50,
           monthly_limit_usd: 50,
           remaining_usd: 44.50,
           percentage: 11,
           alert: null
         },
         coffee_count: 1,
         logs_today: [
           {
             id: 1,
             shop: "Blue Bottle",
             coffee_type: "espresso",
             price: 5.50,
             quality_rating: 4.5,
             before_feeling: "alert",
             after_feeling: "focused",
             felt_better_after_mins: 240,
             caffeine_mg: 95,
             created_at: "2024-04-23T10:30:00Z"
           }
         ]
       }
```

#### Monthly Summary

```
GET    /api/v1/analytics/monthly
       Headers: Authorization: Bearer {token}
       Query: ?month=2024-04
       Returns: {
         period: "2024-04",
         total_spent_usd: 42.30,
         monthly_budget_usd: 50,
         remaining_budget_usd: 7.70,
         avg_daily_caffeine_mg: 185,
         days_over_capacity: 3,
         total_logs: 28,
         total_visits: 28,
         unique_shops: 4,
         most_visited_shop: {
           shop_id: 1,
           name: "Blue Bottle",
           visit_count: 8
         },
         average_quality_rating: 4.2,
         trends: {
           spending_trend: "stable",
           caffeine_trend: "increasing"
         }
       }
```

#### Shop Comparison

```
GET    /api/v1/analytics/shops/comparison
       Headers: Authorization: Bearer {token}
       Returns: {
         shops: [
           {
             shop_id: 1,
             name: "Blue Bottle",
             visits: 8,
             total_spent_usd: 42.00,
             avg_price_usd: 5.25,
             avg_quality_rating: 4.5,
             quality_score_out_of_5: 4.5,
             price_per_quality: 1.17,
             avg_focus_duration_mins: 240,
             avg_caffeine_per_visit: 95,
             most_common_type: "espresso",
             badges: ["best_bang_for_buck", "highest_quality"],
             impressions: {
               consistent: true,
               most_common_before: "alert",
               most_common_after: "focused"
             }
           },
           {
             shop_id: 2,
             name: "Local Cafe",
             visits: 4,
             total_spent_usd: 14.00,
             avg_price_usd: 3.50,
             avg_quality_rating: 3.2,
             quality_score_out_of_5: 3.2,
             price_per_quality: 1.09,
             avg_focus_duration_mins: 120,
             avg_caffeine_per_visit: 85,
             most_common_type: "latte",
             badges: ["cheapest"],
             impressions: {
               consistent: false,
               most_common_before: "tired",
               most_common_after: "energized"
             }
           }
         ]
       }
```

#### Productivity Insights

```
GET    /api/v1/analytics/productivity
       Headers: Authorization: Bearer {token}
       Returns: {
         best_coffee_type: "espresso",
         best_shop: {
           shop_id: 1,
           name: "Blue Bottle",
           avg_focus_mins: 240,
           productivity_score: 9.2
         },
         by_coffee_type: [
           {
             type: "espresso",
             avg_focus_mins: 240,
             sample_count: 8,
             productivity_score: 9.2
           },
           {
             type: "latte",
             avg_focus_mins: 180,
             sample_count: 5,
             productivity_score: 7.8
           },
           {
             type: "cold_brew",
             avg_focus_mins: 300,
             sample_count: 2,
             productivity_score: 10.0
           }
         ],
         by_shop: [
           {
             shop_id: 1,
             name: "Blue Bottle",
             avg_focus_mins: 240,
             productivity_score: 9.2,
             sample_count: 8
           }
         ],
         insights: [
           "You're most productive with espresso at Blue Bottle (4h focus)",
           "Cold brew has the best focus time but you rarely get it (only 2 times)",
           "Local Cafe coffee is cheaper but only keeps you focused 2h",
           "Consider switching to espresso for better productivity"
         ]
       }
```

#### Impressions & Taste

```
GET    /api/v1/analytics/impressions
       Headers: Authorization: Bearer {token}
       Returns: {
         by_shop: {
           "Blue Bottle": {
             avg_quality: 4.5,
             quality_consistency: "high",
             visit_count: 8,
             common_feelings_before: [
               { feeling: "alert", count: 5 },
               { feeling: "focused", count: 3 }
             ],
             common_feelings_after: [
               { feeling: "focused", count: 6 },
               { feeling: "energized", count: 2 }
             ],
             quality_ratings: {
               "5_stars": 6,
               "4_stars": 2,
               "3_stars": 0
             }
           },
           "Local Cafe": {
             avg_quality: 3.2,
             quality_consistency: "low",
             visit_count: 4,
             common_feelings_before: [
               { feeling: "tired", count: 2 },
               { feeling: "alert", count: 2 }
             ],
             common_feelings_after: [
               { feeling: "energized", count: 3 },
               { feeling: "jittery", count: 1 }
             ],
             quality_ratings: {
               "5_stars": 0,
               "4_stars": 1,
               "3_stars": 2,
               "2_stars": 1
             }
           }
         },
         by_coffee_type: {
           "espresso": {
             avg_quality: 4.3,
             sample_count: 8,
             quality_consistency: "high"
           },
           "latte": {
             avg_quality: 3.8,
             sample_count: 5,
             quality_consistency: "medium"
           },
           "cold_brew": {
             avg_quality: 4.5,
             sample_count: 2,
             quality_consistency: "high"
           }
         }
       }
```

#### Active Alerts

```
GET    /api/v1/analytics/alerts
       Headers: Authorization: Bearer {token}
       Returns: {
         active_alerts: [
           {
             id: 1,
             type: "caffeine_warning",
             severity: "warning",
             title: "High caffeine intake",
             message: "You've had 320mg caffeine. 80mg remaining before your capacity of 400mg.",
             recommendation: "Consider switching to decaf for your next coffee",
             triggered_at: "2024-04-23T14:30:00Z"
           }
         ],
         historical_alerts: [
           {
             date: "2024-04-20",
             type: "over_capacity",
             caffeine_mg: 450,
             capacity_mg: 400
           },
           {
             date: "2024-04-15",
             type: "over_budget",
             spent_usd: 52.30,
             budget_usd: 50.00
           }
         ]
       }
```

### **Database Schema**

```sql
-- Daily aggregations (updated in real-time per event)
CREATE TABLE daily_stats (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  total_caffeine_mg INTEGER DEFAULT 0,
  total_spent_usd DECIMAL(10, 2) DEFAULT 0,
  log_count INTEGER DEFAULT 0,
  exceeded_caffeine_capacity BOOLEAN DEFAULT false,
  exceeded_budget BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, date)
);

-- Shop metrics per user
CREATE TABLE shop_metrics (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  shop_id INTEGER NOT NULL REFERENCES coffee_shops(id),
  visit_count INTEGER DEFAULT 0,
  total_spent_usd DECIMAL(10, 2) DEFAULT 0,
  avg_quality_rating DECIMAL(3, 2) DEFAULT 0,
  avg_price DECIMAL(10, 2) DEFAULT 0,
  price_per_quality DECIMAL(5, 2) DEFAULT 0,
  avg_caffeine_per_visit DECIMAL(6, 2) DEFAULT 0,
  most_common_coffee_type VARCHAR(100),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, shop_id)
);

-- Productivity tracking per shop/type
CREATE TABLE productivity_records (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  shop_id INTEGER REFERENCES coffee_shops(id),
  coffee_type VARCHAR(100),
  avg_focus_duration_mins INTEGER DEFAULT 0,
  sample_count INTEGER DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, shop_id, coffee_type)
);

-- Impression tracking
CREATE TABLE impression_records (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  shop_id INTEGER NOT NULL REFERENCES coffee_shops(id),
  coffee_type VARCHAR(100),
  total_ratings INTEGER DEFAULT 0,
  total_quality_sum INTEGER DEFAULT 0,
  avg_quality DECIMAL(3, 2) DEFAULT 0,
  most_common_before_feeling VARCHAR(100),
  most_common_after_feeling VARCHAR(100),
  feeling_before_counts JSONB,               -- {"alert": 5, "focused": 3, ...}
  feeling_after_counts JSONB,                -- {"focused": 6, "energized": 2, ...}
  quality_distribution JSONB,                -- {"5": 6, "4": 2, "3": 0, ...}
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, shop_id, coffee_type)
);

-- Alert history (append-only)
CREATE TABLE alert_history (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  date DATE NOT NULL,
  alert_type VARCHAR(100) NOT NULL,          -- "over_capacity", "over_budget"
  caffeine_mg INTEGER,
  spent_usd DECIMAL(10, 2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for SKIP LOCK queries
CREATE INDEX idx_daily_stats_user_date 
  ON daily_stats(user_id, date);

CREATE INDEX idx_shop_metrics_user_shop 
  ON shop_metrics(user_id, shop_id);

CREATE INDEX idx_productivity_user_type 
  ON productivity_records(user_id, coffee_type);

CREATE INDEX idx_impression_user_shop 
  ON impression_records(user_id, shop_id);

CREATE INDEX idx_alert_history_user_date 
  ON alert_history(user_id, date);
```

### **Event Consumption**

The Analytics Service consumes `CoffeeLogged` events from RabbitMQ and processes them in real-time:

```python
# Pseudocode for event processing

def process_coffee_logged_event(event: CoffeeLoggedEvent):
    user_id = event.user_id
    shop_id = event.shop_id
    date = event.timestamp.date()
    
    # 1. Update daily_stats (with SKIP LOCK for concurrency)
    update_daily_stats(user_id, date, event)
    
    # 2. Update shop_metrics
    update_shop_metrics(user_id, shop_id, event)
    
    # 3. Update productivity_records
    update_productivity(user_id, shop_id, event.coffee_type, event)
    
    # 4. Update impression_records
    update_impressions(user_id, shop_id, event.coffee_type, event)
    
    # 5. Evaluate alerts
    check_and_create_alerts(user_id, date, event)
    
    # 6. Invalidate Redis cache
    invalidate_user_cache(user_id, date)
    
    # 7. Notify frontend via WebSocket
    notify_frontend(user_id, "daily_stats_updated", get_today_stats(user_id))
```

---

## Database Design

### **Complete Schema Overview**

#### Coffee Service Database

```
users
├── id (PK)
├── email (UNIQUE)
├── hashed_password
├── created_at
└── updated_at

user_preferences
├── id (PK)
├── user_id (FK, UNIQUE)
├── caffeine_capacity_mg
├── monthly_budget_usd
├── alert_enabled
├── created_at
└── updated_at

coffee_shops
├── id (PK)
├── name
├── location
├── created_by (FK)
├── created_at
└── updated_at

coffee_logs (SOURCE OF TRUTH)
├── id (PK)
├── user_id (FK)
├── shop_id (FK)
├── coffee_type
├── price_usd
├── quality_rating
├── before_feeling
├── after_feeling
├── felt_better_after_mins
├── caffeine_mg
├── created_at
└── updated_at
```

#### Analytics Service Database

```
daily_stats (DENORMALIZED)
├── id (PK)
├── user_id (FK)
├── date
├── total_caffeine_mg
├── total_spent_usd
├── log_count
├── exceeded_caffeine_capacity
├── exceeded_budget
├── created_at
└── updated_at

shop_metrics (DENORMALIZED)
├── id (PK)
├── user_id (FK)
├── shop_id (FK)
├── visit_count
├── total_spent_usd
├── avg_quality_rating
├── avg_price
├── price_per_quality
├── avg_caffeine_per_visit
├── most_common_coffee_type
└── updated_at

productivity_records (DENORMALIZED)
├── id (PK)
├── user_id (FK)
├── shop_id (FK)
├── coffee_type
├── avg_focus_duration_mins
├── sample_count
└── updated_at

impression_records (DENORMALIZED)
├── id (PK)
├── user_id (FK)
├── shop_id (FK)
├── coffee_type
├── total_ratings
├── avg_quality
├── most_common_before_feeling
├── most_common_after_feeling
├── feeling_before_counts (JSONB)
├── feeling_after_counts (JSONB)
├── quality_distribution (JSONB)
└── updated_at

alert_history (APPEND-ONLY LOG)
├── id (PK)
├── user_id (FK)
├── date
├── alert_type
├── caffeine_mg
├── spent_usd
└── created_at
```

### **Key Design Decisions**

**Why separate databases?**

- Allows each service to scale independently
- Analytics service can optimize for reads/aggregations
- Clear separation of concerns

**Why denormalized tables in Analytics?**

- Pre-calculated metrics for fast API responses
- Avoid expensive joins on every request
- Can be quickly cached or recomputed

**Why append-only alert_history?**

- Complete audit trail
- Can analyze alert patterns over time
- No data loss

**Why JSONB for feeling/quality counts?**

- Flexible schema for different feeling types
- Can easily add new feelings without migration
- Efficient for counting occurrences

---

## Event-Driven Flow

### **Event Schema**

```json
{
  "event_type": "coffee.logged",
  "version": "1.0",
  "event_id": "uuid",
  "timestamp": "2024-04-23T10:35:00Z",
  "payload": {
    "log_id": 123,
    "user_id": 5,
    "shop_id": 1,
    "coffee_type": "espresso",
    "price_usd": 5.00,
    "quality_rating": 4,
    "before_feeling": "focused",
    "after_feeling": "energized",
    "felt_better_after_mins": 240,
    "caffeine_mg": 95
  }
}
```

### **Complete Event Flow Example**

**Scenario**: You log a coffee at 10:35 AM

```
STEP 1: User Action
────────────────────
10:35 AM - User opens app, clicks "Log Coffee"
          Fills form: Blue Bottle, Espresso, $5.00, quality 4.5/5
          Before: "focused", After: "energized", lasted 240 mins
          Caffeine: 95mg

STEP 2: Coffee Service Receives Request
─────────────────────────────────────────
POST /api/v1/logs
Headers: Authorization: Bearer {token}
Body: {
  shop_id: 1,
  coffee_type: "espresso",
  price_usd: 5.00,
  quality_rating: 4.5,
  before_feeling: "focused",
  after_feeling: "energized",
  felt_better_after_mins: 240,
  caffeine_mg: 95
}

STEP 3: Coffee Service Processing
──────────────────────────────────
1. Validate request (authentication, authorization, data validation)
2. Save to coffee_logs table:
   INSERT INTO coffee_logs (user_id, shop_id, ...) VALUES (5, 1, ...)
   → Returns log_id = 42
3. Create and publish event to RabbitMQ:
   {
     "event_type": "coffee.logged",
     "payload": {
       "log_id": 42,
       "user_id": 5,
       "shop_id": 1,
       "coffee_type": "espresso",
       "price_usd": 5.00,
       "quality_rating": 4.5,
       "caffeine_mg": 95,
       ...
     }
   }
4. Return success response to frontend

STEP 4: Event in Message Queue
──────────────────────────────
RabbitMQ message broker holds event
Exchange: "coffee.events"
Routing key: "coffee.logged"

STEP 5: Analytics Service Consumes Event
─────────────────────────────────────────
0. Analytics service always listening on "coffee.logged" queue
1. Receives event immediately (< 100ms typically)
2. Fetches user preferences from Coffee Service DB:
   - caffeine_capacity_mg = 400
   - monthly_budget_usd = 50.00
3. Enters database transaction with SKIP LOCK

STEP 6: Analytics Service Updates Daily Stats
──────────────────────────────────────────────
SELECT * FROM daily_stats 
WHERE user_id = 5 AND date = '2024-04-23'
FOR UPDATE SKIP LOCKED;  ← Key: Don't block if locked!

If row exists:
  UPDATE daily_stats
  SET total_caffeine_mg = 190 + 95 = 285,
      total_spent_usd = 10.50 + 5.00 = 15.50,
      log_count = 2 + 1 = 3,
      exceeded_caffeine_capacity = false,
      exceeded_budget = false,
      updated_at = NOW()
  WHERE user_id = 5 AND date = '2024-04-23'

If row doesn't exist:
  INSERT INTO daily_stats
  (user_id, date, total_caffeine_mg, total_spent_usd, log_count)
  VALUES (5, '2024-04-23', 95, 5.00, 1)

STEP 7: Analytics Service Updates Shop Metrics
───────────────────────────────────────────────
SELECT * FROM shop_metrics
WHERE user_id = 5 AND shop_id = 1
FOR UPDATE SKIP LOCKED;

If exists:
  UPDATE shop_metrics
  SET visit_count = 7 + 1 = 8,
      total_spent_usd = 40.00 + 5.00 = 45.00,
      avg_quality_rating = (32 + 4.5) / 8 = 4.56,  ← Recalculate
      avg_price = 45.00 / 8 = 5.63,
      price_per_quality = 5.63 / 4.56 = 1.23,
      avg_caffeine_per_visit = (760 + 95) / 8 = 106.88,
      updated_at = NOW()
  WHERE user_id = 5 AND shop_id = 1

STEP 8: Analytics Service Updates Productivity Records
──────────────────────────────────────────────────────
For coffee_type = "espresso":
  avg_focus_duration_mins = (240 * 7 + 240) / 8 = 240  ← Still 240min
  sample_count = 7 + 1 = 8

STEP 9: Analytics Service Updates Impressions
──────────────────────────────────────────────
For shop_id = 1, coffee_type = "espresso":
  total_ratings = 7 + 1 = 8
  avg_quality = (32 + 4.5) / 8 = 4.56
  Update feeling counts:
    feeling_before_counts: {"focused": 5+1=6, "alert": 2}
    feeling_after_counts: {"energized": 3+1=4, "focused": 3}

STEP 10: Evaluate Alerts
────────────────────────
Check against user preferences:
  - Is 285mg > 400mg capacity? NO
  - Is 15.50 > 50.00 budget? NO
  - No alerts to trigger

STEP 11: Invalidate Cache
─────────────────────────
Redis DEL commands:
  DEL user:5:daily:2024-04-23
  DEL user:5:shop_comparison
  DEL user:5:productivity
  (Cache invalidated for immediate fresh reads)

STEP 12: Send Real-time Update to Frontend
───────────────────────────────────────────
WebSocket to user session:
{
  "type": "daily_stats_updated",
  "payload": {
    "caffeine": {
      "total_mg": 285,
      "capacity_mg": 400,
      "remaining_mg": 115,
      "percentage": 71
    },
    "budget": {
      "spent_usd": 15.50,
      "monthly_limit_usd": 50,
      "remaining_usd": 34.50,
      "percentage": 31
    }
  }
}

STEP 13: Frontend Updates UI
────────────────────────────
< 1 second from log submission:
✅ Caffeine bar shows: 285/400 (71% filled)
✅ Budget shows: $15.50/$50 (31% spent)
✅ New log appears in "Today's logs" list
✅ Shop comparison updates Blue Bottle visit count (8)
✅ Alert section remains empty (no warnings)

COMPLETE: Full flow from log to real-time UI update takes ~500-1000ms
```

---

## Real-Time Features

### **WebSocket Implementation**

The Analytics Service maintains WebSocket connections to deliver real-time updates:

```javascript
// Frontend (Next.js)
import { useEffect, useState } from 'react';

export function CoffeeTracker() {
  const [stats, setStats] = useState(null);

  useEffect(() => {
    // Connect to Analytics Service WebSocket
    const ws = new WebSocket('wss://analytics.app.com/ws/analytics');
    
    ws.onopen = () => {
      console.log('Connected to analytics');
      // Send subscription message
      ws.send(JSON.stringify({
        type: 'subscribe',
        user_id: currentUser.id,
        channels: ['daily_stats', 'alerts']
      }));
    };
    
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      
      if (update.type === 'daily_stats_updated') {
        setStats(update.payload);
        // UI instantly reflects new caffeine/budget totals
      } else if (update.type === 'alert_triggered') {
        // Show alert banner
        showAlertBanner(update.payload.message);
      }
    };
    
    return () => ws.close();
  }, []);

  return (
    <div>
      <CaffeineMeter total={stats?.caffeine?.total_mg} 
                     capacity={stats?.caffeine?.capacity_mg} />
      <BudgetMeter spent={stats?.budget?.spent_usd} 
                   limit={stats?.budget?.monthly_limit_usd} />
    </div>
  );
}
```

### **Polling Alternative**

If WebSocket is not needed, simple polling also works:

```javascript
// Simpler: Just poll every 5 seconds
useEffect(() => {
  const interval = setInterval(async () => {
    const response = await fetch('/api/v1/analytics/today');
    setStats(response.data);
  }, 5000);
  
  return () => clearInterval(interval);
}, []);
```

### **Cache-Aware Updates**

```python
# Analytics Service cache invalidation logic

def process_coffee_logged_event(event):
    # ... process event, update DB ...
    
    # Step 1: Delete stale cache
    cache.delete(f"user:{event.user_id}:daily:{event.date}")
    cache.delete(f"user:{event.user_id}:shop_comparison")
    cache.delete(f"user:{event.user_id}:productivity")
    
    # Step 2: Recalculate fresh data
    daily_stats = calculate_daily_stats(event.user_id, event.date)
    shop_comparison = calculate_shop_comparison(event.user_id)
    productivity = calculate_productivity(event.user_id)
    
    # Step 3: Cache fresh data (short TTL so stale data has less impact)
    cache.set(f"user:{event.user_id}:daily:{event.date}", 
              daily_stats, ttl=300)  # 5 minutes
    cache.set(f"user:{event.user_id}:shop_comparison", 
              shop_comparison, ttl=3600)  # 1 hour
    cache.set(f"user:{event.user_id}:productivity", 
              productivity, ttl=86400)  # 24 hours
    
    # Step 4: Notify frontend
    websocket.broadcast_to_user(event.user_id, {
      "type": "daily_stats_updated",
      "payload": daily_stats
    })
```

---

## Caching Strategy

### **Redis Cache Architecture**

```
Redis Instance
├── user:{user_id}:daily:{date}
│   ├── Stores: caffeine total, budget total, log count
│   ├── TTL: 5 minutes (short - changes frequently)
│   └── Invalidated on: new coffee log
│
├── user:{user_id}:shop_comparison
│   ├── Stores: all shops with metrics
│   ├── TTL: 1 hour (moderate - rarely changes)
│   └── Invalidated on: new coffee log
│
├── user:{user_id}:productivity
│   ├── Stores: productivity insights
│   ├── TTL: 24 hours (long - very stable)
│   └── Invalidated on: new coffee log
│
├── shop:{shop_id}:metadata
│   ├── Stores: shop name, location
│   ├── TTL: 7 days
│   └── Invalidated on: shop edit
│
└── user:{user_id}:preferences
    ├── Stores: caffeine capacity, budget limit
    ├── TTL: 24 hours
    └── Invalidated on: preference update
```

### **Cache Invalidation Pattern**

```python
# Define what to invalidate when

INVALIDATION_RULES = {
    "coffee.logged": [
        f"user:{user_id}:daily:{date}",
        f"user:{user_id}:shop_comparison",
        f"user:{user_id}:productivity",
        f"user:{user_id}:alerts"
    ],
    "preferences.updated": [
        f"user:{user_id}:preferences"
    ],
    "shop.updated": [
        f"shop:{shop_id}:metadata",
        f"user:{user_id}:shop_comparison"  # All users who visited this shop
    ]
}

def invalidate_cache(event_type, user_id, shop_id=None):
    keys_to_delete = INVALIDATION_RULES.get(event_type, [])
    for key in keys_to_delete:
        cache.delete(key)
```

### **Cache Hit Rates (Expected)**

|Cache Key|Hit Rate|Rationale|
|---|---|---|
|`daily_stats`|~85%|Most users check daily stats multiple times|
|`shop_comparison`|~60%|Checked less frequently, invalidated often|
|`productivity`|~50%|Checked occasionally, very stable|
|`preferences`|~95%|Rarely changes, hits across API calls|

---

## SKIP LOCK Implementation

### **What is SKIP LOCK?**

PostgreSQL's `SKIP LOCKED` clause allows a transaction to:

- Lock specific rows for update
- **Skip rows that are already locked** (don't wait)
- Continue processing without blocking

### **Why We Need It**

Scenario: Multiple coffee logs within seconds trigger concurrent updates

```
10:35:00 - Log 1: espresso 95mg
10:35:01 - Log 2: latte 190mg
10:35:02 - Log 3: cold brew 160mg

Without SKIP LOCKED:
- Thread 1 locks daily_stats row for user:5
- Thread 2 waits... waiting... waiting...
- Thread 3 waits... waiting... waiting...
- If locks held too long → deadlock risk
- Response time increases

With SKIP LOCKED:
- Thread 1 locks and updates row → done (100ms)
- Thread 2 skips (row locked) → creates new entry or retries → done (100ms)
- Thread 3 skips (row locked) → creates new entry or retries → done (100ms)
- All threads finish concurrently, no waiting
```

### **Implementation in Analytics Service**

```python
# Python with psycopg2

import psycopg2
from datetime import date

def update_daily_stats_with_skip_lock(user_id: int, event: CoffeeLoggedEvent):
    """
    Update daily_stats for a user, using SKIP LOCKED to avoid deadlocks
    when multiple logs are processed concurrently.
    """
    conn = db.get_connection()
    cursor = conn.cursor()
    
    try:
        # Start transaction
        cursor.execute("BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;")
        
        # Step 1: SELECT with SKIP LOCKED
        # This locks the row for update, but skips if already locked
        select_query = """
        SELECT id, total_caffeine_mg, total_spent_usd, log_count
        FROM daily_stats 
        WHERE user_id = %s AND date = %s
        FOR UPDATE SKIP LOCKED;
        """
        
        cursor.execute(select_query, (user_id, date.today()))
        row = cursor.fetchone()
        
        if row:
            # Row found and locked - update it
            row_id, caffeine, spent, count = row
            new_caffeine = caffeine + event.caffeine_mg
            new_spent = spent + event.price_usd
            new_count = count + 1
            
            update_query = """
            UPDATE daily_stats
            SET total_caffeine_mg = %s,
                total_spent_usd = %s,
                log_count = %s,
                exceeded_caffeine_capacity = %s,
                exceeded_budget = %s,
                updated_at = NOW()
            WHERE id = %s;
            """
            
            # Get user preferences to check thresholds
            prefs = get_user_preferences(user_id)
            exceeded_caffeine = new_caffeine > prefs['caffeine_capacity_mg']
            exceeded_budget = new_spent > prefs['monthly_budget_usd']
            
            cursor.execute(update_query, (
                new_caffeine,
                new_spent,
                new_count,
                exceeded_caffeine,
                exceeded_budget,
                row_id
            ))
            
        else:
            # Row not found or skipped - insert new row
            insert_query = """
            INSERT INTO daily_stats 
            (user_id, date, total_caffeine_mg, total_spent_usd, log_count,
             exceeded_caffeine_capacity, exceeded_budget)
            VALUES (%s, %s, %s, %s, 1, %s, %s);
            """
            
            prefs = get_user_preferences(user_id)
            exceeded_caffeine = event.caffeine_mg > prefs['caffeine_capacity_mg']
            exceeded_budget = event.price_usd > prefs['monthly_budget_usd']
            
            cursor.execute(insert_query, (
                user_id,
                date.today(),
                event.caffeine_mg,
                event.price_usd,
                exceeded_caffeine,
                exceeded_budget
            ))
        
        # Commit transaction
        conn.commit()
        return True
        
    except Exception as e:
        conn.rollback()
        # Retry logic here if needed
        logger.error(f"Error updating daily stats: {e}")
        return False
    finally:
        cursor.close()
        conn.close()


def update_shop_metrics_with_skip_lock(user_id: int, shop_id: int, 
                                      event: CoffeeLoggedEvent):
    """Update shop metrics using SKIP LOCKED"""
    conn = db.get_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute("BEGIN;")
        
        select_query = """
        SELECT id, visit_count, total_spent_usd, total_ratings, 
               total_quality_sum, avg_caffeine_per_visit
        FROM shop_metrics
        WHERE user_id = %s AND shop_id = %s
        FOR UPDATE SKIP LOCKED;
        """
        
        cursor.execute(select_query, (user_id, shop_id))
        row = cursor.fetchone()
        
        if row:
            row_id, visits, spent, ratings, quality_sum, avg_caff = row
            new_visits = visits + 1
            new_spent = spent + event.price_usd
            new_ratings = ratings + 1
            new_quality_sum = quality_sum + event.quality_rating
            new_avg_quality = new_quality_sum / new_ratings
            new_avg_price = new_spent / new_visits
            new_price_per_quality = new_avg_price / new_avg_quality
            new_avg_caff = (avg_caff * visits + event.caffeine_mg) / new_visits
            
            update_query = """
            UPDATE shop_metrics
            SET visit_count = %s,
                total_spent_usd = %s,
                avg_quality_rating = %s,
                avg_price = %s,
                price_per_quality = %s,
                avg_caffeine_per_visit = %s,
                updated_at = NOW()
            WHERE id = %s;
            """
            
            cursor.execute(update_query, (
                new_visits, new_spent, new_avg_quality, 
                new_avg_price, new_price_per_quality,
                new_avg_caff, row_id
            ))
        else:
            insert_query = """
            INSERT INTO shop_metrics
            (user_id, shop_id, visit_count, total_spent_usd, 
             avg_quality_rating, avg_price, price_per_quality, 
             avg_caffeine_per_visit)
            VALUES (%s, %s, 1, %s, %s, %s, %s, %s);
            """
            
            cursor.execute(insert_query, (
                user_id, shop_id, event.price_usd,
                event.quality_rating, event.price_usd,
                event.price_usd / event.quality_rating,
                event.caffeine_mg
            ))
        
        conn.commit()
        return True
        
    except Exception as e:
        conn.rollback()
        logger.error(f"Error updating shop metrics: {e}")
        return False
    finally:
        cursor.close()
        conn.close()
```

### **Concurrency Handling**

```python
# Robust event processing with retry logic

import time
from functools import retry

@retry(max_attempts=3, backoff_seconds=0.1)
def process_coffee_logged_event(event: CoffeeLoggedEvent):
    """Process with automatic retry on conflict"""
    try:
        # Update daily stats
        if not update_daily_stats_with_skip_lock(event.user_id, event):
            raise Exception("Failed to update daily stats")
        
        # Update shop metrics
        if not update_shop_metrics_with_skip_lock(event.user_id, event.shop_id, event):
            raise Exception("Failed to update shop metrics")
        
        # Update productivity
        if not update_productivity_with_skip_lock(event.user_id, event.shop_id, event):
            raise Exception("Failed to update productivity")
        
        # Update impressions
        if not update_impressions_with_skip_lock(event.user_id, event.shop_id, event):
            raise Exception("Failed to update impressions")
        
        # Check alerts
        check_and_create_alerts(event.user_id, date.today(), event)
        
        # Invalidate cache and notify
        invalidate_user_cache(event.user_id)
        notify_frontend_update(event.user_id)
        
        logger.info(f"Successfully processed log {event.log_id}")
        return True
        
    except Exception as e:
        logger.warning(f"Event processing failed (will retry): {e}")
        raise  # Retry decorator will handle

def retry(max_attempts=3, backoff_seconds=0.1):
    """Decorator for retry logic"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_attempts - 1:
                        wait_time = backoff_seconds * (2 ** attempt)
                        time.sleep(wait_time)
            raise last_error
        return wrapper
    return decorator
```

---

## Frontend Features

### **Next.js Application Structure**

```
coffee-tracker-frontend/
├── app/
│   ├── layout.tsx
│   ├── page.tsx (dashboard)
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login/route.ts
│   │   │   └── register/route.ts
│   │   └── proxy/
│   │       └── [...].ts (forward to backend)
│   └── (auth)/
│       ├── login/page.tsx
│       └── register/page.tsx
├── components/
│   ├── CoffeeDashboard.tsx
│   ├── QuickLogModal.tsx
│   ├── CaffeineMeter.tsx
│   ├── BudgetMeter.tsx
│   ├── ShopComparison.tsx
│   ├── ProductivityInsights.tsx
│   └── AlertBanner.tsx
├── hooks/
│   ├── useAnalytics.ts (WebSocket/polling)
│   ├── useCoffeeLog.ts
│   └── useAuth.ts
├── lib/
│   ├── api.ts (HTTP client)
│   └── websocket.ts (WebSocket management)
└── styles/
    └── globals.css
```

### **Key Components**

#### Quick Log Modal

```typescript
// components/QuickLogModal.tsx
import { useState } from 'react';

export function QuickLogModal({ isOpen, onClose }) {
  const [form, setForm] = useState({
    shopId: '',
    coffeeType: 'espresso',
    price: '',
    qualityRating: 3,
    beforeFeeling: 'alert',
    afterFeeling: 'focused',
    focusDuration: '',
    caffeineMg: 95
  });
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    const response = await fetch('/api/logs', {
      method: 'POST',
      body: JSON.stringify(form)
    });
    if (response.ok) {
      onClose();
      // Real-time update will come via WebSocket
    }
  };
  
  return (
    <dialog open={isOpen}>
      <form onSubmit={handleSubmit}>
        <select value={form.shopId} onChange={e => setForm({...form, shopId: e.target.value})}>
          <option>Select Coffee Shop</option>
          {/* List shops */}
        </select>
        
        <select value={form.coffeeType}>
          <option>Espresso</option>
          <option>Latte</option>
          <option>Cold Brew</option>
        </select>
        
        <input type="number" placeholder="Price ($)" value={form.price} />
        <input type="range" min="1" max="5" value={form.qualityRating} />
        <input type="number" placeholder="Felt better after (mins)" value={form.focusDuration} />
        
        <button type="submit">Log Coffee</button>
      </form>
    </dialog>
  );
}
```

#### Dashboard

```typescript
// components/CoffeeDashboard.tsx
import { useAnalytics } from '@/hooks/useAnalytics';

export function CoffeeDashboard() {
  const { stats, alerts, shops, productivity } = useAnalytics();
  
  if (!stats) return <div>Loading...</div>;
  
  return (
    <div className="dashboard">
      {/* Today's Summary */}
      <div className="summary">
        <div className="metric caffeine">
          <h3>Caffeine</h3>
          <CaffeineMeter 
            current={stats.caffeine.total_mg}
            capacity={stats.caffeine.capacity_mg}
            remaining={stats.caffeine.remaining_mg}
          />
          <p>{stats.caffeine.total_mg}mg / {stats.caffeine.capacity_mg}mg</p>
        </div>
        
        <div className="metric budget">
          <h3>Budget</h3>
          <BudgetMeter
            spent={stats.budget.spent_usd}
            limit={stats.budget.monthly_limit_usd}
          />
          <p>${stats.budget.spent_usd} / ${stats.budget.monthly_limit_usd}</p>
        </div>
      </div>
      
      {/* Alerts */}
      {alerts && alerts.length > 0 && (
        <AlertBanner alerts={alerts} />
      )}
      
      {/* Today's Logs */}
      <div className="logs-today">
        <h3>Today's Logs ({stats.coffee_count})</h3>
        {stats.logs_today.map(log => (
          <LogEntry key={log.id} log={log} />
        ))}
      </div>
      
      {/* Shop Comparison */}
      <ShopComparison shops={shops} />
      
      {/* Productivity Insights */}
      <ProductivityInsights insights={productivity} />
    </div>
  );
}
```

#### Caffeine Meter

```typescript
// components/CaffeineMeter.tsx
export function CaffeineMeter({ current, capacity, remaining }) {
  const percentage = (current / capacity) * 100;
  const color = percentage > 80 ? 'red' : percentage > 50 ? 'yellow' : 'green';
  
  return (
    <div className="caffeine-meter">
      <div className={`meter-bar ${color}`} style={{ width: `${percentage}%` }}>
        {Math.round(percentage)}%
      </div>
      <p className="remaining">{remaining}mg remaining</p>
    </div>
  );
}
```

#### Shop Comparison Table

```typescript
// components/ShopComparison.tsx
export function ShopComparison({ shops }) {
  return (
    <table className="shops-table">
      <thead>
        <tr>
          <th>Shop</th>
          <th>Visits</th>
          <th>Avg Quality</th>
          <th>Quality/$</th>
          <th>Focus Time</th>
          <th>Consistency</th>
        </tr>
      </thead>
      <tbody>
        {shops.map(shop => (
          <tr key={shop.shop_id}>
            <td>{shop.name}</td>
            <td>{shop.visits}</td>
            <td>⭐ {shop.avg_quality_rating}/5</td>
            <td>${shop.price_per_quality.toFixed(2)}</td>
            <td>{shop.avg_focus_duration_mins}m</td>
            <td>{shop.impressions.consistent ? '✓ Consistent' : '⚠ Variable'}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

#### Productivity Insights

```typescript
// components/ProductivityInsights.tsx
export function ProductivityInsights({ insights }) {
  return (
    <div className="productivity">
      <h3>Productivity Insights</h3>
      
      <div className="insight-card">
        <h4>🏆 Best Coffee Type</h4>
        <p>{insights.best_coffee_type}</p>
        <span>{insights.by_coffee_type[insights.best_coffee_type].avg_focus_mins}m focus</span>
      </div>
      
      <div className="insight-card">
        <h4>🏪 Best Shop</h4>
        <p>{insights.best_shop.name}</p>
        <span>Avg {insights.best_shop.avg_focus_mins}m focus</span>
      </div>
      
      <div className="insights-list">
        <h4>💡 Recommendations</h4>
        <ul>
          {insights.insights.map((insight, i) => (
            <li key={i}>{insight}</li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

### **Real-Time Hook**

```typescript
// hooks/useAnalytics.ts
import { useEffect, useState } from 'react';

export function useAnalytics() {
  const [stats, setStats] = useState(null);
  const [alerts, setAlerts] = useState([]);
  const [shops, setShops] = useState([]);
  const [productivity, setProductivity] = useState(null);
  
  useEffect(() => {
    // Option 1: WebSocket for real-time updates
    const ws = new WebSocket('wss://api.app.com/ws/analytics');
    
    ws.onopen = () => {
      ws.send(JSON.stringify({ 
        type: 'subscribe', 
        channels: ['daily_stats', 'alerts'] 
      }));
    };
    
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      
      if (message.type === 'daily_stats_updated') {
        setStats(message.payload);
      } else if (message.type === 'alert_triggered') {
        setAlerts(prev => [message.payload, ...prev]);
      }
    };
    
    // Option 2: Initial fetch + polling fallback
    const fetchData = async () => {
      try {
        const [statsRes, shopsRes, productivityRes] = await Promise.all([
          fetch('/api/analytics/today'),
          fetch('/api/analytics/shops/comparison'),
          fetch('/api/analytics/productivity')
        ]);
        
        setStats(await statsRes.json());
        setShops(await shopsRes.json());
        setProductivity(await productivityRes.json());
      } catch (error) {
        console.error('Failed to fetch analytics:', error);
      }
    };
    
    fetchData();
    const pollInterval = setInterval(fetchData, 5000);
    
    return () => {
      ws.close();
      clearInterval(pollInterval);
    };
  }, []);
  
  return { stats, alerts, shops, productivity };
}
```

---

## Tech Stack

### **Backend Services**

|Component|Technology|Reasoning|
|---|---|---|
|Coffee Service|Golang + Gin/Chi|Fast, concurrent, simple CRUD, great for learning DevOps|
|Analytics Service|Python + FastAPI|Data aggregation, easy to write business logic, event libraries|
|Message Queue|RabbitMQ|Simple, reliable, great for learning event-driven|
|Primary DB|PostgreSQL|SKIP LOCK support, powerful queries, great for analytics|
|Cache|Redis|Standard choice, fast, perfect for cache patterns|

### **Frontend**

|Component|Technology|Reasoning|
|---|---|---|
|Framework|Next.js|Full-stack, SSR, great DX|
|Styling|Tailwind CSS or CSS Modules|Fast development, responsive|
|Real-time|WebSocket + Polling|Learn real-time patterns|
|State|React Hooks|Modern, sufficient for this project|

### **DevOps**

|Component|Technology|Reasoning|
|---|---|---|
|Containerization|Docker|Industry standard, essential learning|
|Orchestration|Docker Compose (local)|Learn without Kubernetes complexity first|
|CI/CD|GitHub Actions|Free, integrated, good for learning|
|Infrastructure|AWS/GCP (optional)|Deploy for real learning experience|

### **Version Control**

- Git + GitHub
- Monorepo (all services in one repo) or separate repos (more realistic)

---

## Implementation Timeline

### **Phase 1: Foundation (Week 1-2)**

**Goals**: Basic services, DB, CRUD operations

- [ ] Set up project structure
    
    - [ ] Golang Coffee Service scaffold
    - [ ] Python Analytics Service scaffold
    - [ ] Next.js frontend scaffold
    - [ ] PostgreSQL setup with schemas
    - [ ] RabbitMQ setup
- [ ] Coffee Service core
    
    - [ ] User authentication (JWT)
    - [ ] User preferences CRUD
    - [ ] Coffee shops CRUD
    - [ ] Coffee logs CRUD
- [ ] Basic frontend
    
    - [ ] Login/Register pages
    - [ ] Simple log form
    - [ ] List logs view

**Deliverable**: Can log coffee, see list of logs

---

### **Phase 2: Event-Driven (Week 3)**

**Goals**: Event publishing and consumption

- [ ] RabbitMQ setup
    
    - [ ] Coffee Service publishes `CoffeeLogged` events
    - [ ] Analytics Service consumes events
- [ ] Analytics Service basic
    
    - [ ] Consumes `CoffeeLogged` events
    - [ ] Updates `daily_stats` table
    - [ ] GET `/api/v1/analytics/today` endpoint
- [ ] Frontend real-time
    
    - [ ] WebSocket or polling connection
    - [ ] Display daily caffeine/budget totals

**Deliverable**: Real-time caffeine and budget tracking

---

### **Phase 3: SKIP LOCK & Concurrency (Week 4)**

**Goals**: Concurrent updates, SKIP LOCK implementation

- [ ] Implement SKIP LOCK in Analytics Service
    
    - [ ] Update `daily_stats` with SKIP LOCKED
    - [ ] Update `shop_metrics` with SKIP LOCKED
    - [ ] Handle retries and conflicts
- [ ] Add stress testing
    
    - [ ] Simulate 100 concurrent logs
    - [ ] Verify no deadlocks
    - [ ] Measure performance

**Deliverable**: Robust concurrent processing

---

### **Phase 4: Analytics & Insights (Week 5)**

**Goals**: Full analytics features

- [ ] Shop metrics
    
    - [ ] Track visit frequency
    - [ ] Calculate quality/price ratio
    - [ ] Average caffeine per visit
- [ ] Productivity tracking
    
    - [ ] Aggregate focus duration per shop
    - [ ] Track by coffee type
    - [ ] Generate insights
- [ ] Impressions tracking
    
    - [ ] Quality ratings aggregation
    - [ ] Feeling tracking (before/after)
    - [ ] Consistency metrics

**Deliverable**: Shop comparison and productivity insights

---

### **Phase 5: Caching (Week 6)**

**Goals**: Redis caching strategy

- [ ] Implement Redis cache layer
    
    - [ ] Cache daily stats (5min TTL)
    - [ ] Cache shop comparison (1hr TTL)
    - [ ] Cache productivity insights (24hr TTL)
- [ ] Cache invalidation
    
    - [ ] Invalidate on new log
    - [ ] Invalidate on preferences update
    - [ ] Measure cache hit rates

**Deliverable**: Optimized API response times

---

### **Phase 6: Alerts & Full Frontend (Week 7)**

**Goals**: Alerts and complete UI

- [ ] Alert system
    
    - [ ] Caffeine capacity alerts
    - [ ] Budget alerts
    - [ ] Alert history
- [ ] Complete frontend
    
    - [ ] Caffeine meter component
    - [ ] Budget meter component
    - [ ] Shop comparison table
    - [ ] Productivity insights display
    - [ ] Alert banner

**Deliverable**: Functional web application

---

### **Phase 7: DevOps & Deployment (Week 8)**

**Goals**: Containerization and deployment

- [ ] Docker
    
    - [ ] Dockerfile for each service
    - [ ] docker-compose.yml
    - [ ] Local environment setup
- [ ] CI/CD
    
    - [ ] GitHub Actions workflow
    - [ ] Automated tests on push
    - [ ] Build and push Docker images
- [ ] Deployment (optional)
    
    - [ ] Deploy to cloud (AWS/GCP)
    - [ ] Set up database backups
    - [ ] Monitor health

**Deliverable**: Containerized, deployable application

---

### **Phase 8: Polish & Documentation (Week 8-9)**

**Goals**: Quality and knowledge sharing

- [ ] Testing
    
    - [ ] Unit tests for business logic
    - [ ] Integration tests for event flow
    - [ ] E2E tests for key workflows
- [ ] Documentation
    
    - [ ] README with setup instructions
    - [ ] API documentation (OpenAPI)
    - [ ] Architecture diagram explanations
    - [ ] Learning notes on SKIP LOCK, caching, etc.
- [ ] Code quality
    
    - [ ] Linting and formatting
    - [ ] Error handling
    - [ ] Logging and monitoring

**Deliverable**: Production-ready codebase, comprehensive docs

---

## Learning Outcomes

By completing this project, you'll have learned:

### **Microservices**

- ✅ Service boundaries and responsibilities
- ✅ Inter-service communication
- ✅ Scaling independent services
- ✅ Data consistency across services

### **Event-Driven Architecture**

- ✅ Event publishing and subscribing
- ✅ Asynchronous processing
- ✅ Message queue patterns
- ✅ Event sourcing concepts

### **Database Optimization**

- ✅ SKIP LOCK for concurrent updates
- ✅ Denormalization for performance
- ✅ Query optimization with indexes
- ✅ ACID transactions

### **Caching Strategies**

- ✅ Multi-layer caching architecture
- ✅ Cache invalidation patterns
- ✅ TTL strategies
- ✅ Cache hit rate optimization

### **DevOps & Infrastructure**

- ✅ Docker containerization
- ✅ Service orchestration
- ✅ CI/CD pipelines
- ✅ Deployment practices

### **Frontend Integration**

- ✅ Real-time WebSocket updates
- ✅ Responsive UI components
- ✅ State management with hooks
- ✅ API integration patterns

---

## Next Steps

1. **Create the project scaffold** with folder structure
2. **Start Phase 1** with Coffee Service CRUD operations
3. **Document decisions** as you build
4. **Test frequently** with concurrent operations
5. **Deploy early** - don't wait until the end

Good luck! This is a fantastic project for your portfolio. 🚀