# Dimension E — Persistence, SQLite, Caching State

> **Era note:** Campfire 2024 hand-patched SQLite for production (busy-handler retries in
> `config/initializers/sqlite3.rb`, `retries:`/`default_transaction_mode: immediate` in
> `database.yml`) and used Redis for cache/cable. **Modern Rails (7.2+/8) ships the SQLite
> tuning out of the box** and the Solid trifecta (Queue/Cache/Cable) makes the database the
> backing store for everything. Extract the philosophy below; do not port the 2024 patches —
> check whether current Rails already does it first.

## Pattern: SQLite is a production database — and one volume is an ops strategy

**One-line rule:** For single-server apps, run SQLite in production with WAL-mode defaults and `IMMEDIATE` transactions, and co-locate the database and file storage under one directory (`storage/`) so backup = copy one volume.

**Why 37signals does it:** "SQLite is good, actually" — their words, in the config (`production.rb:68-69`, disabling the framework's own warning). No connection pool service, no managed-DB bill, no network hop, and reads measured in microseconds. Campfire keeps `storage/db/production.sqlite3` and `storage/files` (Active Storage) side by side: one volume mount, one backup target, one thing to move. This is the persistence half of the one-container deployment story.

**Citations:**
- `config/environments/production.rb:68-69` — `# SQLite is good, actually` / `sqlite3_production_warning = false`
- `config/database.yml:7-10` — `retries: 100`, `default_transaction_mode: immediate` (busy-wait + write-lock-upfront: the two classic SQLite-under-concurrency fixes; **now built into modern Rails defaults**)
- `config/initializers/sqlite3.rb:1-14` — backoff busy_handler adapter patch (same: superseded by the framework — delete on upgrade)
- `config/database.yml:13-31` — every env's DB lives under `storage/db/`
- `config/initializers/storage_paths.rb:1-5` — boot-time `mkpath` for `storage/{db,files}`
- `app/controllers/users_controller.rb:15` — designing for SQLite's strengths: rescue the constraint, don't pre-check with a racy SELECT

**Complete example** (`config/database.yml`, the part that matters):

```yaml
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 10 } %>
  retries: 100
  default_transaction_mode: immediate

production:
  primary:
    <<: *default
    database: storage/db/production.sqlite3
```

`immediate` mode takes the write lock at BEGIN instead of at first write — converting mid-transaction `SQLITE_BUSY` deadlocks into orderly queueing. (On current Rails, verify before adding: this became default behavior.)

**Counter-example:** Reflexively provisioning Postgres + Redis + a cache cluster for an app with one web box and a few thousand users — three services to operate, secure, and pay for, replacing files the OS already knows how to back up.

**When to break the rule:** Multi-node web tiers (SQLite is single-host), heavy concurrent *write* workloads, or genuine need for Postgres features (logical replication, advisory locks, extensions). Litestream/Litestack-style continuous replication mitigates the durability story but not multi-writer. Choose per app — the point is that "boring default = Postgres cluster" is no longer automatic.

**Detection heuristic:** `database.yml` pointing SQLite at the repo root or `db/` (ephemeral in containers — must live on the mounted volume); production SQLite without WAL/immediate/timeout settings on pre-7.2 Rails; Active Storage `local` service on a *different* volume than the DB (two backup targets).

---

## Pattern: Full-text search is an FTS5 table maintained by model callbacks

**One-line rule:** Build search on the database you already run — a SQLite FTS5 virtual table, kept in sync with `after_*_commit` callbacks writing raw SQL, exposed as a model scope, with `schema_format = :sql` so the virtual table survives schema dumps.

**Why 37signals does it:** Search is the textbook excuse to add Elasticsearch; Campfire does it in 28 lines and zero services. The index is denormalized plain text (`plain_text_body`), updated transactionally-adjacent via commit callbacks, joined by rowid. Porter stemming comes free. The same shape works on Postgres (tsvector column + GIN) — the pattern is *search lives in the primary database until proven otherwise*.

**Citations:**
- `app/models/message/searchable.rb:1-28` — the entire search implementation
- `db/structure.sql:22-23` — `CREATE VIRTUAL TABLE message_search_index using fts5(body, tokenize=porter)`
- `config/application.rb:15-16` — `# Use SQL schema format to include search-related objects` / `schema_format = :sql`
- `app/controllers/searches_controller.rb:23` — consumption: `Current.user.reachable_messages.search(query).last(100)` (search composes with the authorization scope!)
- `app/controllers/searches_controller.rb:30` — input hygiene: `params[:q]&.gsub(/[^[:word:]]/, " ")` neutralizes FTS5 query operators

**Complete example** (`app/models/message/searchable.rb`, entire file):

```ruby
module Message::Searchable
  extend ActiveSupport::Concern

  included do
    after_create_commit  :create_in_index
    after_update_commit  :update_in_index
    after_destroy_commit :remove_from_index

    scope :search, ->(query) { joins("join message_search_index idx on messages.id = idx.rowid").where("idx.body match ?", query).ordered }
  end

  private
    def create_in_index
      execute_sql_with_binds "insert into message_search_index(rowid, body) values (?, ?)", id, plain_text_body
    end

    def update_in_index
      execute_sql_with_binds "update message_search_index set body = ? where rowid = ?", plain_text_body, id
    end

    def remove_from_index
      execute_sql_with_binds "delete from message_search_index where rowid = ?", id
    end

    def execute_sql_with_binds(*statement)
      self.class.connection.execute self.class.sanitize_sql(statement)
    end
end
```

**Counter-example:** Elasticsearch/OpenSearch + an indexing pipeline + sync drift + a second query language, for sub-million-row corpora a database index handles in single-digit milliseconds.

**When to break the rule:** Faceting, relevance tuning, typo tolerance, or >10M documents are real search-engine territory. Commit callbacks can drift on crashes between commit and callback — acceptable for chat search; add a reindex task (or triggers) where staleness is costly.

**Detection heuristic:** Search gems/services in apps with one searchable model; `LIKE '%term%'` scans where an FTS index belongs; `schema_format` left as `:ruby` while raw-SQL database objects silently vanish from `schema.rb`.

---

## Pattern: Columns before stores — keep ephemeral state in the database you have

**One-line rule:** Before adding Redis (or any side-store) for presence/unread/last-seen state, model it as columns with TTL semantics expressed in scopes — and let *reads* expire state passively instead of running reapers.

**Why 37signals does it:** Presence — the canonical "you need Redis" feature — is two columns on `memberships` (`connections` int, `connected_at` datetime) plus range scopes. A connection bump is one `update_all`; "who's connected" is `WHERE connected_at >= now - 60s`; crashed clients expire by *being ignored*, not by cleanup jobs. State stays joinable (`Push::Subscription.joins(user: :memberships).merge(Membership.visible.disconnected...)`) — try that against a Redis set. The 2024 snapshot did keep Redis for cache/cable plumbing; with Solid Cache/Cable that's now also just the database — the philosophy completed itself.

**Citations:**
- `app/models/membership/connectable.rb:4-50` — TTL presence in 50 lines (`CONNECTION_TTL = 60.seconds`, `scope :connected/:disconnected`, counter inc/dec)
- `app/channels/presence_channel.rb:5-17` — channel lifecycle drives it: `on_subscribe :present`, `on_unsubscribe :absent`, periodic `refresh`
- `app/javascript/controllers/presence_controller.js:5` — client heartbeat every 50s against the 60s TTL (heartbeat < TTL: the whole protocol)
- `app/models/room.rb:69` — unread state: `memberships.visible.disconnected.where.not(user: message.creator).update_all(unread_at: ...)` — presence and unread *compose in one query*
- `app/models/room/message_pusher.rb:56-60` — push targeting joins presence: only disconnected members get notified
- `db/structure.sql:53` — the columns, with sane defaults

**Complete example** — the composition payoff (one query answers "who should be notified"):

```ruby
# app/models/room/message_pusher.rb
def relevant_subscriptions
  Push::Subscription
    .joins(user: :memberships)
    .merge(Membership.visible.disconnected.where(room: room).where.not(user: message.creator))
end
```

Presence (`disconnected`), preferences (`visible`), and authorship exclusion combine relationally because presence is rows, not keys in a side-store.

**Counter-example:** `Redis.sadd("room:#{id}:online", user_id)` with EXPIRE — now presence can't join against subscriptions or involvement, needs its own serialization, its own failure handling, and a second mental model.

**When to break the rule:** Write rates beyond what row updates absorb (tens of thousands of presence flips/sec), or genuinely ephemeral coordination (distributed locks, leaderboards with atomic ZINCRBY semantics). Kredis exists for when you do want Redis structures with Rails ergonomics — Campfire includes it but barely leans on it; that restraint is the pattern.

**Detection heuristic:** Redis keys encoding per-record state that controllers then re-fetch row-by-row; cron jobs whose only purpose is expiring state a TTL-range scope could ignore; "online" booleans flipped by reapers.
