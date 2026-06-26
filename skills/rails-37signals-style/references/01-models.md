# Dimension A — Models & Domain

## Pattern: Model-namespaced concerns for readable grouping

**One-line rule:** Extract cohesive facets of a model into concerns namespaced under that model (`User::Role` in `app/models/user/role.rb`), included on one line — even when nothing else will ever include them.

**Why 37signals does it:** Concerns aren't about reuse; they're a table of contents. `class User < ApplicationRecord; include Bot, Mentionable, Role, Transferable` tells you everything a user *is* in one line, and each facet reads as a complete 20-50 line story. The alternative — one 400-line model or a generic `app/models/concerns/` junk drawer — hides the domain.

**Citations:**
- `app/models/user.rb:2` — `include Bot, Mentionable, Role, Transferable` → `app/models/user/{bot,mentionable,role,transferable}.rb`
- `app/models/message.rb:2` — `include Attachment, Broadcasts, Mentionee, Pagination, Searchable` → `app/models/message/*.rb`
- `app/models/account.rb:2` + `app/models/account/joinable.rb`
- `app/models/membership.rb:2` + `app/models/membership/connectable.rb`

**Complete example** (`app/models/user/transferable.rb`, entire file, with host):

```ruby
class User < ApplicationRecord
  include Bot, Mentionable, Role, Transferable
  # ...
end

module User::Transferable
  extend ActiveSupport::Concern

  TRANSFER_LINK_EXPIRY_DURATION = 4.hours

  class_methods do
    def find_by_transfer_id(id)
      find_signed(id, purpose: :transfer)
    end
  end

  def transfer_id
    signed_id(purpose: :transfer, expires_in: TRANSFER_LINK_EXPIRY_DURATION)
  end
end
```

Note the anatomy: a constant scoping the concern's tunables, `class_methods do` for finders, instance methods below. Shared cross-model concerns are rare and earn a top-level name only when genuinely shared.

**Counter-example:** The average Rails app puts everything in `app/models/concerns/` with names like `Tokenable`, includes them in one model anyway, or skips extraction and grows a 600-line `user.rb` where roles, auth, and notifications interleave.

**When to break the rule:** Don't extract a concern for one method — `User#initials` lives happily in `user.rb`. Extract when a facet has its own constants, callbacks, or 3+ methods that read as a unit.

**Detection heuristic:** Models over ~80 lines with no `include` of own-namespace concerns; existence of `app/models/concerns/` files included by exactly one model (should be namespaced under it instead).

---

## Pattern: Association extensions are the domain API

**One-line rule:** Put multi-record domain operations in `has_many` block extensions so they read as `room.memberships.grant_to(users)` — not in service objects.

**Why 37signals does it:** The association already *is* the aggregate boundary. Extending it puts the verb exactly where the data lives, callers read like English, and `proxy_association.owner` gives access to the parent without passing it around.

**Citations:**
- `app/models/room.rb:2-18` — `grant_to`, `revoke_from`, `revise` on `has_many :memberships`
- `app/controllers/rooms/closeds_controller.rb:29` — `@room.memberships.revise(granted: grantees, revoked: revokees)`
- `app/models/rooms/open.rb:7` — `memberships.grant_to(User.active)`
- `app/models/first_run.rb:12` — `room.memberships.grant_to administrator`
- `test/models/room_test.rb:4-18` — the extension tested directly

**Complete example** (`app/models/room.rb:2-18`):

```ruby
class Room < ApplicationRecord
  has_many :memberships, dependent: :delete_all do
    def grant_to(users)
      room = proxy_association.owner
      Membership.insert_all(Array(users).collect { |user| { room_id: room.id, user_id: user.id, involvement: room.default_involvement } })
    end

    def revoke_from(users)
      destroy_by user: users
    end

    def revise(granted: [], revoked: [])
      transaction do
        grant_to(granted) if granted.present?
        revoke_from(revoked) if revoked.present?
      end
    end
  end
end
```

**Counter-example:** `MembershipGrantingService.call(room:, users:)` — a verb wearing a class costume, in a directory Rails never asked for, hiding the fact that this is a one-line bulk insert scoped to a room.

**When to break the rule:** When the operation spans multiple unrelated aggregates or needs heavy collaborators, promote it to a PORO **named as a noun** in `app/models` (see "POROs live in app/models"). Association extensions also aren't inherited by `has_many :through` chains.

**Detection heuristic:** Any `app/services/**/*_service.rb` whose `call` mutates a single parent's collection; controllers doing `Membership.create!` loops that belong on the association.

---

## Pattern: `Current.user` is ambient context, defaulted into associations

