# Dimension I — Code Style & Conventions

## Pattern: Comments exist only to flag surprises

**One-line rule:** Write no comment a reader could derive from the code; write a comment wherever the code is about to surprise them — a non-obvious why, a documented tradeoff, or a FIXME with its reasoning attached.

**Why 37signals does it:** `app/models/` contains **18 comment lines total** across 38 files. Naming carries the explanation (`grant_membership_to_open_rooms` needs no comment). What *does* get commented is exactly the stuff you couldn't guess — and every comment earns its place by recording intent, not narrating mechanics. FIXMEs ship with their analysis so the future fixer starts ahead.

**Citations:**
- `app/models/message.rb:11` — `before_create -> { self.client_message_id ||= Random.uuid } # Bots don't care` (why the `||=`: web clients send one, bots won't)
- `config/environments/production.rb:68` — `# SQLite is good, actually` (a stance, documented at the config it justifies)
- `app/models/rooms/direct.rb:8-9` — `# FIXME: Find a more performant algorithm that won't be a problem on accounts with 10K+ direct rooms, which could be to store the membership id list as a hash...` (problem + threshold + sketched fix)
- `app/models/application_platform.rb:31-34` — Apple Messages spoofs Facebook/Twitter bot user agents; the comment explains the otherwise-bizarre double `match?`
- `app/controllers/rooms/closeds_controller.rb:54` — `# Optimization so we don't have to render the same shared room per user`
- `app/views/messages/_message.html.erb:1` — `<%# Be sure to check/update messages/_template.html.erb when changing this file %>` (cross-file contract)
- `config/initializers/session_store.rb:3` — the 20-year cookie's rationale, at the setting

**Complete example** (`app/models/application_platform.rb:30-35`):

```ruby
def apple_messages?
  # Apple Messages pretends to be Facebook and Twitter bots via spoofed user agent.
  # We want to avoid showing "Unsupported browser" message when a Campfire link
  # is shared via Messages.
  match?(/facebookexternalhit/i) && match?(/Twitterbot/i)
end
```

Delete the comment and the next engineer "fixes" this method. That's the test for whether a comment belongs.

**Counter-example:** `# Set the user's name` above `user.name = name`; doc-comment blocks on every method per style-guide mandate; commented-out code kept "just in case" (git has it).

**When to break the rule:** Public gems/APIs need real API docs. Regulated codebases may mandate headers. Inside an application team, the bar stays: surprise or delete.

**Detection heuristic:** Comment density >1 per 20 lines in app code; comments restating the next line; FIXME/TODO without reasoning or condition attached.

---

## Pattern: One visual rhythm — public story first, indented private basement, whitespace as grouping

**One-line rule:** Order every class as: includes → associations/macros → callbacks → scopes → class methods (`class << self` when several) → public instance methods → a single indented `private` section; format with rubocop-rails-omakase and use blank lines (single within, double between groups) as the only other organizer.

**Why 37signals does it:** Every file reads the same way: the top third is declarative (what this thing *is*), the middle is its public API, the basement is mechanism. The omakase RuboCop config is two lines — style debates are outsourced to the framework authors' defaults, including the signature indented-`private` and `%i[ ... ]`/`[ 1, 2 ]` inner-space style. Double blank lines inside long modules mark sub-chapters without resorting to comment banners.

**Citations:**
- `.rubocop.yml:1-2` — entire file: `inherit_gem: { rubocop-rails-omakase: rubocop.yml }`
- `app/models/user.rb:1-68` — the canonical ordering top to bottom; `private` indented with methods nested under it
- `app/models/user/bot.rb:30-56` — double blank lines separating `update_bot!` / key methods / webhook methods sub-groups
- `app/controllers/messages_controller.rb:48-81` — private section grouped by blank lines: setters+guards, finders, params, webhooks
- multiline-call continuation with `\`: `app/models/user/bot.rb:23-24` (`http.request \`-style), `app/helpers/rooms_helper.rb:9-14` (`link_to \`), `app/models/room/message_pusher.rb` (`Push::Subscription` chain)
- argument spacing throughout: `%i[ show update ]` (`config/routes.rb:8`), `[ name, bio ].compact_blank` (`app/models/user.rb:32`)
- parens dropped on DSL-ish/final calls: `update! active: false, ...` (`user.rb:44`), `redirect_to room_url(room)` everywhere

**Complete example** (the shape, as `app/models/session.rb` exhibits it whole):

```ruby
class Session < ApplicationRecord
  ACTIVITY_REFRESH_RATE = 1.hour

  has_secure_token

  belongs_to :user

  before_create { self.last_active_at ||= Time.now }

  def self.start!(user_agent:, ip_address:)
    create! user_agent: user_agent, ip_address: ip_address
  end

  def resume(user_agent:, ip_address:)
    if last_active_at.before?(ACTIVITY_REFRESH_RATE.ago)
      update! user_agent: user_agent, ip_address: ip_address, last_active_at: Time.now
    end
  end
end
```

Constant first (the file's one tunable), macros, callback, class method, instance method. Nothing to scroll past, nothing out of order.

**Counter-example:** Private methods interleaved with public; `private :method_name` scattered inline; a 40-rule `.rubocop.yml` re-litigating string literals; section comment banners (`# == Associations ==`) doing what position should.

**When to break the rule:** Inherited codebases with a different established rhythm — consistency beats this convention. The omakase config is itself designed to be *layered onto*, not fought; add project rules below the inherit line if you must.

**Detection heuristic:** `def private_helper` above public API; `.rubocop.yml` >20 lines of style overrides; mixed `%i[a b]`/`%i[ a b ]` in one codebase (pick the omakase spacing).

---

## Pattern: Names do the documenting — a consistent verb/noun system

**One-line rule:** Predicates end in `?` and read as questions (`open?`, `connected?`, `deactivated?`); bang methods mean "raises or mutates importantly" (`create_for` returns, `revise` wraps a transaction, `update_bot!` raises); finders are `find_by_*`/`find_or_create_for`; scopes are adjectives or participles (`ordered`, `active`, `unread`, `with_creator`, `without_bots`); async pairs add `_later`; "fresh/original/reachable" style adjectives name derived concepts.

**Why 37signals does it:** With comments banned for the derivable, names carry the spec. The system is consistent enough to guess an API you've never read: what's the scope for non-bot users? `without_bots`. How do you enqueue webhook delivery? `deliver_webhook_later`. What's a message the user is allowed to see? `reachable_messages`. Negative predicates get positive names (`deactivated?` wraps `!active?`) so call sites never double-negate.

**Citations:**
- scopes as adjectives: `app/models/user.rb:17,24-25` (`active`, `ordered`, `filtered_by`); `app/models/membership.rb:11-15` (`with_ordered_room`, `without_direct_rooms`, `visible`, `unread`); `app/models/message.rb:14-15` (`ordered`, `with_creator`)
- predicate wrappers: `app/models/user.rb:48-50` (`deactivated?` = `!active?`); `app/models/membership.rb:21-23` (`unread?`); `app/models/room.rb:51-61`
- `_later` pairs: `app/models/user/bot.rb:51-57`, `app/models/room.rb:72-74`
- derived-concept nouns: `app/models/user.rb:7` (`reachable_messages`), `config/routes.rb:28,57` (`fresh_account_logo`, `fresh_user_avatar`), `app/models/room.rb:41-43` (`Room.original`)
- enum readability: `app/models/membership.rb:9` — `enum involvement: %w[ invisible nothing mentions everything ].index_by(&:itself), _prefix: :involved_in` → `involved_in_mentions?` (string-valued enums via `index_by(&:itself)`: DB stays readable, predicates stay fluent)
- intention-revealing locals: `app/controllers/rooms/closeds_controller.rb:41-47` (`grantees`, `revokees`)

**Complete example** (`app/models/membership.rb:9-23` — the naming system in one screen):

```ruby
enum involvement: %w[ invisible nothing mentions everything ].index_by(&:itself), _prefix: :involved_in

scope :with_ordered_room, -> { includes(:room).joins(:room).order("LOWER(rooms.name)") }
scope :without_direct_rooms, -> { joins(:room).where.not(room: { type: "Rooms::Direct" }) }

scope :visible, -> { where.not(involvement: :invisible) }
scope :unread,  -> { where.not(unread_at: nil) }

def read
  update!(unread_at: nil)
end

def unread?
  unread_at.present?
end
```

`memberships.visible.unread` reads as a sentence; `membership.read` is the verb users perform.

**Counter-example:** `get_active_users`, `is_open`, `process_data`, `handle_message`, `UserHelperService` — names that describe motion without meaning; scopes named after their WHERE clause (`status_eq_active`).

**When to break the rule:** Domain terms beat system regularity — `grant_to`/`revoke_from` aren't on any naming chart; they're the words the domain uses, which always wins.

**Detection heuristic:** `get_`/`is_`/`handle_`/`process_` prefixes; boolean methods without `?`; scopes that read as SQL not English; sync/async pairs with unrelated names.

---

## Pattern: Use the whole framework, current — the full ActiveSupport vocabulary on an up-to-date Rails

**One-line rule:** Reach for the framework's expressive layer before writing manual code — `index_by`, `compact_blank`, `presence_in`, `including`/`excluding`/`without`, `find_each`, `.tap`/`.then`, beginless/endless ranges, `to_fs(:custom_format)` — and keep Rails current enough that new framework answers (`rate_limit`, `authenticate_by`, `allow_browser`, `normalizes`) replace your workarounds as they land.

**Why 37signals does it:** They run Rails `main` (`Gemfile:9` — `gem "rails", github: "rails/rails", branch: "main"`) and harvest features the week they exist: `rate_limit` and `allow_browser` appear in this app *before* shipping in a Rails release. You shouldn't run `main` — but the posture transfers: stay within one minor of current, and when upgrading, *delete* the code the framework obsoleted (this snapshot's SQLite patches and auth concern are both now framework features — the 2026 move is removal). Meanwhile the small vocabulary keeps everyday code dense and declarative; extensions you do need go in `lib/rails_ext/` loaded by one initializer, openly monkeypatching with `prepend`/reopen rather than wrapper indirection.

