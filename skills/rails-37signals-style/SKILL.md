---
name: rails-37signals-style
description: >
  Write and review Rails code the way 37signals does, with patterns extracted from the
  Campfire codebase (basecamp/once-campfire). Use when the user asks for "37signals style",
  "DHH style", "Campfire style", "Rails reference patterns", "vanilla Rails", asks
  "is this idiomatic Rails", "how would 37signals/DHH do this", or requests a review of
  Rails code against reference-quality standards. Also use when starting a new Rails app
  that should follow this style, or refactoring an existing one toward it.
---

# Rails, 37signals Style

Patterns extracted from a full audit of Campfire — the 37signals chat app written by the
team that authors Rails. Every pattern is backed by 3+ citations into that codebase
(`basecamp/once-campfire`, audited at the Feb 2024 ONCE snapshot; MIT-licensed and
public since 2025). Citations are repo-relative, e.g. `app/models/room.rb:2-18`.

**Era note (important):** the audited snapshot predates Rails 8. Where its hand-built
choices were later absorbed into Rails (auth generator, Solid Queue/Cache/Cable, Thruster,
SQLite production tuning, Kamal), the reference files say so explicitly. Never recommend
the 2024 workaround when the 2026 framework default does the same thing.

## The philosophy in ten lines

1. **The framework is the architecture.** No service objects, no form objects, no query
   objects, no policy objects, no decorators. Models, controllers, helpers, jobs, views —
   used at full power.
2. **Fat models, organized by concerns** — namespaced per-model, extracted for
   *readability*, not reuse.
3. **Controllers only ever do CRUD.** Anything that isn't CRUD becomes a new resource,
   not a custom action.
4. **`Current.user` is ambient context**, defaulted into associations — never threaded
   through method signatures.
5. **Trust the database.** Constraints and unique indexes over validation ceremony;
   validations are for trust boundaries (untrusted external input).
6. **Hotwire symmetry:** the `dom_id` you render is the `dom_id` you broadcast to;
   broadcasts fire explicitly from controllers, not callbacks.
7. **Jobs are async transports** — three lines, delegating to a domain method that also
   works synchronously.
8. **Fixtures are the test world.** A small named cast, exercised through real endpoints,
   asserted at the integration seam.
9. **Dependencies are a liability.** ~30 gems total. SQLite + the framework replace most
   of them. (Today: the Solid trifecta makes this even easier.)
10. **Comments are for surprises only.** The code is the documentation; a comment means
    "you couldn't have guessed this."

## How to use this skill

Load reference files **on demand** based on the dimension being worked on — do not load
all of them at once:

| You are working on… | Load |
|---|---|
| Models, domain logic, concerns, callbacks, validations | `references/01-models.md` |
| Controllers, routes, auth/authz | `references/02-controllers.md` |
| Views, Turbo, Stimulus, partials, CSS | `references/03-hotwire-views.md` |
| Background jobs | `references/04-jobs.md` |
| Database, SQLite, search, caching state | `references/05-persistence-sqlite.md` |
| Tests | `references/06-testing-minitest.md` |
| Security-sensitive code (URLs, tokens, cookies) | `references/07-security.md` |
| Dockerfiles, deploys, bin/ scripts | `references/08-deployment.md` |
| Naming, formatting, file organization, code review tone | `references/09-conventions.md` |

For task-shaped work, use the checklists:

- `checklists/new-rails-app.md` — boot a new app in this style from scratch
- `checklists/refactor-to-style.md` — migrate an existing app incrementally
- `checklists/pre-merge-review.md` — review a diff against these patterns

## Review stance

When reviewing code against this style, cite the pattern name and its detection
heuristic, show the Campfire-shaped rewrite, and respect each card's "when to break the
rule" section — these are opinions with documented limits, not laws. The audited app is
small (10 tables, one account per install); several patterns note honestly where scale
changes the answer. Flag, don't dogmatize.