**One-line rule:** Set `Current.user` once at authentication, then absorb it with `belongs_to :creator, class_name: "User", default: -> { Current.user }` — never pass the current user down through method chains, and name the FK for its *role*.

**Why 37signals does it:** Who-did-this is request metadata, not a domain argument. The `default:` option means `@room.messages.create!(body: ...)` is automatically attributed. Role-named foreign keys (`creator_id`, `booster_id` — never bare `user_id` when a role exists) make every query self-documenting.

**Citations:**
- `app/models/current.rb:1-7` — entire class: `attribute :user`, plus `def account = Account.first` (one-account system)
- `app/models/message.rb:5`, `app/models/room.rb:23`, `app/models/boost.rb:3` — three `default: -> { Current.user }` associations
- `app/controllers/concerns/authentication.rb:70-74` — `Current.user = session.user` set in one place
- `config/routes.rb:29` — even routes use it: `Current.account&.updated_at` in a `direct` URL helper

**Complete example:**

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user

  def account
    Account.first
  end
end

class Message < ApplicationRecord
  belongs_to :room, touch: true
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end

# Controller — no creator mentioned anywhere:
@message = @room.messages.create_with_attachment!(message_params)
```

**Counter-example:** `MessageCreator.new(user: current_user).create(room, params)` or threading `current_user:` through four method signatures. The signature noise spreads to every caller and test.

**When to break the rule:** Background jobs and consoles have no `Current.user` — code paths reachable from jobs must pass the actor explicitly (Campfire's `Webhook#deliver` sets `creator: user` explicitly for exactly this reason, `app/models/webhook.rb:53`). Don't put derivable domain state in `Current`; it's for request-scoped identity (user, account, request details).

**Detection heuristic:** Methods/POROs taking `user:`/`current_user:` parameters that only re-attribute records; `belongs_to :user` columns that actually mean creator/author/booster.

---

## Pattern: Named class-method constructors wrap multi-step creation

**One-line rule:** When creation involves more than `create!`, give the model a named, intention-revealing class method that owns the transaction — `Room.create_for`, `User.create_bot!`, `Session.start!` — instead of a service or form object.

**Why 37signals does it:** The model is the factory for its own aggregate. Call sites read as domain sentences, the transaction boundary is owned by the code that knows what must be atomic, and there is exactly one way to create the thing correctly.

**Citations:**
- `app/models/room.rb:32-39` — `Room.create_for(attributes, users:)` wraps create + membership grant in a transaction
- `app/models/user/bot.rb:14-21` — `User.create_bot!` generates token, creates webhook
- `app/models/message/attachment.rb:14-16` — `Message.create_with_attachment!`
- `app/models/session.rb:10-12` — `Session.start!(user_agent:, ip_address:)`
- `app/models/search.rb:8-12` — `Search.record(query)` (idempotent: `find_or_create_by(...).touch`)
- `app/models/first_run.rb:5-15` — `FirstRun.create!` composes Account + Room + User

**Complete example** (`app/models/room.rb:32-44`):

```ruby
class << self
  def create_for(attributes, users:)
    transaction do
      create!(attributes).tap do |room|
        room.memberships.grant_to users
      end
    end
  end

  def original
    order(:created_at).first
  end
end
```

Used from `app/controllers/rooms/opens_controller.rb:15`: `room = Rooms::Open.create_for(room_params, users: Current.user)`.

**Counter-example:** `Rooms::CreateService.new(params).call` returning a `Result` object. Now creation logic lives outside the model, the transaction is in a third place, and `Room.create!` still exists as a loaded gun that skips the memberships.

**When to break the rule:** Plain `create!` needs no wrapper — don't manufacture `User.create_user!`. If construction needs heavy external collaborators (payment gateways), a noun-named PORO is honest.

**Detection heuristic:** `*Service`/`*Interactor`/`*Operation` classes whose body is `ActiveRecord::Base.transaction { ... }` around one aggregate; controllers calling `create!` then doing follow-up writes inline.

---

## Pattern: Trust the database; validate at trust boundaries

**One-line rule:** Enforce integrity with NOT NULL, unique indexes, and FKs — then handle `ActiveRecord::RecordNotUnique` where it matters; reserve `validates` for genuinely untrusted input crossing into your system.

**Why 37signals does it:** The database constraint is the only enforcement that can't be bypassed (bulk inserts, console, race conditions). Campfire's AR models contain **zero** validations — even `has_secure_password validations: false`. The only validations in the app live on `Opengraph::Metadata`/`Location`, ActiveModel POROs parsing arbitrary internet HTML — a real trust boundary.

