---
name: rails-code-review
description: >
  Opinionated code review for Ruby on Rails changes — runs a standard correctness/security/
  performance review AND checks the diff against 37signals/DHH "vanilla Rails" patterns
  (from the rails-37signals-style skill), suggesting the preferred idiom with citations when
  it spots a violation. Use when reviewing Rails code: "review my Rails changes", "review this
  PR" in a Rails repo, "is this idiomatic Rails", "review against main", "DHH-style review",
  "37signals review". For non-Rails diffs, use the generic `code-review` skill instead.
---

# Rails Code Review (37signals-opinionated)

This skill marries two things: the general-purpose review engine (correctness, security,
performance, clarity) and the opinionated **rails-37signals-style** pattern library. It
reviews a diff on **two layers** and reports them as **two separate tables**:

1. **Correctness & risk** — universal bugs/security/perf/etc. These are *problems*.
2. **Rails idiom (37signals)** — where the code works but isn't how 37signals would write
   it. These are *opinions* — suggested with a citation and an honest "when to break" note.

Keeping them separate is the whole point: "this is a race condition" and "this would read
better as an association extension" deserve different weight, and the reviewer should never
dress a style preference up as a blocker.

## Dependency

This skill cites the pattern cards in `~/.claude/skills/rails-37signals-style/`. When you
report a Layer-2 finding, **load the specific reference file** named in the detection matrix
to quote the exact rule, the copy-pasteable preferred example, and — critically — the
*When to break the rule* caveat before flagging. If that skill isn't installed, fall back
to the inline detection matrix here (it stands alone for the common cases, minus citations).

## Process

### Step 1 — Confirm it's Rails, and determine scope

Confirm the diff is Rails (a `Gemfile` with `rails`, `app/models` etc.). If it's not a
Rails change, stop and tell the user to use the generic `code-review` skill — don't apply
opinionated Rails idioms to non-Rails code.

Determine what to review (ask only if genuinely ambiguous):

| User says | Scope |
| --- | --- |
| "review my changes" | Uncommitted (`git diff HEAD`) |
| "review against main" | `git diff main...HEAD` |
| "review this file" | The named file(s) |
| "review the last commit" | `git diff HEAD~1` |
| "review the PR/MR" | Current branch vs base branch |

### Step 2 — Gather the diff and surrounding context

```bash
git diff HEAD            # uncommitted
git diff main...HEAD     # vs branch
git diff HEAD~N          # last N commits
git diff -- path/to/file # specific file
```

Read the full changed files when you need context — Rails idiom review especially needs to
see the *whole* model/controller, not just the changed lines (e.g. to judge whether a model
has crossed the "extract a concern" threshold, or whether a `Model.find` is unscoped).

### Step 3 — Layer 1: correctness & risk review

Run these universal checks (skip those irrelevant to the change):

| Check | What to look for |
| --- | --- |
| **Bugs & Logic** | Off-by-one, nil access, wrong comparisons, race conditions |
| **Security** | Injection, hardcoded secrets, missing auth checks, unsafe deserialization, mass-assignment |
| **Performance** | N+1 queries, missing `includes`, missing indexes, large allocations in hot paths |
| **Error Handling** | Swallowed exceptions, missing error cases, unhelpful messages |
| **API Design** | Breaking changes, inconsistent naming, missing validation, wrong HTTP verbs |
| **Concurrency** | Data races, missing locks, unsafe shared state, non-idempotent retries |
| **Resource Management** | Leaks (connections, file handles, ActionCable subscriptions), missing cleanup |
| **Code Clarity** | Confusing names, overly complex logic, missing comments on non-obvious code |
| **Duplication** | Copy-pasted logic that should be extracted |
| **Testing** | Missing tests for new logic, broken assertions, inadequate coverage |

> Some of these overlap with Layer 2 and that's fine — but route the finding to the table
> that matches its *weight*. An unscoped `Model.find(params[:id])` on user-owned data is a
> **Layer-1 Security** finding (missing authorization), even though rails-37signals-style
> also frames scoped lookups as an idiom. A model that's merely getting long is **Layer 2**.

### Step 4 — Layer 2: Rails idiom (37signals) review

Load `references/detection-matrix.md` and scan the diff against it. The matrix is organized
by the same nine dimensions as rails-37signals-style; each row gives a **smell** (often with
a grep), a **severity**, the **preferred pattern** (one-line rule), and the **card to cite**.

For every candidate finding:

1. **Read the cited card** in `~/.claude/skills/rails-37signals-style/references/NN-*.md`.
2. **Check the card's _When to break the rule_ section against this code.** If the code has a
   legitimate reason to deviate (the card names it), **do not flag it** — or flag it as LOW
   with an explicit "this may be a justified exception because…". This is non-negotiable: the
   pattern library is opinionated, not absolutist, and a reviewer who flags every deviation
   is noise.
3. **Only flag what's in the diff.** Don't demand the surrounding codebase be converted. If
   the change merely *follows* an existing non-idiomatic pattern, note it once, gently.

Severity calibration for Layer 2 (see the matrix for per-row guidance):
- **HIGH/CRITICAL** only when the idiom is also a real risk: SSRF on user-supplied URLs,
  unscoped lookups on user-owned data, hand-rolled crypto/tokens, broad `skip_before_action`.
