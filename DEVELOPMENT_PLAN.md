# 🚀 URL Shortener — 1-Week Development Plan & Architecture Guide

> **For Go developers porting to Ruby on Rails.**
> This guide gives you implementation *pointers*, not full solutions — you learn by doing.

---

## 📋 Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Project Structure](#2-project-structure)
3. [Database Schema](#3-database-schema)
4. [Day-by-Day Plan (7 Days)](#4-day-by-day-plan-7-days)
   - [Day 1 — Project Setup & Go→Rails Mental Model](#-day-1--project-setup--gorails-mental-model)
   - [Day 2 — Core Models & URL Shortening Logic](#-day-2--core-models--url-shortening-logic)
   - [Day 3 — API Controllers & Redis Caching](#-day-3--api-controllers--redis-caching)
   - [Day 4 — Click Tracking, Background Jobs & Rate Limiting](#-day-4--click-tracking-background-jobs--rate-limiting)
   - [Day 5 — Analytics Stats API & URL Expiration](#-day-5--analytics-stats-api--url-expiration)
   - [Day 6 — Hardening, Validation & DevOps](#-day-6--hardening-validation--devops)
   - [Day 7 — Polish, Documentation & Deploy Prep](#-day-7--polish-documentation--deploy-prep)
5. [Go → Rails Mental Model Map](#5-go--rails-mental-model-map)
6. [Key Gems (Gemfile Reference)](#6-key-gems-gemfile-reference)
7. [Additional Recommendations & Tips](#7-additional-recommendations--tips)

---

## 1. 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT (Browser / API Tool)                  │
│           POST /api/v1/urls      GET /:short_code               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                  RUBY ON RAILS APP (API Mode)                   │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │   Controllers   │  │    Models    │  │      Services      │ │
│  │                 │  │              │  │                    │ │
│  │  UrlsController │  │  Url (AR)    │  │  UrlShortener      │ │
│  │  StatsController│  │  Click (AR)  │  │  ClickTracker      │ │
│  │  Redirects...   │  │              │  │  RateLimiter       │ │
│  └────────┬────────┘  └──────┬───────┘  └─────────┬──────────┘ │
│           │                  │                     │            │
└───────────┼──────────────────┼─────────────────────┼────────────┘
            │                  │                     │
            ▼                  ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Redis (Cache + Rate Limit)                    │
│                                                                 │
│   short:{code}   →  original_url          TTL: 24h (cache)     │
│   clicks:{code}  →  buffered counter      TTL: flushed/5min    │
│   rate:{ip}      →  request counter       TTL: 1 min window    │
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│               PostgreSQL (Persistent Storage)                   │
│                                                                 │
│   urls   (id, short_code, original_url, expires_at, ...)       │
│   clicks (id, url_id, ip_address, user_agent, referer, ...)    │
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│               Sidekiq (Background Jobs via Redis)               │
│                                                                 │
│   FlushClicksJob  →  Redis counters → PostgreSQL (every 5 min) │
│   ExpireUrlsJob   →  Purge/soft-delete expired URLs            │
└─────────────────────────────────────────────────────────────────┘
```

### Component Roles

| Component | Role |
|---|---|
| **Rails App** | Handles HTTP requests, business logic, orchestrates other components |
| **Redis** | Ultra-fast cache for URL lookups, atomic click counters, rate-limit windows |
| **PostgreSQL** | Source of truth — durable storage for URLs, click history, analytics |
| **Sidekiq** | Async/background processing; flushes Redis counters to DB, expires old URLs |

---

## 2. 📂 Project Structure

```
url_shortener/
├── app/
│   ├── controllers/
│   │   ├── application_controller.rb       # Base controller — shared error handling
│   │   ├── redirects_controller.rb         # GET /:short_code → 302 redirect
│   │   └── api/
│   │       └── v1/
│   │           ├── urls_controller.rb      # POST /api/v1/urls (create short URL)
│   │           └── stats_controller.rb     # GET /api/v1/urls/:code/stats (analytics)
│   ├── models/
│   │   ├── application_record.rb           # Base model (like embedding a base struct in Go)
│   │   ├── url.rb                          # ActiveRecord model for urls table
│   │   └── click.rb                        # ActiveRecord model for clicks table
│   ├── services/
│   │   ├── url_shortener_service.rb        # Generates short codes, persists to DB + Redis
│   │   ├── click_tracker_service.rb        # Buffers clicks in Redis, enqueues flush job
│   │   └── rate_limiter_service.rb         # Redis INCR/EXPIRE sliding window rate limiter
│   ├── jobs/
│   │   ├── application_job.rb              # Base job class
│   │   ├── flush_clicks_job.rb             # Sidekiq: flush Redis click counters → PostgreSQL
│   │   └── expire_urls_job.rb              # Sidekiq: purge URLs past expires_at
│   └── serializers/
│       ├── url_serializer.rb               # Shapes JSON response for URL resource
│       └── click_serializer.rb             # Shapes JSON response for click/stats
├── config/
│   ├── database.yml                        # PostgreSQL connection config
│   ├── redis.yml                           # Redis connection config (Rails 7.1+)
│   ├── routes.rb                           # All URL routing rules
│   ├── sidekiq.yml                         # Sidekiq queue config
│   └── initializers/
│       ├── redis.rb                        # Sets up global REDIS constant
│       └── sidekiq.rb                      # Configures Sidekiq server/client Redis pool
├── db/
│   ├── schema.rb                           # Auto-generated; source of truth for DB shape
│   └── migrate/
│       ├── YYYYMMDDHHMMSS_create_urls.rb   # urls table migration
│       └── YYYYMMDDHHMMSS_create_clicks.rb # clicks table migration
├── spec/                                   # RSpec test suite
│   ├── rails_helper.rb
│   ├── spec_helper.rb
│   ├── models/
│   │   ├── url_spec.rb
│   │   └── click_spec.rb
│   ├── services/
│   │   ├── url_shortener_service_spec.rb
│   │   └── rate_limiter_service_spec.rb
│   ├── jobs/
│   │   └── flush_clicks_job_spec.rb
│   └── requests/
│       ├── urls_spec.rb                    # Integration tests for API endpoints
│       └── redirects_spec.rb
├── Gemfile
├── Gemfile.lock
├── Dockerfile
├── docker-compose.yml                      # Rails + PostgreSQL + Redis + Sidekiq
└── README.md
```

> **Go→Rails note:** In Go you organize by `pkg/`, `cmd/`, `internal/`. Rails enforces its own convention — `app/models/`, `app/controllers/`, `app/services/`. Trust the convention; don't fight it.

---

## 3. 📊 Database Schema

### PostgreSQL Tables

```sql
-- urls table
CREATE TABLE urls (
    id            BIGSERIAL PRIMARY KEY,
    short_code    VARCHAR(10)  NOT NULL UNIQUE,
    original_url  TEXT         NOT NULL,
    expires_at    TIMESTAMP,
    click_count   INTEGER      NOT NULL DEFAULT 0,
    created_at    TIMESTAMP    NOT NULL,
    updated_at    TIMESTAMP    NOT NULL
);

CREATE INDEX idx_urls_short_code ON urls (short_code);
CREATE INDEX idx_urls_expires_at ON urls (expires_at);

-- clicks table
CREATE TABLE clicks (
    id           BIGSERIAL PRIMARY KEY,
    url_id       BIGINT       NOT NULL REFERENCES urls(id) ON DELETE CASCADE,
    ip_address   INET,
    user_agent   TEXT,
    referer      TEXT,
    country      VARCHAR(2),
    clicked_at   TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clicks_url_id     ON clicks (url_id);
CREATE INDEX idx_clicks_clicked_at ON clicks (clicked_at);
```

> **Rails note:** You don't write this SQL by hand. Rails migrations generate it. Run `rails g model Url short_code:string ...` and Rails creates the migration file for you.

### Redis Key Design

| Key Pattern | Type | Purpose | TTL |
|---|---|---|---|
| `short:{code}` | String | Cached `original_url` for fast redirect lookups | 24 hours |
| `clicks:{code}` | String (counter) | Buffered click count (flushed to PostgreSQL) | Cleared on flush (every ~5 min) |
| `rate:{ip}` | String (counter) | Request counter for rate limiting per IP | 60 seconds (sliding window) |

---

## 4. 📅 Day-by-Day Plan (7 Days)

---

### 🗓️ Day 1 — Project Setup & Go→Rails Mental Model

**Goal:** Get Rails app running with PostgreSQL and Redis. Understand the Go→Rails conceptual shift.

#### Tasks

| # | Task | Command / Notes |
|---|---|---|
| 1 | Install Ruby via `rbenv` or `asdf` | `rbenv install 3.3.0 && rbenv global 3.3.0` |
| 2 | Install Rails | `gem install rails` |
| 3 | Create project (API mode, no default tests) | `rails new url_shortener --api --database=postgresql -T` |
| 4 | Set up Docker Compose | Create `docker-compose.yml` with `postgres:16`, `redis:7`, Rails app service |
| 5 | Update `Gemfile` | Add `redis`, `sidekiq`, `rack-cors`, `rspec-rails`, `factory_bot_rails`, `faker`, `dotenv-rails` |
| 6 | Configure `config/database.yml` | Point to Docker PostgreSQL service, use ENV vars |
| 7 | Create Redis initializer | Create `config/initializers/redis.rb` |
| 8 | Verify boot | `rails db:create && rails server` |

#### 💡 Implementation Pointers

- Look into `config/initializers/` — files here are auto-loaded at startup. Use this to set up a global `REDIS` constant with `Redis.new(url: ENV["REDIS_URL"])`.
- For `docker-compose.yml`: define `postgres` and `redis` services, then add a `depends_on` key in the `web` service. Check the [Compose docs on service linking](https://docs.docker.com/compose/compose-file/05-services/#depends_on).
- In `config/database.yml`, use `<%= ENV["DATABASE_URL"] %>` (ERB is supported) to read env vars.
- After adding gems to `Gemfile`, run `bundle install` — this is the Rails equivalent of `go mod tidy`.
- Run `rails db:create` to create the dev and test databases.

<details>
<summary>🔍 Hint: Redis initializer shape (click to reveal)</summary>

```ruby
# config/initializers/redis.rb
# Hint: create a Redis connection and assign it to a constant
# so you can use REDIS.get / REDIS.set anywhere in the app.
# Look up: Redis.new, connection_pool gem for thread safety.

REDIS = Redis.new(url: ENV.fetch("REDIS_URL", "redis://localhost:6379/0"))
# <!-- TODO: Add error handling if Redis is unreachable -->
```

</details>

#### 🔄 Go → Rails Cheatsheet (Day 1 Focus)

| Go | Rails |
|---|---|
| `go mod init` / `go.sum` | `bundle init` / `Gemfile.lock` |
| `go run main.go` | `rails server` |
| `os.Getenv("KEY")` | `ENV["KEY"]` or `ENV.fetch("KEY")` |
| Package imports | Gem dependencies in `Gemfile` |
| `log.Println(...)` | `Rails.logger.info(...)` |

**Deliverable ✅** App boots, connects to PostgreSQL (`rails db:create` succeeds) and Redis (no connection errors in logs).

---

### 🗓️ Day 2 — Core Models & URL Shortening Logic

**Goal:** Generate `Url` and `Click` models, write validations, and build the `UrlShortenerService`.

#### Tasks

| # | Task | Command / Notes |
|---|---|---|
| 1 | Generate `Url` model | `rails g model Url short_code:string original_url:text expires_at:datetime click_count:integer` |
| 2 | Generate `Click` model | `rails g model Click url:references ip_address:string user_agent:text referer:text country:string clicked_at:datetime` |
| 3 | Edit migrations | Add `null: false`, `default: 0` for `click_count`, add indexes |
| 4 | Run migrations | `rails db:migrate` |
| 5 | Add model validations | Presence, URL format, uniqueness on `short_code` |
| 6 | Create `UrlShortenerService` | `app/services/url_shortener_service.rb` |
| 7 | Write unit tests | `rspec spec/models/` and `rspec spec/services/` |

#### 💡 Implementation Pointers

- Use `validates :original_url, presence: true, format: { with: URI::DEFAULT_PARSER.make_regexp }` for URL validation.
- Use `validates :short_code, uniqueness: true` on the model — but also enforce uniqueness at the DB level with an index (migrations).
- For short code generation, look up `SecureRandom.alphanumeric(7)` — it generates a random 7-character string. Use a `loop` to keep generating until you find one that doesn't already exist in the database.
- The **Service Object pattern** means: a plain Ruby class (not a model, not a controller) that does one thing. Name it with a verb, give it a `call` method.
- Check `Url.exists?(short_code: code)` inside your generation loop.

#### 🏗️ Service Skeleton

```ruby
# app/services/url_shortener_service.rb
class UrlShortenerService
  def initialize(original_url, expires_in: nil, custom_code: nil)
    @original_url = original_url
    @expires_in   = expires_in
    @custom_code  = custom_code
  end

  def call
    # <!-- TODO: Validate @original_url before proceeding -->
    short_code = @custom_code || generate_unique_code
    # <!-- TODO: Handle collision if custom_code already exists -->

    url = Url.create!(
      short_code:   short_code,
      original_url: @original_url,
      expires_at:   @expires_in ? Time.current + @expires_in : nil
    )

    # <!-- TODO: Cache the new URL in Redis (REDIS.setex) -->
    url
  end

  private

  def generate_unique_code
    loop do
      code = SecureRandom.alphanumeric(7)
      # <!-- TODO: break out of the loop only when code is unique -->
    end
  end
end
```

> **Go→Rails note:** In Go you'd return `(Url, error)`. In Rails, `create!` *raises* an exception (`ActiveRecord::RecordInvalid`) on failure. Wrap callers in `begin/rescue` or use `create` (no bang) and check `.persisted?`.

**Deliverable ✅** `UrlShortenerService.new("https://example.com").call` creates a row in the database.

---

### 🗓️ Day 3 — API Controllers & Redis Caching

**Goal:** Wire up routes, build `UrlsController` and `RedirectsController`, add Redis caching.

#### Tasks

| # | Task | Notes |
|---|---|---|
| 1 | Set up routes | Edit `config/routes.rb` |
| 2 | Generate controllers | `rails g controller api/v1/urls` and `rails g controller redirects` |
| 3 | Implement `UrlsController#create` | Call `UrlShortenerService`, return JSON |
| 4 | Implement `RedirectsController#show` | Redis lookup → DB fallback → `redirect_to` |
| 5 | Add Redis caching layer | Cache on creation, populate on miss |
| 6 | Add error handling | Return structured JSON errors |
| 7 | Write request specs | `spec/requests/` |

#### 💡 Implementation Pointers

- Use `REDIS.get("short:#{code}")` for cache hits, `REDIS.setex("short:#{code}", 86_400, url)` for writes (24h TTL).
- Use `redirect_to original_url, status: :found` for the 302 redirect in `RedirectsController`.
- `before_action` is Rails' middleware per-controller — use it to DRY up shared logic (e.g., finding a URL by short code).
- For JSON responses, use `render json: { ... }, status: :created` — no view files needed in API mode.
- Think about cache invalidation: what happens when a URL expires? Clear the Redis key too.

#### 🗺️ Routes Skeleton

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Redirect route — must match short code format
  get "/:short_code", to: "redirects#show",
    constraints: { short_code: /[a-zA-Z0-9]{4,12}/ }

  namespace :api do
    namespace :v1 do
      resources :urls, only: [:create], param: :short_code do
        member do
          get :stats
        end
      end
      # <!-- TODO: Add a health check route -->
    end
  end
end
```

#### 🏗️ Controller Skeletons

```ruby
# app/controllers/redirects_controller.rb
class RedirectsController < ApplicationController
  def show
    # <!-- TODO: Look up short_code in Redis first -->
    # <!-- TODO: If cache miss, look up in PostgreSQL -->
    # <!-- TODO: If not found, render 404 JSON -->
    # <!-- TODO: Increment click counter (async via ClickTrackerService) -->
    # <!-- TODO: redirect_to with 302 status -->
  end
end
```

```ruby
# app/controllers/api/v1/urls_controller.rb
module Api
  module V1
    class UrlsController < ApplicationController
      def create
        # <!-- TODO: Permit and validate params -->
        # <!-- TODO: Call UrlShortenerService -->
        # <!-- TODO: Return 201 with short URL in JSON response -->
        # <!-- TODO: Handle ActiveRecord::RecordInvalid and render 422 -->
      end
    end
  end
end
```

**Deliverable ✅** `POST /api/v1/urls` creates a short URL; `GET /:code` redirects to the original URL.

---

### 🗓️ Day 4 — Click Tracking, Background Jobs & Rate Limiting

**Goal:** Track clicks asynchronously, flush counters to PostgreSQL via Sidekiq, add rate limiting.

#### Tasks

| # | Task | Notes |
|---|---|---|
| 1 | Create `ClickTrackerService` | Increment Redis counter, enqueue metadata job |
| 2 | Configure Sidekiq | `config/initializers/sidekiq.rb`, `config/sidekiq.yml` |
| 3 | Create `FlushClicksJob` | Read Redis counters, bulk-update DB, insert click rows |
| 4 | Schedule `FlushClicksJob` | Use `sidekiq-cron` gem or Sidekiq Scheduler every 5 minutes |
| 5 | Create `RateLimiterService` | Redis INCR + EXPIRE, 60 req/min per IP |
| 6 | Add rate limiter to controllers | `before_action :check_rate_limit` |
| 7 | Write tests | Job tests with `sidekiq/testing` fake mode |

#### 💡 Implementation Pointers

- `REDIS.incr("clicks:#{code}")` atomically increments — safe for concurrent requests (like an atomic counter in Go).
- `REDIS.expire("rate:#{ip}", 60)` only sets TTL if the key is **new** (check `if current == 1`), avoiding TTL refresh on every request.
- Call `FlushClicksJob.perform_async` or schedule it with a cron expression — never call `.perform_sync` in production for background work.
- For bulk DB updates, look into `Url.update_counters(id, click_count: n)` or `ActiveRecord#update_all`.
- When inserting many `clicks` rows at once, look into `Click.insert_all(array_of_hashes)` (Rails 6+) to avoid N+1 inserts.

#### 🏗️ Service Skeletons

```ruby
# app/services/click_tracker_service.rb
class ClickTrackerService
  def initialize(url, request)
    @url     = url
    @request = request
  end

  def track
    # <!-- TODO: Increment REDIS counter "clicks:{short_code}" -->
    # <!-- TODO: Enqueue a job to persist click metadata (ip, user_agent, referer) -->
    # Hint: use @request.remote_ip, @request.user_agent, @request.referer
  end
end
```

```ruby
# app/services/rate_limiter_service.rb
class RateLimiterService
  LIMIT  = 60
  WINDOW = 60 # seconds

  def initialize(ip_address)
    @key = "rate:#{ip_address}"
  end

  def allowed?
    current = REDIS.incr(@key)
    # <!-- TODO: Set TTL only on the first increment (current == 1) -->
    # <!-- TODO: Return true if current <= LIMIT, false otherwise -->
  end
end
```

```ruby
# app/jobs/flush_clicks_job.rb
class FlushClicksJob < ApplicationJob
  queue_as :default

  def perform
    # <!-- TODO: Scan Redis for all "clicks:*" keys -->
    # <!-- TODO: For each key, read count and update Url#click_count in DB -->
    # <!-- TODO: Delete the Redis key after flushing -->
    # <!-- TODO: Insert queued click metadata rows into clicks table -->
  end
end
```

> **Go→Rails note:** `goroutine` + `channel` patterns become Sidekiq jobs. Sidekiq uses Redis as its queue backend, so you already have the infrastructure. `FlushClicksJob.perform_async` is equivalent to `go func() { flushClicks() }()`.

**Deliverable ✅** Clicks are counted in Redis; every 5 minutes they are flushed to PostgreSQL.

---

### 🗓️ Day 5 — Analytics Stats API & URL Expiration

**Goal:** Build the stats API, add query scopes, set up URL expiration.

#### Tasks

| # | Task | Notes |
|---|---|---|
| 1 | Create `StatsController` | `GET /api/v1/urls/:code/stats` |
| 2 | Add scopes to `Click` model | `scope :today`, `scope :by_date`, `scope :by_country` |
| 3 | Implement stats query logic | Total clicks, clicks-per-day, top referers, top countries |
| 4 | Create `ExpireUrlsJob` | Sidekiq job to soft-delete/purge expired URLs |
| 5 | Support custom aliases | Accept optional `custom_code` param in `POST /api/v1/urls` |
| 6 | Integration tests | End-to-end: create → redirect → check stats |

#### 💡 Implementation Pointers

- Use `Click.where(url: @url).group("DATE(clicked_at)").count` for clicks-per-day — ActiveRecord translates this to SQL `GROUP BY`.
- Model `scope` is like a named, reusable `WHERE` clause: `scope :recent, -> { where("clicked_at > ?", 7.days.ago) }`.
- For expiration: `Url.where("expires_at < ?", Time.current)` selects expired URLs. Decide: hard delete or add a `deleted_at` column (soft delete with `paranoia` gem).
- When a URL is expired/deleted, also delete the Redis key: `REDIS.del("short:#{code}")`.
- Schedule `ExpireUrlsJob` using sidekiq-cron (e.g., every hour).

#### 🏗️ Stats Controller Skeleton

```ruby
# app/controllers/api/v1/stats_controller.rb
module Api
  module V1
    class StatsController < ApplicationController
      def show
        # <!-- TODO: Find Url by short_code, render 404 if not found -->
        # <!-- TODO: Build stats hash:
        #   - total_clicks: @url.click_count
        #   - clicks_by_day: Click.where(url: @url).group("DATE(clicked_at)").count
        #   - top_referers: Click.where(url: @url).group(:referer).count.sort_by { ... }.first(5)
        #   - top_countries: Click.where(url: @url).group(:country).count
        # -->
        # <!-- TODO: render json: stats_hash -->
      end
    end
  end
end
```

**Deliverable ✅** `GET /api/v1/urls/:code/stats` returns click analytics; expired URLs are purged.

---

### 🗓️ Day 6 — Hardening, Validation & DevOps

**Goal:** Harden the app against bad input, containerize it, and set up CI.

#### Tasks

| # | Task | Notes |
|---|---|---|
| 1 | URL validation hardening | Reject `localhost`, private IPs, non-HTTP(S) schemes |
| 2 | Input sanitization | Normalize URLs, strip tracking params (optional) |
| 3 | Write `Dockerfile` | Multi-stage build with `ruby:3.3-slim` |
| 4 | Update `docker-compose.yml` | Full stack: Rails + Sidekiq + PostgreSQL + Redis |
| 5 | Health check endpoint | `GET /health` — check DB and Redis reachability |
| 6 | Environment config | Use `Rails.application.credentials` for secrets, ENV for runtime config |
| 7 | GitHub Actions CI | Lint (RuboCop) + test (RSpec) on push |

#### 💡 Implementation Pointers

- Use `URI.parse(url).host` to get the hostname, then check against a blocklist (`localhost`, `127.0.0.1`, `0.0.0.0`, private CIDRs).
- For multi-stage Dockerfile: first stage installs gems (`bundle install`), second stage copies the built app — keeps the final image small.
- Health check: try `ActiveRecord::Base.connection.execute("SELECT 1")` and `REDIS.ping` — if either raises, return 503.
- `Rails.application.credentials` stores secrets in an encrypted file (`config/credentials.yml.enc`). For Docker/CI, use ENV vars via `ENV.fetch("SECRET_KEY_BASE")`.
- RuboCop config: create `.rubocop.yml` inheriting from `rubocop-rails-omakase` or set your own rules.

<details>
<summary>🔍 Hint: GitHub Actions CI structure (click to reveal)</summary>

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.3"
          bundler-cache: true
      # <!-- TODO: Set DATABASE_URL and REDIS_URL env vars -->
      # <!-- TODO: Run rails db:setup -->
      # <!-- TODO: Run bundle exec rspec -->
      # <!-- TODO: Run bundle exec rubocop -->
```

</details>

#### 🏗️ Health Check Skeleton

```ruby
# In config/routes.rb, add:
# get "/health", to: "health#show"

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def show
    checks = {}
    # <!-- TODO: Check PostgreSQL: ActiveRecord::Base.connection.execute("SELECT 1") -->
    # <!-- TODO: Check Redis: REDIS.ping -->
    # <!-- TODO: Return 200 if all checks pass, 503 if any fail -->
    render json: checks, status: :ok
  end
end
```

**Deliverable ✅** App runs in Docker; CI pipeline lints and tests on every push.

---

### 🗓️ Day 7 — Polish, Documentation & Deploy Prep

**Goal:** Document the API, add seed data, profile performance, and prepare for deployment.

#### Tasks

| # | Task | Notes |
|---|---|---|
| 1 | Update `README.md` | Setup instructions, API endpoints, architecture diagram |
| 2 | API documentation | Use `rswag` gem for OpenAPI/Swagger, or write `docs/api.md` manually |
| 3 | Seed data | `db/seeds.rb` — create sample short URLs for demo/development |
| 4 | N+1 query check | Add `bullet` gem, run test suite, fix any N+1s it detects |
| 5 | Structured logging | Add `lograge` gem — converts Rails' multi-line logs to single JSON lines |
| 6 | CORS configuration | Configure `rack-cors` in `config/initializers/cors.rb` |
| 7 | Performance profiling | Add `rack-mini-profiler` in development group |
| 8 | Final test pass | `bundle exec rspec` — all green |

#### 💡 Implementation Pointers

- `rswag` generates Swagger UI from your RSpec tests — look at `rswag-specs` and `rswag-ui` gems. Alternatively, write an `openapi.yml` manually.
- `lograge` replaces Rails' default multi-line request logs with single structured lines — much easier to parse in log aggregators (Datadog, Splunk, etc.).
- `bullet` gem logs N+1 queries in development. Add `config.after_initialize { Bullet.enable = true }` in `config/environments/development.rb`.
- For CORS: use `rack-cors` and allow only the domains that will call your API. Don't use `origins "*"` in production.
- `db/seeds.rb` can call your service objects: `UrlShortenerService.new("https://github.com").call` — seeds are just Ruby.

<details>
<summary>🔍 Hint: CORS initializer shape (click to reveal)</summary>

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    # <!-- TODO: Replace * with your actual frontend domain in production -->
    origins "*"
    resource "*",
      headers: :any,
      methods: [:get, :post, :options]
  end
end
```

</details>

**Deliverable ✅** Fully documented, all tests green, ready to `docker-compose up` in any environment.

---

## 5. 🔄 Go → Ruby on Rails Mental Model Map

| Go Concept | Ruby on Rails Equivalent | Notes |
|---|---|---|
| `main.go` / `func main()` | `bin/rails server` / `config/routes.rb` | Rails boots via Rack; routing is declarative |
| `struct { ... }` | ActiveRecord Model (`class Url < ApplicationRecord`) | Model = struct + DB persistence + validation |
| `interface` | Duck typing (no keyword needed) | If it responds to `.call`, it "implements" the interface |
| `http.HandleFunc` / `chi.Get` | Controller action (`def show`) | Each action maps to one HTTP verb + path |
| `goroutine` / `chan` | Sidekiq background job | Persistent, retryable, queue-backed async |
| `go mod` / `go.sum` | `Gemfile` / `Gemfile.lock` | `bundle install` = `go mod tidy` |
| `context.Context` | `request` object / `Current` attributes | Thread-local request context via `ActiveSupport::CurrentAttributes` |
| Explicit `(val, error)` returns | Exceptions + `rescue` blocks | Rails uses bang methods (`!`) to raise, plain methods to return nil |
| `sql.DB` / `sqlx` | ActiveRecord ORM | `Url.find_by(short_code: code)` replaces manual SQL |
| `go-redis` client | `redis-rb` gem + `REDIS` global constant | Same concepts, Ruby syntax |
| Middleware (`chi.Use`) | `before_action` / Rack middleware | `before_action` is controller-scoped; Rack middleware is app-wide |
| `_test.go` / `go test ./...` | `_spec.rb` / `bundle exec rspec` | RSpec uses `describe`/`it` blocks instead of `func TestXxx` |
| `Dockerfile` multi-stage | Same concept, `ruby:3.3-slim` base image | Rails apps are packaged the same way |
| `fmt.Println` / `log.Println` | `Rails.logger.debug` / `puts` (dev only) | Use `Rails.logger` for anything that should appear in log files |
| `os.Getenv("KEY")` | `ENV["KEY"]` / `ENV.fetch("KEY")` | `.fetch` raises `KeyError` if missing — prefer it for required vars |

---

## 6. 📦 Key Gems (Gemfile Reference)

```ruby
source "https://rubygems.org"
ruby "3.3.0"

gem "rails", "~> 7.1"
gem "pg"                        # PostgreSQL adapter — replaces database/sql + lib/pq in Go
gem "redis"                     # Redis client — replaces go-redis
gem "sidekiq"                   # Background jobs — replaces goroutines for async work
gem "sidekiq-cron"              # Cron-style recurring jobs (for FlushClicksJob, ExpireUrlsJob)
gem "rack-cors"                 # CORS headers — replaces chi/cors middleware in Go

# Serialization
gem "jsonapi-serializer"        # Fast JSON serialization — replaces encoding/json marshaling

group :development, :test do
  gem "rspec-rails"             # Test framework — replaces testing package + go test
  gem "factory_bot_rails"       # Test data factories — replaces table-driven test fixtures
  gem "faker"                   # Fake data generation — replaces test helpers that build strings
  gem "rubocop-rails", require: false  # Linter — replaces golangci-lint
  gem "rubocop-rspec", require: false  # RSpec-specific lint rules
  gem "dotenv-rails"            # Load .env file — replaces godotenv
  gem "bullet"                  # N+1 query detector — no direct Go equivalent
end

group :development do
  gem "rack-mini-profiler"      # Request profiling in dev — like pprof but for HTTP
end

group :test do
  gem "shoulda-matchers"        # One-liner model/association matchers in RSpec
  gem "webmock"                 # Stub HTTP requests in tests — replaces httpmock in Go
end
```

> **Note on `jsonapi-serializer`:** It replaces the Rails default `jbuilder` with faster, explicit serializer classes. You can also use `active_model_serializers` — pick one and stick with it.

---

## 7. ⚡ Additional Recommendations & Tips

### Use `rails console` Heavily

The Rails console (`rails c`) drops you into a live IRB session connected to your app. This is invaluable:

```bash
rails c
# Try things interactively:
# > UrlShortenerService.new("https://example.com").call
# > Url.all.to_sql
# > REDIS.ping
```

Think of it as Go's playground, but connected to your *actual* running database.

### Debug with `binding.irb`

Drop `binding.irb` anywhere in your code to pause execution and open an interactive console at that exact point — like adding a breakpoint. Much more powerful than `fmt.Println` debugging.

```ruby
def create
  binding.irb  # execution pauses here; type any Ruby expression
  # ...
end
```

### Trust the Service Object Pattern

Rails' default "fat model, skinny controller" advice can lead to bloated models. As a Go developer, you're already wired to think in focused functions. Service objects map naturally to that:

- One service = one piece of business logic
- Plain Ruby class (no inheritance needed)
- `call` method as the entry point
- Easy to test in isolation

### Learn ActiveRecord Incrementally

Don't write raw SQL. Instead, write ActiveRecord queries and use `.to_sql` to see what SQL it generates:

```ruby
Click.where(url: @url).group("DATE(clicked_at)").count.to_sql
# => "SELECT COUNT(*) ... GROUP BY DATE(clicked_at)"
```

This bridges the mental gap between Go's explicit SQL and Rails' ORM.

### Week 2 Stretch Goals

Once the core app is working, consider these extensions:

- 🔐 **Authentication** — Add `devise-jwt` or `rodauth` for user accounts; let users manage their own URLs.
- 📊 **Real-time analytics** — Add `ActionCable` (Rails WebSockets) to push click counts to a dashboard in real time.
- 🌍 **GeoIP lookup** — Use `maxminddb` gem to resolve `ip_address` to country in `ClickTrackerService`.
- 🔗 **QR code generation** — Use `rqrcode` gem to return a QR code image for each short URL.
- 📈 **Metrics & observability** — Add `prometheus-client` gem and expose `/metrics` for Prometheus scraping.
- 🚦 **Tiered rate limiting** — Authenticated users get higher limits than anonymous users.
- 🗑️ **Soft delete** — Add `paranoia` or `discard` gem to `Url` for recoverable deletes.

---

> **Final tip:** Rails is opinionated. When you're fighting the framework, you're usually doing it wrong. Lean into conventions, read error messages carefully (they're excellent), and use `rails console` to experiment before writing production code. You'll be productive faster than you think. 🚀