**Citations:**
- `app/models/user.rb:20` — `has_secure_password validations: false`
- `db/structure.sql:69-71` — users: NOT NULL name, unique indexes on `email_address` and `bot_token`
- `db/structure.sql:57` — unique composite index on memberships `(room_id, user_id)`
- `app/controllers/users_controller.rb:11-17` — `User.create!` + `rescue ActiveRecord::RecordNotUnique` → redirect to sign-in
- `app/models/opengraph/metadata.rb:7-8` — `validates_presence_of :title, :url, :description` on untrusted input (the exception that proves the rule)

**Complete example** (`app/controllers/users_controller.rb:11-17`):

```ruby
def create
  @user = User.create!(user_params)
  start_new_session_for @user
  redirect_to root_url
rescue ActiveRecord::RecordNotUnique
  redirect_to new_session_url(email_address: user_params[:email_address])
end
```

The unique index *is* the uniqueness validation — race-proof, and the rescue turns it into UX.

**Counter-example:** `validates :email, presence: true, uniqueness: true` with no backing index — a TOCTOU race that corrupts data under concurrency, plus a validation error UI for a case better handled as "you already have an account, sign in."

**When to break the rule:** Public-facing forms where field-level error messages are the product (long signup forms) justify validations *in addition to* constraints. This codebase has terse forms and admin-ish UX; a consumer SaaS signup deserves friendlier failure modes. Never break the *constraint* half — indexes always.

**Detection heuristic:** `uniqueness: true` without a matching unique index in schema; models with 10+ validations mirroring NOT NULL columns; `rescue ActiveRecord::RecordInvalid` treated as control flow for trusted input.

---

## Pattern: Choose `dependent:` deliberately; write in bulk when callbacks don't matter

**One-line rule:** Use `:delete_all`/`:delete` when dependents have no callbacks worth firing, `:destroy` when they do; reach for `insert_all`/`update_all`/`destroy_by` consciously per call site.

**Why 37signals does it:** Every `dependent: :destroy` is an N+1 of callbacks at deletion time. Auditing which dependents actually need lifecycle hooks is a one-time cost that buys predictable deletes. The same audit mindset applies to writes: membership grants are `insert_all` (no callbacks needed), unread-marking is one `update_all` with an explicit `updated_at`.

**Citations:**
- `app/models/user.rb:4-15` — mixed on one model: `memberships, push_subscriptions, searches` are `:delete_all`; `messages, boosts, sessions` are `:destroy` (they have attachments/index cleanup/etc.)
- `app/models/user/bot.rb:8` — `has_one :webhook, dependent: :delete`
- `app/models/room.rb:5` — `Membership.insert_all` in `grant_to`
- `app/models/room.rb:69` — `memberships.visible.disconnected...update_all(unread_at: message.created_at, updated_at: Time.current)`
- `app/models/user.rb:35-46` — `deactivate` runs four `delete_all`s inside one transaction
- `app/models/membership/connectable.rb:12-18` — `update_all` for connect/disconnect

**Complete example** (`app/models/user.rb:35-46`):

```ruby
def deactivate
  transaction do
    close_remote_connections

    memberships.without_direct_rooms.delete_all
    push_subscriptions.delete_all
    searches.delete_all
    sessions.delete_all

    update! active: false, email_address: deactived_email_address
  end
end
```

**Counter-example:** `dependent: :destroy` on everything "to be safe," then a user deletion that fires 10,000 callbacks; or `users.each { |u| Membership.create!(...) }` in a loop.

**When to break the rule:** If a dependent gains a callback later (e.g., a counter cache, search index), you must remember to upgrade `:delete_all` → `:destroy` — note it where you add the callback. When in doubt about future callbacks on hot models, `:destroy` is the safer default; Campfire chooses per-association because the app is small enough to audit.

**Detection heuristic:** Uniform `dependent: :destroy` across a whole app (no thought applied); creation loops over homogeneous rows; `update_all` calls *missing* `updated_at` (cache-busting bug — Campfire always sets it).

---

## Pattern: Callbacks are one-line delegations to named domain methods

**One-line rule:** Callbacks may set defaults or kick off domain reactions, but the body is a lambda or single method call with a name — never inline business logic.

**Why 37signals does it:** Callbacks are the most-feared Rails feature because people inline logic into them. Used as one-line triggers (`after_create_commit -> { room.receive(self) }`), they keep "what happens when a message arrives" as a readable, testable method (`Room#receive`) while preserving the guarantee that it always happens.