**Citations:**
- vocabulary in the wild: `app/models/sound.rb:88` (`INDEX = BUILTIN.index_by(&:name)`); `app/models/user.rb:32` (`compact_blank`); `app/controllers/accounts/users_controller.rb:24` (`presence_in(%w[ member administrator ])`); `app/controllers/rooms/directs_controller.rb:18` (`ids.including(Current.user.id)`); `app/models/search.rb:16` (`excluding`); `app/views/users/sidebars/show.html.erb` + `rooms_helper.rb:57` (`without`); `app/models/membership/connectable.rb:7-8` (beginless/endless ranges); `config/initializers/time_formats.rb` (`Time::DATE_FORMATS[:epoch]` with a why-comment); `config/environments/production.rb:34-36` (`.tap{}.then{}` logger chain)
- edge harvesting: `app/controllers/sessions_controller.rb:3` (`rate_limit`), `:11` (`authenticate_by`); `app/controllers/concerns/allow_browser.rb:7` (`allow_browser versions:`)
- extension discipline: `config/initializers/extensions.rb:1-3` (auto-require `lib/rails_ext/*`); `lib/rails_ext/string.rb` (`String#all_emoji?` — 3 lines, tested at `test/models/message_test.rb:12-17`); `config/initializers/web_push.rb:18-44` (`WebPush::Request.prepend` to add persistent connections)
- custom config namespacing: `Rails.configuration.x.web_push_pool`, `config.x.vapid.*` (`config/initializers/{web_push,vapid}.rb`) — the blessed home for app-level config, not constants in initializers

