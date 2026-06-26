# Checklist — Pre-Merge Review, 37signals Style

Review the diff against these questions. Each maps to a pattern card (dimension file in
parentheses) — cite the card when flagging, and honor its "when to break the rule."
Calibrate: flag what's *in the diff*; don't demand the whole codebase be converted.

## Architecture smells (01, 02)

- [ ] New `*Service`/`*Interactor`/`def self.call`? → model method, association extension,
      or noun-PORO under `app/models/<owner>/` (01)
- [ ] New concern in generic `concerns/` used by one model? → namespace it under the model (01)
- [ ] Model gained a facet inline pushing it past ~80 lines? → suggest `Model::Facet` concern (01)
- [ ] `current_user`/`user:` threaded through new method signatures? → `Current.user` +
      `belongs_to ... default:` — unless the path is job/console-reachable, then explicit is right (01)
- [ ] Multi-step creation outside a transaction, or transaction logic in the controller? →
      named class-method constructor owns it (01)

## Controllers & routes (02)

- [ ] New `member do`/`collection do` action? → name the resource it's hiding (02)
- [ ] Action >10 lines or with branching on a type param? → subclass controller / extracted
      private methods (02)
- [ ] New before_action not `set_*`/`ensure_*`-shaped, or unscoped? (02)
- [ ] Any `Model.find(params[...])` on user-owned data? → scope through `Current.user`'s reach;
      this one is security, not style (02, 07)
- [ ] Error responses speaking JSON-envelope in an HTML app, or numeric statuses? → `head :symbol` (02)

## Data integrity (01, 05)

- [ ] New uniqueness/presence rule with validation but no constraint? → index/NOT NULL required;
      validation optional (01)
- [ ] New boolean state column? → would `_at` timestamp serve better? (01)
- [ ] `update_all`/`insert_all` near cached fragments without explicit `updated_at`? (03, 05)
- [ ] `dependent:` chosen deliberately? `:destroy` on callback-free children, or `:delete_all`
      on children that just gained callbacks? (01)
- [ ] New Redis/external-store usage for state two columns could hold? (05)

## Callbacks & jobs (01, 04)

- [ ] Callback body >1 statement, or business logic inline? → named method (01)
- [ ] External effects (jobs, broadcasts, HTTP) in non-`_commit` callbacks? (01)
- [ ] Job `perform` >a delegation? → push logic to domain method with sync/`_later` pair (04)
- [ ] New async work that could be synchronous? Queue only slow/fan-out/external I/O (04)

## Hotwire (03)

- [ ] `broadcasts_to`/broadcast in model callbacks? → explicit controller broadcast (03)
- [ ] Hand-interpolated DOM ids or stream names? → `dom_id(record, :purpose)` both sides;
      `[parent, :collection]` streams (03)
- [ ] Raw multi-attribute `data-*` wiring pasted into templates? → tag-builder helper (03)
- [ ] New Stimulus controller >150 lines, page-named, or `querySelector`-ing across
      boundaries? → split, extract JS models, use targets/outlets/dispatch (03)
- [ ] New record partial without `cache record`, or list without `cached: true`? Is the
      `touch:` chain in place? (03)

## Security (07)

- [ ] Server-side fetch of user-influenced URL? → PrivateNetworkGuard-equivalent + redirect
      re-validation + size cap + content-type allowlist. Blocking issue, never style (07)
- [ ] Hand-rolled tokens/expiry where `has_secure_token`/`signed_id(purpose:)` fits? (07)
- [ ] `skip_before_action` on auth/CSRF broader than one declared path? (07)
- [ ] New unauthenticated endpoint — is it declared via the auth concern's class methods,
      and rate-limited if it accepts credentials/codes? (07)

## Tests (06)

- [ ] New behavior with no test at the integration seam? (06)
- [ ] Mocks/stubs of app-internal classes? → real objects + fixtures; mock boundaries only (06)
- [ ] Macro/ceremony tests (associations exist, validations validate)? → delete (06)
- [ ] New real-time feature without a `using_session` system test — or conversely, new system
      tests duplicating integration coverage? (06)
- [ ] Fixture additions consistent with the cast (named, relative times), not `user47`? (06)

## Style (09)

- [ ] Comments narrating the obvious, or surprising code with *no* comment? FIXMEs without
      reasoning? (09)
- [ ] Class reads in order — macros, scopes, public API, indented private basement? (09)
- [ ] Names: `?` predicates, adjective scopes, `_later` pairs, no `get_`/`handle_`/`process_`? (09)
- [ ] Manual code where ActiveSupport has a word (`index_by`, `compact_blank`, `presence_in`,
      `including`/`excluding`)? (09)
- [ ] New gem in the diff — what framework feature was tried first? (09, Gemfile discipline)

## The two questions that catch most violations

1. **"Which model owns this?"** — if the answer is a service, a job, or "the controller," keep asking.
2. **"What happens when this renders/broadcasts/raises twice?"** — idempotency, dom_id symmetry,
   and constraint-backed uniqueness are where this style earns its keep under concurrency.