**Citations:**
- `app/models/message.rb:11-12` — `before_create -> { self.client_message_id ||= Random.uuid }` (with the famous comment `# Bots don't care`) and `after_create_commit -> { room.receive(self) }`
- `app/models/user.rb:22` — `after_create_commit :grant_membership_to_open_rooms`
- `app/models/membership.rb:7` — `after_destroy_commit { user.reset_remote_connections }`
- `app/models/rooms/open.rb:3-9` — `after_save_commit :grant_access_to_all_users` guarded by `type_previously_changed?(to: "Rooms::Open")`
- `app/models/account/joinable.rb:5` — `before_create { self.join_code = generate_join_code }`
- `app/models/search.rb:4` — `after_create :trim_recent_searches`

**Complete example** (`app/models/rooms/open.rb`, entire file):

```ruby
# Rooms open to all users on the account. When a new user is added to the account, they're automatically granted membership.
class Rooms::Open < Room
  after_save_commit :grant_access_to_all_users

  private
    def grant_access_to_all_users
      memberships.grant_to(User.active) if type_previously_changed?(to: "Rooms::Open")
    end
end
```

Note `_commit` variants whenever the reaction touches the outside world (jobs, cable, other processes) — every external-effect callback here is `after_*_commit`.

**Counter-example:** `after_save :sync_to_crm, :send_emails, :update_search, if: :saved_change_to_status?` with 30 lines of conditionals inline — the model becomes unloadable in tests and every save is a minefield.

**When to break the rule:** UI-delivery side effects (Turbo broadcasts) do **not** belong in callbacks here — they fire explicitly from controllers (see `03-hotwire-views.md`). Cross-aggregate workflows with failure handling deserve explicit orchestration, not callback chains.

**Detection heuristic:** Callback bodies longer than one statement; `after_save` (not `_commit`) enqueuing jobs; more than ~3 callbacks of the same kind on one model.

---

## Pattern: STI for closed sets of behavioral variants

**One-line rule:** When a model has a small, fixed set of variants that differ in *behavior*, use single-table inheritance with predicate methods and type scopes — and `becomes!` to convert.

**Why 37signals does it:** `Rooms::Open`, `Rooms::Closed`, `Rooms::Direct` differ in access-granting behavior, not data. STI gives each variant a home for its hook overrides (`Open` auto-grants, `Direct` is a singleton-per-user-set) while sharing one table, one base API, and one set of routes/controllers (also subclassed — see `02-controllers.md`).

**Citations:**
- `app/models/room.rb:25-28` — type scopes: `scope :opens, -> { where(type: "Rooms::Open") }` etc.
- `app/models/room.rb:51-61` — predicates `open?`/`closed?`/`direct?` via `is_a?`
- `app/models/rooms/direct.rb:4-8` — `find_or_create_for` singleton semantics; `default_involvement` override
- `app/models/rooms/open.rb:3-9` — behavior in the subclass
- `app/controllers/rooms/opens_controller.rb:28-30` — `@room.becomes!(Rooms::Open)` to convert type on edit

**Complete example** (`app/models/room.rb:51-65` + subclass override):

```ruby
def open?
  is_a?(Rooms::Open)
end

def closed?
  is_a?(Rooms::Closed)
end

def direct?
  is_a?(Rooms::Direct)
end

def default_involvement
  "mentions"
end
```

```ruby
class Rooms::Direct < Room
  def default_involvement
    "everything"
  end
end
```

The base class defines the default; variants override. Callers never switch on type — they call the method.

**Counter-example:** A `room_type` string column with `case room.room_type when "open" ...` scattered across controllers and views; or three separate tables with 90% duplicated columns and a polymorphic mess to join messages.

**When to break the rule:** Variants that differ mainly in *data shape* (different columns) want separate tables. STI types are part of your schema's vocabulary — don't use it for open-ended user-defined kinds. Note Campfire stores class names in `type`; renaming classes means data migrations.

**Detection heuristic:** `case`/`when` on a `*_type`/`kind` column in 3+ places (should be polymorphic methods); empty STI subclasses *plus* type-conditionals elsewhere (the behavior never moved in).

---

## Pattern: POROs live in app/models — named as domain nouns

**One-line rule:** Plain-Ruby domain objects (composers, pushers, value objects, config readers) live in `app/models`, namespaced under the model they serve, named as nouns — `Room::MessagePusher`, never `PushMessageService`.

**Why 37signals does it:** `app/models` means "the domain," not "ActiveRecord subclasses." When logic genuinely doesn't belong on a record (multi-step push fan-out, first-run composition, a static sound catalog), it gets a noun name and sits next to the records it collaborates with. There is no second architecture to learn — and no `app/services` directory to become the junk drawer.

