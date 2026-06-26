# Rails Idiom Detection Matrix (Layer 2)

Every row maps a **smell** in the diff to the **preferred 37signals idiom**, a default
**severity**, and the **card to cite**. Cards live in
`~/.claude/skills/rails-37signals-style/references/`. **Always read the cited card's
_When to break the rule_ section before flagging** — many rows have legitimate exceptions,
and the severities below assume no such exception applies. Drop a finding (or downgrade to
LOW with a note) when the card's caveats fit the code.

Grep commands assume you're at the repo root. They surface *candidates*, not verdicts.

---

## A — Models & domain → cite `01-models.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| Service/interactor for a single-aggregate write — `rg -n "def self.call|class \w+(Service|Interactor|Operation)"` ; `app/services/` | MEDIUM | Named class-method constructor on the model owning the transaction (`Room.create_for`), or an instance method. |
| Collection mutation in a service or controller loop — `rg -n "\.each.*\.create!"` | MEDIUM | `has_many` association extension (`room.memberships.grant_to(users)`). |
| Multi-collaborator logic that resists a model home | LOW | Keep it, but rename to a **noun**, namespace under its model, move to `app/models/` (`Room::MessagePusher`). |
| Concern in generic `app/models/concerns/` included by one model | LOW | Namespace it under the model (`app/models/user/role.rb`, `include Role`). |
| Model past ~80 lines with no own-namespace `include` — check file length | MEDIUM | Extract cohesive facets into model-namespaced concerns (readability, not reuse). |
| `current_user`/`user:` parameter threaded only to re-attribute records | MEDIUM | `Current.user` + `belongs_to :creator, class_name: "User", default: -> { Current.user }`. (Exception: job/console paths — pass explicitly.) |
| `belongs_to :user` where the role is creator/author/booster | LOW | Role-named FK: `creator_id`, `booster_id`. |
| `validates … uniqueness: true` with no matching unique index | HIGH | Unique index is the enforcement; rescue `ActiveRecord::RecordNotUnique` at the controller. |
| Model with 10+ validations mirroring NOT NULL columns | MEDIUM | Trust the DB; reserve `validates` for untrusted external input / form-UX. |
| Uniform `dependent: :destroy` everywhere, or creation loops | MEDIUM | Choose per association: `:delete_all` when no callbacks matter, `:destroy` when they do; `insert_all`/`update_all` for bulk. |
| `update_all`/`insert_all` **missing** `updated_at:` | MEDIUM | Set `updated_at` explicitly — bulk writes skip touch, staling fragment caches. |
| Callback body >1 statement, or business logic inline | MEDIUM | One-line lambda/method delegating to a named domain method. |
| `after_save`/`after_create` (not `_commit`) enqueuing jobs / broadcasting / HTTP | MEDIUM | Use `after_*_commit` for external effects. |
| `case`/`when` on a `*_type`/`kind` column in 3+ places | MEDIUM | STI subclasses + predicate methods; callers call the method, never switch on type. |
| Boolean `is_*`/`*_flag` column that always flips with a timestamp | LOW | Model state as a nullable `_at` timestamp (queryable + dated); derive predicate/scope. |
| Redis for per-record state (presence, flags, last-seen) | MEDIUM | Columns + TTL-range scopes; let reads expire state passively. (See also dimension E.) |

## B — Controllers & routes → cite `02-controllers.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| `member do` / `collection do` / non-REST action — `rg -n "member do|collection do"` ; routes with custom verbs | MEDIUM | Name the hidden resource; new 7-action controller (`resource :involvement`). |
| Controller with >7 public actions, or actions branching on a type param | MEDIUM | Subclass controller per variant; override private hooks (params/guards). |
| `ApplicationController` > ~30 lines / fat with private methods | MEDIUM | Reduce to an include-list of small named concerns. |
| `before_action` method not `set_*`/`ensure_*`/`verify_*`-shaped, or >5 lines | LOW | `set_*` looks up, `ensure_*` guards (`head :forbidden`); scope with `only:`/`except:`. |
| before_action computing view data / sending mail / building forms | LOW | That's a method call inside the action, not a filter. |
| `Model.find(params[:id])` on user-owned resource — `rg -n "\b[A-Z]\w+\.find\(params"` | HIGH | Scope through the user's associations (`Current.user.rooms.find…`) so unauthorized = unfindable. **Security.** |
| `gem "devise"` using ~2 modules; `session[:user_id]` with no Session model | MEDIUM | Rails 8 auth generator; preserve the shape (Session record + signed httponly cookie + `Current` + declarative class methods). Don't hand-roll. |
| JSON error envelopes / numeric status codes in an HTML app | LOW | `head :forbidden`/`:no_content`; `fresh_when` for conditional GET; `redirect_to … notice:`. |
| Versioned/parallel `Api::` controllers duplicating a resource for one integration | MEDIUM | Subclass the controller (`Messages::ByBotsController < MessagesController`), override params. |

