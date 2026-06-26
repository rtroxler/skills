# Checklist — Migrate an Existing App Toward 37signals Style

Incremental, value-ordered, reversible steps. Never a big-bang rewrite; each phase ships
independently. Skip phases that don't pay for themselves in *your* app — this style's
first law is proportionality.

## Phase 0 — Measure the gap (read-only, ~an hour)

- [ ] `bundle list | wc -l` — gem count and the five most replaceable gems
- [ ] `ls app/` — count non-Rails directories (services, forms, decorators, serializers, …)
- [ ] `grep -rc "def call" app/services 2>/dev/null` — service-object inventory
- [ ] `grep -rn "broadcasts_to\|broadcast_" app/models app/controllers | wc -l` per layer
- [ ] Longest model, longest controller, ApplicationController line count
- [ ] Test suite: factories-per-test? mock density? system-test count and runtime?
- [ ] Schema: validations without backing indexes (`uniqueness:` grep vs unique indexes)

Write the numbers down. They're your before/after.

## Phase 1 — Safety net (do first, enables everything else)

- [ ] Unique indexes under every uniqueness rule; NOT NULL where the model assumes presence
      (use `add_index ... unique: true, algorithm: :concurrently`-equivalents as your DB requires)
- [ ] One `test/test_helpers/` module with a real-endpoint `sign_in`; adopt in new tests
- [ ] Start a fixtures cast for the 4-6 core records new tests will share (factories and
      fixtures coexist fine during transition)

## Phase 2 — Dissolve the parallel architecture (highest payoff)

Work one service object at a time, newest/most-touched first:

- [ ] Single-aggregate writer service → named class-method constructor or instance method
      on the model, transaction included (`01-models.md` "Named class-method constructors")
- [ ] Collection-mutating service → association extension (`grant_to` shape)
- [ ] Multi-collaborator orchestration that resists both → keep it, but rename to a noun,
      namespace under its primary model, move to `app/models/` (`Room::MessagePusher` shape)
- [ ] Each move: call sites updated, old class deleted *in the same PR*, behavior covered
      by an integration test first if it wasn't
- [ ] Form objects → `params.expect/permit` + model defaults; query objects → scopes
      composed on the model; decorators → helpers (namespaced per controller)

Stop condition per object: if the rewrite makes the model worse (god-model risk), extract
a model-namespaced concern instead — facets, not layers.

## Phase 3 — Controllers back to CRUD

- [ ] List every custom route action (`member`/`collection` blocks); for each, name the
      hidden resource (mute → `resource :mute` or an Involvement-style setting) and move it
- [ ] Convert per-controller auth bolt-ons to concern-with-class-methods (or adopt the
      Rails 8 auth generator wholesale if auth is gem-based and you're ready — separate PR)
- [ ] Rename before_actions to `set_*`/`ensure_*`; push view-data computation into actions
- [ ] Re-scope lookups through `Current.user`'s associations; keep the policy layer if rich,
      but make scoping the floor underneath it

## Phase 4 — Hotwire alignment (if you're on Turbo already)

- [ ] Inventory `broadcasts_to`/callback broadcasts → move to explicit controller calls,
      extracting `Model::Broadcasts` vocabulary where 2+ call sites share shape
- [ ] Replace hand-built DOM ids with `dom_id(record, :purpose)` on both render and target sides
- [ ] Normalize stream names to `[parent, :collection]`
- [ ] Wrap multi-attribute Stimulus wiring into tag-builder helpers
- [ ] Add record fragment caching + `touch: true` chains; audit `update_all` for `updated_at`

## Phase 5 — Test suite re-anchoring (gradual, no flag-day)

- [ ] New tests: fixtures + integration seam by default
- [ ] When touching a mock-heavy test, rewrite against real objects
- [ ] Cut system tests that assert what integration tests already prove; add `using_session`
      coverage for any real-time feature that has none
- [ ] Delete shoulda-matcher/macro tests when their file is next open

## Phase 6 — Dependency diet (last; each removal is its own PR)

- [ ] Sidekiq/Resque → Solid Queue (jobs already thin from Phase 2 makes this mechanical)
- [ ] Redis cache/cable → Solid Cache/Cable (measure first if cache is hot)
- [ ] Devise → Rails auth generator — only with real session-migration planning
- [ ] Pagination/soft-delete/state-machine gems → scopes + `_at` columns where usage is shallow
- [ ] Re-run Phase 0 numbers; write the delta in the PR description

## Anti-goals (don't do these)

- Don't convert Postgres → SQLite on an existing multi-node app to "complete the look"
- Don't rewrite working Tailwind to vanilla CSS for style points — apply token layering within it
- Don't delete validations that drive form UX your product depends on; add the indexes and keep both
- Don't move everything at once: one pattern, one PR, call sites + deletion included
