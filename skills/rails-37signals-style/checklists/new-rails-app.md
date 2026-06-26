# Checklist — Boot a New Rails App in 37signals Style

Target: a Rails 8+ app that starts where Campfire's philosophy ends up, using current
framework defaults instead of 2024-era workarounds.

## 1. Generate

```bash
rails new myapp \
  --database=sqlite3 \
  --skip-jbuilder \
  --skip-action-mailbox \
  # keep: propshaft, importmap, turbo, stimulus, solid trifecta, kamal, thruster, brakeman, rubocop-omakase — all defaults
cd myapp
bin/rails generate authentication   # Session model + concern, Campfire-shaped
```

- [ ] SQLite for all environments unless you already know why not (multi-node? heavy concurrent writes?)
- [ ] Keep the Solid trifecta (Queue/Cache/Cable) — do NOT add Redis/Sidekiq reflexively
- [ ] Keep importmap unless you have a hard dependency that demands bundling
- [ ] `.rubocop.yml` stays at the omakase inherit line — no style overrides on day one

## 2. Gemfile discipline

- [ ] Audit every gem you're tempted to add against the framework: Devise→generator,
      Pundit→`can_administer?`+scoping (until role matrix demands more), Kaminari→geared_pagination
      or LIMIT/OFFSET scopes, FactoryBot→fixtures, RSpec→Minitest, Sidekiq→Solid Queue,
      friendly_id→`to_param`, paper_trail→an `*_at`/`*_by` column, dotenv→Rails credentials/ENV
- [ ] Target ≤ ~35 gems including dev/test (Campfire ships a whole product on 30)
- [ ] Add: `bcrypt` (generator wants it), an error reporter (Sentry/Honeybadger), `web-push` only if PWA

## 3. Structure (create these now, so the defaults have homes)

- [ ] `app/models/<model>/` directories for concerns as soon as a model has a second facet
- [ ] NO `app/services`, `app/forms`, `app/queries`, `app/presenters` — if a PR introduces one, that's a review conversation
- [ ] `lib/rails_ext/` + the auto-require initializer (copy from `09-conventions.md`)
- [ ] `test/test_helpers/` with `SessionTestHelper#sign_in` hitting the real endpoint
- [ ] `script/dev/populate` for development data (no `db/seeds.rb` for fake data)
- [ ] `script/admin/` for operator tasks as they appear

## 4. Database conventions (enforce from migration #1)

- [ ] NOT NULL on everything that's conceptually required; unique indexes for every uniqueness rule
- [ ] Role-named FKs: `creator_id`, `author_id`, `approver_id` — `user_id` only when the user is just "the user"
- [ ] State as `_at` timestamps (`archived_at`, not `archived: boolean`) or counters — see `01-models.md`
- [ ] `rescue ActiveRecord::RecordNotUnique` at the controller where races are user-visible
- [ ] If you add raw-SQL objects (FTS5, triggers): `config.active_record.schema_format = :sql`

## 5. App skeleton

- [ ] `Current` with `attribute :user` (+ `:account` etc.); set only in the auth concern
- [ ] `belongs_to :creator, class_name: "User", default: -> { Current.user }` on user-attributed models
- [ ] ApplicationController = include-list of concerns only
- [ ] First controllers: resources only; the moment a verb appears, noun it (see `02-controllers.md`)
- [ ] Layout: `content_for` slots (`:nav`, `:sidebar`, `:footer`), `@page_title`/`@body_class` ivars
- [ ] CSS: `colors.css` two-layer tokens + `utilities.css` + one file per component; `stylesheet_link_tag :all`

## 6. Fixtures cast (before the third test exists)

- [ ] Name 4-6 people with distinct roles; one shared password constant via ERB
- [ ] Relative timestamps (`<%= 2.days.ago %>`) wherever time matters
- [ ] `fixtures :all` in test_helper; include your test_helpers modules there too

## 7. Ops

- [ ] `bin/setup`: idempotent, `--reset` flag, refuses production, ends with a health-check curl
- [ ] Keep the generated Dockerfile (Thruster included) and `config/deploy.yml` (Kamal 2); mutable state on one volume; for SQLite ensure the DB path is on that volume
- [ ] Log to STDOUT tagged with request_id; wire the error reporter; keep `GET /up`
- [ ] CI: `bin/rubocop`, `bin/brakeman`, `bin/rails test test:system` — nothing more until it earns its place

## 8. First-week tripwires (where new apps drift off-style)

- [ ] Someone adds a `*Service` → redirect to model method / association extension / noun-PORO
- [ ] Someone adds `validates ... uniqueness:` → make sure the index exists; consider dropping the validation
- [ ] Someone writes `broadcasts_to` on a model → move delivery to the controller
- [ ] Someone reaches for Redis "for presence/flags" → two columns and a TTL scope first
- [ ] Test suite grows mock-heavy → re-anchor on fixtures + integration seam