## C — Views, Hotwire, Stimulus, CSS → cite `03-hotwire-views.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| `broadcasts_to` / `broadcast_*` inside model callbacks — `rg -n "broadcast" app/models` | MEDIUM | Broadcast explicitly from the controller; keep vocabulary in a `Model::Broadcasts` concern. Models keep durable effects (unread, push), not DOM delivery. |
| Interpolated DOM id / target — `rg -n "id=.*#\{|target: .#\{|\"\w+_#\{"` | MEDIUM | `dom_id(record, :purpose)` on the render side AND `target: [record, :purpose]` on the broadcast side. |
| Interpolated/ad-hoc stream names (`turbo_stream_from "room_#{id}"`) | MEDIUM | `[parent, :collection]` convention (`turbo_stream_from @room, :messages`). |
| Multi-attribute `data-*` Stimulus wiring pasted inline in 2+ templates | LOW | Wrap in a tag-builder helper (`message_tag`), one Ruby method owns the contract. |
| Stimulus controller >150 lines / page-named / `querySelector` across boundaries | MEDIUM | Small single-purpose controllers via `static values/targets/outlets`; extract logic to `javascript/models/`; cross-talk via `dispatch`/outlets. |
| Record partial without `cache record`, or collection render without `cached: true` | LOW | Russian-doll fragment caching; ensure the `touch: true` chain + explicit `updated_at` on bulk writes. |
| Hex colors inline in component CSS; one giant `application.css`; per-component dark mode | LOW | Two-layer custom props (`--lch-*` raw → `--color-*` semantic), dark mode redefines raw layer only; one file per component. (Apply token-layering even inside Tailwind.) |
| Duplicate-element bugs after stream broadcasts | MEDIUM | Shared client identity (`client_message_id` → `to_key` → stable `dom_id`) for optimistic UI. |

## D — Jobs & background work → cite `04-jobs.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| Job `perform` with real logic / branching — read `app/jobs/*` | MEDIUM | Three-line transport delegating to a domain method; logic lives on the model/PORO. |
| Model behavior reachable only via a job (no synchronous counterpart) | MEDIUM | Expose sync (`deliver_webhook`) + async (`deliver_webhook_later`) pair; default sync. |
| `perform_later` for trivial DB writes / non-external work | LOW | Enqueue only slow, fan-out, or failure-isolated external I/O. |
| Flat `app/jobs` of verb-named jobs | LOW | Namespace jobs under their domain owner (`Room::PushMessageJob`, `Bot::WebhookJob`). |
| New Sidekiq/Resque/Redis dependency in a fresh app | LOW | Solid Queue (Rails 8 default); the job *shape* is adapter-agnostic. |

## E — Persistence, SQLite, search → cite `05-persistence-sqlite.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| New search gem / Elasticsearch for one searchable model; `LIKE '%…%'` scans | MEDIUM | FTS index in the primary DB (SQLite FTS5 / PG tsvector), kept by commit callbacks, exposed as a scope; `schema_format = :sql` if raw-SQL objects exist. |
| Redis keys encoding per-record state then re-fetched row-by-row; reaper crons | MEDIUM | Columns + TTL-range scopes that compose relationally; passive expiry, no reaper. |
| New Postgres+Redis+cache-cluster stack for a single-box, low-traffic app | LOW | Consider SQLite + Solid trifecta; back up one volume. (Not for multi-node/write-heavy.) |
| Raw-SQL DB objects (FTS, triggers) with `schema_format = :ruby` | MEDIUM | Switch to `:sql` or they vanish from the schema dump. |
| SQLite DB path at repo root/`db/` (ephemeral in containers) | MEDIUM | Put DB + Active Storage on one mounted `storage/` volume. |
| 2024-era hand-patched SQLite tuning (busy_handler, `retries:`) being *added* today | LOW | Modern Rails ships this — verify before adding; delete on upgrade. |

## F — Testing → cite `06-testing-minitest.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| `gem "factory_bot"`/`factory :` in a team that controls its conventions | MEDIUM | Fixtures-as-world: a small named cast, ERB for digests/relative times, `fixtures :all`. |
| Mocks/stubs of **app-internal** classes — `rg -n "expects\(|\.stub|allow\(.*\)\.to receive"` | MEDIUM | Real objects + fixtures; assert at the integration seam; mock only true boundaries. |
| Auth stubbing / `session[:user_id] = …` backdoor in integration tests | MEDIUM | `sign_in` helper hitting the real endpoint, asserting the cookie. |
| Macro/ceremony tests (`should belong_to`, validations validate) | LOW | Delete; test domain methods, response semantics, side effects, boundaries. |
| New real-time feature with no `using_session` system test | MEDIUM | One multi-session Capybara test for the money path; wait on signals, not `sleep`. |
| Many system tests asserting what an integration test could (status, creation) | MEDIUM | Push coverage down to the integration layer; reserve system tests for journeys. |
| Fixtures named `user1`/`test_user`; absolute timestamps | LOW | Named cast with relative times (`<%= 2.days.ago %>`). |
| `sleep` in Capybara tests | MEDIUM | Condition-based waits (`assert_selector … wait:`, `wait_for_cable_connection`). |