**Citations:**
- `app/models/room/message_pusher.rb:1-65` — the "service object done right": noun, namespaced, single public method
- `app/models/first_run.rb:1-17` — composes Account + Room + User for setup
- `app/models/sound.rb:1-89` — static catalog PORO with `BUILTIN` + `INDEX = BUILTIN.index_by(&:name)`
- `app/models/current.rb`, `app/models/purchaser.rb`, `app/models/application_platform.rb` — more non-AR residents
- `app/models/opengraph/{document,fetch,location,metadata}.rb` — a whole ActiveModel subsystem

**Complete example** (`app/models/room/message_pusher.rb:1-22`, abridged to the shape — full file is 65 lines):

```ruby
class Room::MessagePusher
  attr_reader :room, :message

  def initialize(room:, message:)
    @room, @message = room, message
  end

  def push
    build_payload.tap do |payload|
      push_to_users_involved_in_everything(payload)
      push_to_users_involved_in_mentions(payload)
    end
  end

  private
    def build_payload
      if room.direct?
        build_direct_payload
      else
        build_shared_payload
      end
    end
  # ... payload builders and subscription queries
end
```

Invoked by a 3-line job (`app/jobs/room/push_message_job.rb`). One public method, noun name, lives with `Room`.

**Counter-example:** `app/services/push_notification_service.rb` with `self.call(room_id, message_id)` — verb-as-class, top-level namespace pollution, and an implicit second framework (`Result` objects, `call` conventions) bolted onto Rails.

**When to break the rule:** None, really — this is the load-bearing alternative to service objects. The discipline is the *naming*: if you can't name it as a noun, the behavior probably belongs on a model that already exists.

**Detection heuristic:** `app/services|app/interactors|app/operations` directories; classes named `Verb+Noun+Service`; `def self.call`. Rewrite as model method, association extension, or namespaced noun-PORO.

---

## Pattern: Timestamps as state — semantic `_at` columns instead of booleans and stores

**One-line rule:** Model boolean-ish state as nullable timestamp (or counter) columns — `unread_at`, `connected_at`, `last_active_at` — and derive predicates and scopes from them.

**Why 37signals does it:** A timestamp is a boolean that also tells you *when*. `unread_at: nil` means read; setting it to the message's `created_at` both flags and dates it. Presence tracking — typically "you need Redis for that" — is two columns (`connections` counter + `connected_at` TTL) with range scopes. The state is queryable, indexable, and broadcast-friendly.

**Citations:**
- `db/structure.sql:53` — memberships: `unread_at`, `connections` (default 0), `connected_at`
- `app/models/membership.rb:14-23` — `scope :unread, -> { where.not(unread_at: nil) }`; `def read = update!(unread_at: nil)`; `def unread? = unread_at.present?`
- `app/models/membership/connectable.rb:4-50` — TTL presence: `scope :connected, -> { where(connected_at: CONNECTION_TTL.ago..) }`
- `app/models/room.rb:69` — unread fan-out: one `update_all(unread_at: message.created_at, ...)`
- `app/models/session.rb:1-27` — `last_active_at` + `ACTIVITY_REFRESH_RATE` throttled refresh

**Complete example** (`app/models/membership/connectable.rb:4-27`):

```ruby
module Membership::Connectable
  extend ActiveSupport::Concern

  CONNECTION_TTL = 60.seconds

  included do
    scope :connected,    -> { where(connected_at: CONNECTION_TTL.ago..) }
    scope :disconnected, -> { where(connected_at: [ nil, ...CONNECTION_TTL.ago ]) }
  end

  class_methods do
    def disconnect_all
      connected.update_all connected_at: nil, connections: 0, updated_at: Time.current
    end

    def connect(membership, connections)
      where(id: membership.id).update_all(connections: connections, connected_at: Time.current, unread_at: nil)
    end
  end

  def connected?
    connected_at? && connected_at >= CONNECTION_TTL.ago
  end
end
```

Beginless/endless ranges in scopes; the TTL means stale connections expire passively with no reaper process.

**Counter-example:** `read: boolean` columns (no audit trail, no ordering), or presence in Redis sets with TTLs — a second datastore, a serialization boundary, and state your SQL can't join against.

**When to break the rule:** Extremely hot presence churn (thousands of writes/sec) may outgrow row updates. Enum-like multi-state still wants an enum (Campfire: `enum involvement: %w[ invisible nothing mentions everything ].index_by(&:itself)`, `app/models/membership.rb:9` — note string-valued via `index_by(&:itself)`, with `_prefix` for readable predicates).

**Detection heuristic:** Boolean columns named `is_*`/`*_flag` that always change alongside a time; Redis usage for state that one indexed column could hold; `where("connected = true")` with a separate cleanup cron.