- **MEDIUM** — architecture smells that will cost maintainability: service objects for
  single-aggregate writes, fat callbacks, `broadcasts_to` in models, missing DB constraints
  behind validations, business logic in jobs.
- **LOW** — naming, comments, file organization, "ActiveSupport already has a word for this."

### Step 5 — Report (two tables)

Display results in this **exact format**:

---

## Rails Code Review

**Layer 1 — Correctness & risk: X issues**

| # | Severity | Source | Location | Problem | Why | Fix |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | **CRITICAL** | Security | `app/controllers/messages_controller.rb:12` | Brief problem | Why it matters | Suggested fix |

**Layer 2 — Rails idiom (37signals): Z suggestions**

| # | Severity | Pattern | Location | Current | Preferred idiom | Cite |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | **MEDIUM** | Service object → model | `app/services/grant_membership_service.rb:1` | `GrantMembershipService.call(...)` | Association extension `room.memberships.grant_to(users)` | 01-models §Association extensions |

---

#### Severity definitions

- **CRITICAL** — Bug, data loss, or security hole. Must fix before merge.
- **HIGH** — Likely to cause problems / a real risk wearing an idiom's clothing. Strongly recommended.
- **MEDIUM** — Maintainability or architecture smell. Should fix.
- **LOW** — Nitpick, naming, style suggestion. Nice to fix.

Rules:
- Sort each table by severity (CRITICAL → LOW); number each table independently from 1.
- In Layer 2, **Preferred idiom** must be the actual one-line rule from the card (paraphrase
  faithfully), and **Cite** must point to the reference file + section so the user can read
  the full example and when-to-break.
- If a Layer-2 finding survived a *When to break* check by a hair, say so in the **Current**
  cell ("borderline — justified if …").

### Step 6 — Offer to fix

After the tables, ask:

> Want me to fix any of these? (e.g. "fix L1 #1", "apply idiom L2 #2 and #3", "fix all Layer 1")

When applying a **Layer 2** idiom refactor:
- Re-read the card's complete example and when-to-break before editing.
- Refactors that move logic (service → model, callback → controller broadcast) must keep
  behavior identical and be covered by a test — run the suite or add/adjust a test, and say so.
- Never apply a style refactor that the user declined or that the card's caveats argue against
  for this code.

## Clean review

If a layer is clean, still show its header with **0 issues** and list the checks/dimensions
scanned, so the user knows it was actually reviewed:

> **Layer 1 — Correctness & risk: 0 issues** — scanned: Bugs, Security, Performance, …
>
> **Layer 2 — Rails idiom (37signals): 0 suggestions** — scanned: Models, Controllers,
> Hotwire, Jobs, Persistence, Testing, Security, Conventions. Reads like vanilla Rails. 👌

## Quick reference — the crown-jewel smells

The highest-signal Layer-2 checks, inline so a fast review needs no file load (full set in
`references/detection-matrix.md`):

| Smell (grep) | Severity | Preferred idiom | Cite |
| --- | --- | --- | --- |
| `app/services/`, `def self.call`, `*Service`/`*Interactor` | MEDIUM | Model method / association extension / noun-PORO in `app/models` | 01-models |
| `current_user`/`user:` threaded through new method signatures | MEDIUM | `Current.user` + `belongs_to … default: -> { Current.user }` | 01-models |
| `validates … uniqueness:` with no matching unique index | HIGH | DB unique index; rescue `RecordNotUnique` at the controller | 01-models, 05-persistence |
| `member do`/`collection do` custom route action | MEDIUM | Name the hidden resource; new controller, CRUD only | 02-controllers |
| `Model.find(params[:id])` on user-owned data | HIGH | Scope through `Current.user`'s associations (also security) | 02-controllers, 07-security |
| `gem "devise"` / hand-rolled session in `session[:user_id]` | MEDIUM | Rails 8 auth generator; Session record + signed cookie + `Current` | 02-controllers, 07-security |
| `broadcasts_to` / `broadcast_*` inside model callbacks | MEDIUM | Broadcast explicitly from the controller; vocabulary in `Model::Broadcasts` | 03-hotwire-views |
| Interpolated DOM ids / stream names (`"msg_#{id}"`) | MEDIUM | `dom_id(record, :purpose)` on both render + broadcast sides | 03-hotwire-views |
| Job `perform` with real logic (>~a delegation) | MEDIUM | Thin job delegating to a domain method with a sync/`_later` pair | 04-jobs |
| `Net::HTTP`/`Faraday` on a user-supplied URL, no IP guard | CRITICAL | SSRF guard + per-redirect re-check + size/content-type caps | 07-security |
| Hand-rolled tokens / `JWT.encode` for first-party sessions | HIGH | `has_secure_token` / `signed_id(purpose:)` / `cookies.signed` | 07-security |
| `factory :`/FactoryBot, RSpec, mocks of app-internal classes | MEDIUM | Fixtures + integration-seam tests; mock only boundaries | 06-testing |
| `get_`/`is_`/`handle_` names; manual code where AS has a word | LOW | Predicate `?`, adjective scopes; `index_by`/`compact_blank`/`presence_in` | 09-conventions |