## G — Security → cite `07-security.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| `Net::HTTP`/`open-uri`/`Faraday`/`HTTParty` on a user-supplied/derived URL — `rg -n "Net::HTTP|open-uri|Faraday|HTTParty|URI.open"` | **CRITICAL** | SSRF guard: resolve + reject private/loopback IPs, **re-validate every redirect hop**, cap redirects, allowlist content-type, stream with a max-body cap, fail closed. |
| Redirect-following client on user URLs that checks the IP only once | HIGH | Re-run the guard per hop (302 → metadata endpoint defeats a one-time check). |
| Hand-rolled token tables / `SecureRandom.hex` + manual expiry; `JWT.encode` for first-party sessions | HIGH | `has_secure_token` (DB), `signed_id(purpose:, expires_in:)` (links), `cookies.signed` (client). Always set `purpose:`. |
| Secret compared with `==` | HIGH | `ActiveSupport::SecurityUtils.secure_compare` or an AR finder. |
| `skip_before_action :verify_authenticity_token` broader than one declared auth path | HIGH | Scope CSRF exemption to exactly the token-auth path (`unless: -> { authenticated_by.bot_key? }`). |
| Auth added per-controller (`before_action :authenticate!`) instead of inherited | MEDIUM | Deny-by-default in a concern; declare exceptions (`allow_unauthenticated_access only:`). |
| No `rate_limit` on session/password/code-accepting endpoints | MEDIUM | `rate_limit to:, within:` on the credential-guessing action. |
| New unauthenticated endpoint | MEDIUM | Declare via the auth concern's class methods so the public surface stays grep-able. |
| CSP absent with no recorded reason | LOW | Enable a CSP for normal SaaS (Campfire's commented-out CSP is a self-hosted exception). |

## H — Deployment & ops → cite `08-deployment.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| Dockerfile copying app before installing gems | LOW | `COPY Gemfile* ./ && bundle install` before `COPY . .` (layer cache). |
| Asset precompile needing real secrets | LOW | `SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile`. |
| Container running as root; mutable state outside the declared volume | MEDIUM | Non-root `rails` user; all mutable state on one mounted volume. |
| Hand-rolled health check where `rails/health#show` exists | LOW | Keep `GET /up`. |
| No `bin/setup`, or one that isn't idempotent / has no final verification | LOW | Idempotent `bin/setup`, `--reset` flag, refuses production, ends with a health-check curl. |
| `db/seeds.rb` mixing dev fixtures with production-required data | LOW | Dev data in `script/dev/populate`; ops tasks as `script/admin/*` executables. |

## I — Code style & conventions → cite `09-conventions.md`

| Smell (grep) | Sev | Preferred idiom |
| --- | --- | --- |
| Comment density >1/20 lines; comments restating the next line | LOW | Comment only surprises (non-obvious *why*, tradeoff, FIXME + reasoning). |
| FIXME/TODO with no reasoning or condition | LOW | Attach the analysis/threshold so the future fixer starts ahead. |
| `private` methods interleaved with public; `private :method` inline | LOW | One indented `private` basement; macros → scopes → class methods → public → private. |
| `.rubocop.yml` >~20 lines of style overrides | LOW | Inherit `rubocop-rails-omakase`; layer project rules below the inherit line. |
| Section comment banners (`# == Associations ==`) | LOW | Let position + blank-line grouping organize. |
| `get_`/`is_`/`handle_`/`process_` names; boolean methods without `?` | LOW | Predicate `?`, adjective/participle scopes (`ordered`, `without_bots`), `_later` async pairs. |
| Scopes named after their WHERE clause (`status_eq_active`) | LOW | Domain adjectives that read as English. |
| Manual code where ActiveSupport has a word — `rg -n "each_with_object\(\{\}|reject\(&:blank|\.map.*\.to_h"` | LOW | `index_by`, `compact_blank`, `presence_in`, `including`/`excluding`/`without`. |
| `Gemfile` >~2 minors behind; `lib/` utils wrapping shipped framework features | LOW | Stay current; delete workarounds the framework has absorbed. |
| Monkeypatches hidden in random initializers | LOW | One `lib/rails_ext/` dir auto-required by an initializer; comment + upstream pointer. |

---

## Reviewer discipline (read before reporting)

1. **Two questions catch most violations:** "*Which model owns this?*" (if the answer is a
   service, a job, or "the controller," keep asking) and "*What happens when this
   renders/broadcasts/raises twice?*" (idempotency, dom_id symmetry, constraint-backed
   uniqueness).
2. **Proportionality.** This pattern library was extracted from a small app (one account,
   two roles). Several cards say so. Don't flag a policy-object layer as wrong if the app has
   a real role matrix; don't demand SQLite; don't insist on vanilla CSS over working Tailwind.
   Flag the diff, cite the rule, respect the caveat.
3. **Layer routing.** If a smell is *also* a correctness/security risk, report it in Layer 1
   with the higher severity — Layer 2 is for "works, but not idiomatic," not for risks.