**Complete example** (`config/initializers/extensions.rb` + one extension — the entire framework-extension story):

```ruby
# config/initializers/extensions.rb
%w[ rails_ext ].each do |extensions_dir|
  Dir["#{Rails.root}/lib/#{extensions_dir}/*"].each { |path| require "#{extensions_dir}/#{File.basename(path)}" }
end

# lib/rails_ext/string.rb
class String
  def all_emoji?
    self.match? /\A(\p{Emoji_Presentation}|\p{Extended_Pictographic}|\uFE0F)+\z/u
  end
end
```

Core-class extension, done in the open: one canonical directory, eager-required, no `Module#refine` ceremony, no `StringUtils.all_emoji?(str)` indirection.

**Counter-example:** `users.map { |u| [u.id, u] }.to_h` next to an unused `index_by`; a `DateUtils` module wrapping what `to_fs` formats do; pinning Rails 6.1 in 2026 and maintaining hand-rolled equivalents of five shipped features; monkeypatches hidden inside random initializers with no home directory.

**When to break the rule:** Running framework `main` is for people who employ the committers — track *releases* closely instead. Monkeypatching upstream gems (the WebPush prepend) deserves a comment pointing upstream and a check on each gem bump. Refinements over reopening when you're writing a library, not an app.

**Detection heuristic:** Manual implementations greppable as ActiveSupport methods (`each_with_object({})` building an index; `reject(&:blank?)`; `select { |x| allowed.include?(x) }.first`); `Gemfile` more than ~2 minors behind; `lib/` utilities wrapping framework features that have since shipped.
