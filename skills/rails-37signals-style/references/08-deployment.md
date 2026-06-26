# Dimension H — Deployment & Ops

> **Era + scope note:** the audited snapshot is the ONCE distribution — a product sold as
> a single Docker image customers run themselves. There is **no Kamal in this codebase**;
> ONCE's "installer + one container" model is its own thing. For your apps in 2026, the
> equivalent stack is what Rails 8 generates: a Dockerfile with **Thruster** (the same
> proxy Campfire pioneered — it's literally vendored here as Go source) and **Kamal 2**
> for pushing it to servers. The transferable material below is the philosophy: one
> artifact, one volume, ENV-only config, an idempotent bin/setup. Treat anything
> tool-specific here as historical.

## Pattern: The whole app is one self-contained artifact on one volume

**One-line rule:** Ship a single container that includes everything the app needs to run — web server behind an in-image HTTP/2 proxy (Thruster), workers, even auxiliary services — supervised by a minimal process monitor, with all mutable state under one mounted `storage/` volume.

**Why 37signals does it:** ONCE's promise is "run this image, own your data" — no compose file, no managed services, no orchestration. Campfire's container runs Thruster→Puma, Redis, and the Resque pool from a Procfile, supervised by a **60-line hand-written Ruby ProcessMonitor** (`bin/boot`) instead of foreman/overmind — signal forwarding and child reaping is all that's actually needed, so that's all that exists. SQLite plus `storage/{db,files}` co-location (see `05-persistence-sqlite.md`) makes backup = snapshot one volume. The Dockerfile is the standard Rails multi-stage shape: gems cached before `COPY . .`, assets precompiled with `SECRET_KEY_BASE_DUMMY=1`, non-root `rails` user, final image carrying only runtime libs.

**Citations:**
- `Procfile:1-3` — `web: bin/thrust bin/start-app`, `redis:`, `workers:` — the whole topology in three lines
- `bin/boot:1-60` — ProcessMonitor: spawn each Procfile entry, trap INT/TERM/CLD, TERM children, wait
- `Dockerfile:17-33` — throw-away build stage; `COPY Gemfile Gemfile.lock ./` then `bundle install` *before* `COPY . .` (layer-cache ordering); `SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile`
- `Dockerfile:36-42` — Thruster compiled from vendored source `packaging/thruster/` (it later became a 37signals gem and the Rails 8 Dockerfile default)
- `Dockerfile:58-60` — `useradd rails` / `USER rails:rails`
- `Dockerfile:67-71` — `APP_VERSION`/`GIT_REVISION` build args → ENV → surfaced as `X-Version`/`X-Rev` response headers (`app/controllers/concerns/version_headers.rb`)
- `config/routes.rb:96` — `get "up" => "rails/health#show"` (the stock health check, kept)
- `config/environments/production.rb:33-39` — log to STDOUT, tagged with `request_id`; Sentry for errors (`config/initializers/sentry.rb`)

**Complete example** (`Procfile` + the supervisor entry point):

```
web: bin/thrust bin/start-app
redis: redis-server config/redis.conf
workers: FORK_PER_JOB=false INTERVAL=0.1 bundle exec resque-pool
```

```ruby
# bin/boot (core of the 60-line ProcessMonitor)
class ProcessMonitor
  SIGNALS = %w[ INT TERM CLD ]

  def initialize(procfile)
    @procs = process_list(procfile)
    handle_signals

    @procs.each &:start
    @procs.each &:wait
  end
end
```

(2026 translation: `rails new` gives you the Thruster Dockerfile; Solid Queue runs in-process via the Puma plugin or as a separate role in Kamal — you likely need *no* Procfile supervisor and *no* Redis at all. The surviving rule: everything the app needs rides in the image; everything it mutates lives on one volume.)

**Counter-example:** A docker-compose of app + sidekiq + redis + postgres + nginx for a team of two — five services, five upgrade treadmills, and state scattered across three volumes; or config split between ENV, mounted YAMLs, and a credentials file that the image can't build without.

**When to break the rule:** Real horizontal scale separates web and worker containers (Kamal roles make this cheap). Multi-process-in-one-container is a deliberate simplicity/orthodoxy tradeoff — fine for self-hosted products and tiny ops footprints; choose knowingly. Also note the ENV-over-credentials posture throughout (`ENV.fetch("VAPID_PRIVATE_KEY", credentials fallback)` in `config/initializers/vapid.rb`; `ENV["DISABLE_SSL"]` toggling `force_ssl` in `production.rb:59-60`) — image-portable config, credentials only as fallback.

**Detection heuristic:** Dockerfiles copying the app before installing gems (cache-busted builds); images that can't precompile without production secrets (missing `SECRET_KEY_BASE_DUMMY`); containers running as root; mutable state outside the declared volume; health checks absent or hand-rolled where `rails/health#show` exists.

---

## Pattern: bin/setup is the onboarding contract — idempotent, resettable, self-verifying

**One-line rule:** Maintain a `bin/setup` that takes a fresh clone to a working app in one command — idempotent (`bundle check || bundle install`, `db:prepare`), with a `--reset` flag for nuking state, environment guards, and a final *proof* (curl the health endpoint) — and seed development through a realistic `script/dev/populate` rather than `db/seeds.rb`.

**Why 37signals does it:** Setup scripts that "usually work" cost every new machine an afternoon. Campfire's announces each step, refuses to run in production, wires up the local TLS dev server (puma-dev symlink), and ends by asserting the app actually responds — setup isn't done until proven. There's no `db/seeds.rb` at all; development data comes from `script/dev/populate` (Faker-driven, dev-only by placement), and operator tasks live in `script/admin/` (`reset-password`, `prepare-backup`) — runnable scripts instead of a wiki page.

**Citations:**
- `bin/setup:14-18` — `if [ "$RAILS_ENV" == "production" ]; then echo "...bailing out"; exit; fi`
- `bin/setup:20-22` — `bundle check || bundle install` (idempotence)
- `bin/setup:24-30` — `--reset` flag: `rm -rf ./storage/{db,files}` + `db:migrate:reset`, else `db:prepare`
- `bin/setup:38-43` — puma-dev symlink + `rails restart` + final `curl -Is "https://campfire.test/up" | head -n 1`
- `script/dev/{populate,flood-room}` — seed and load-test scripts, separate from the app
- `script/admin/{reset-password,prepare-backup}` — ops runbook as executables
- `db/migrate/` — note `20231215043540_create_initial_schema.rb`: they **squash migrations**; history lives in git, the schema file is the truth

**Complete example** (`bin/setup`, the spine):

```bash
#!/bin/bash
set -eo pipefail

if [ "$RAILS_ENV" == "production" ]
then
  echo "RAILS_ENV is production; bailing out"
  exit
fi

announce "Installing dependencies"
bundle check || bundle install && rbenv rehash

announce "Preparing database"
if [[ $* == *--reset* ]]; then
  rm -rf ./storage/{db,files}
  rails db:migrate:reset
else
  rails db:prepare
fi

announce "Checking that https://$app_name.test/up is live: "
curl -Is "https://$app_name.test/up" | head -n 1
```

**Counter-example:** A README with 14 numbered steps, three of which are stale; `db/seeds.rb` that half the team is afraid to run because it might fire callbacks against shared services; setup that exits 0 having silently failed at step 4.

**When to break the rule:** Teams not on puma-dev swap that step for their local-TLS approach (or `bin/dev`); the contract — one command, idempotent, resettable, ends with proof — is the part to keep. Migration squashing needs care on long-lived branches; do it at quiet moments.

**Detection heuristic:** No `bin/setup`, or one that isn't idempotent (fails on second run); seeds mixing dev fixtures with production-required data; onboarding docs describing what a script should be doing; setup scripts with no final verification step.
