# Dimension B — Controllers & Routes

## Pattern: Everything is a resource — new controllers, never custom actions

**One-line rule:** When a "verb" doesn't fit CRUD, invent a resource whose CRUD *is* that verb — `resource :involvement, only: %i[ show update ]` — instead of adding a custom route/action.

**Why 37signals does it:** Campfire has 27 controllers over 10 tables and a routes file with **zero** `member`/`collection` custom actions (one `delete :clear, on: :collection` is the lone exception). "Refresh a room" is `Rooms::RefreshesController#show`. "Reset a bot key" is `Accounts::Bots::KeysController#update`. Every controller stays 7-action-max, every URL is guessable, and the noun-ing forces domain thinking: notification settings became a thing called *involvement*. 37signals now states this rule outright — fizzy's `STYLE.md` (§ CRUD controllers) gives the same `post :close` → `resource :closure` rewrite as its one canonical example, and that app carries exactly **one** custom (non-resource) route action across its entire route table.

**Citations:**
- `config/routes.rb:61-73` — rooms nest `resource :refresh, :settings, :involvement` under `scope module: "rooms"`
- `config/routes.rb:6-10` — sessions nest `resources :transfers` (QR-code device transfer = a resource)
- `config/routes.rb:16-20` — bots nest `resource :key, only: :update` (key rotation = update of a Key)
- `app/controllers/users/sidebars_controller.rb` — the sidebar is a showable resource
- `app/controllers/messages/boosts_controller.rb` — reactions are a sub-resource

**Complete example** (`config/routes.rb:61-79`):

```ruby
resources :rooms do
  resources :messages

  post ":bot_key/messages", to: "messages/by_bots#create", as: :bot_messages

  scope module: "rooms" do
    resource :refresh, only: :show
    resource :settings, only: :show
    resource :involvement, only: %i[ show update ]
  end

  get "@:message_id", to: "rooms#show", as: :at_message
end

namespace :rooms do
  resources :opens
  resources :closeds
  resources :directs
end
```

Singular `resource` for one-per-parent things; `scope module:` to keep controllers in `app/controllers/rooms/` without double-nesting URLs.

**Counter-example:** `post :refresh, :mark_read, :mute, on: :member` growing a 15-action `RoomsController` where before_actions apply to overlapping subsets and nobody can say what the controller is *for*.

**When to break the rule:** Don't noun things that are genuinely the same resource's lifecycle (publishing a post can be `update` with a status). Two levels of nesting max — Campfire never exceeds resource → sub-resource.

**Detection heuristic:** `member do`/`collection do` blocks in routes; controllers responding to non-REST action names; controllers with >7 public methods.

---

## Pattern: ApplicationController is a list of tiny concerns

**One-line rule:** Keep ApplicationController to include-lines only; each cross-cutting behavior is a named concern of ~10-30 lines owning its own before_actions and helper_methods.

**Why 37signals does it:** Each concern is independently readable, testable, and skippable (`allow_unauthenticated_access` is just a wrapped `skip_before_action`). The composition line *is* the documentation of what every request gets.

**Citations:**
- `app/controllers/application_controller.rb:1-5` — the whole file is two include lines
- `app/controllers/concerns/{authentication,authorization,room_scoped,set_platform,tracked_room_visit,version_headers,allow_browser}.rb` — seven single-purpose concerns
- `app/controllers/concerns/room_scoped.rb:1-13` — opt-in concern included by 3+ room-nested controllers

**Complete example** (`app/controllers/application_controller.rb`, entire file, plus one concern):

```ruby
class ApplicationController < ActionController::Base
  include AllowBrowser, Authentication, Authorization, SetPlatform, TrackedRoomVisit, VersionHeaders
  include Turbo::Streams::Broadcasts, Turbo::Streams::StreamName
end

module SetPlatform
  extend ActiveSupport::Concern

  included do
    helper_method :platform
  end

  private
    def platform
      @platform ||= ApplicationPlatform.new(request.user_agent)
    end
  end
```

**Counter-example:** A 200-line ApplicationController with 12 private methods, 8 before_actions, and helper methods for three unrelated features — every controller inherits all of it and nobody dares touch it.

**When to break the rule:** A concern used by exactly one controller belongs in that controller (Campfire keeps `RoomScoped` a concern because 3+ controllers include it).

**Detection heuristic:** ApplicationController > 30 lines; `app/controllers/concerns/` empty while ApplicationController is fat.

---

## Pattern: before_action discipline — `set_*` looks up, `ensure_*` guards

**One-line rule:** before_actions do exactly two things: assign instance variables (`set_room`, `set_message`) or halt with a guard (`ensure_can_administer`, `verify_join_code`) — each a small private method, scoped with `only:`/`except:`.

**Why 37signals does it:** The filter list at the top of a controller reads as a contract: what gets loaded, what gets checked, for which actions. Guards respond with `head :forbidden`/redirect and never half-mutate state.

**Citations:**
- `app/controllers/messages_controller.rb:4-6` — `set_room` / `set_message` / `ensure_can_administer only: %i[ edit update destroy ]`
- `app/controllers/rooms_controller.rb:2-4` — same trio shape plus `remember_last_room_visited, only: :show`
- `app/controllers/accounts/bots_controller.rb:2-3`, `app/controllers/users_controller.rb:4-5`, `app/controllers/sessions_controller.rb:5` (`ensure_user_exists`)
- `app/controllers/first_runs_controller.rb:4` — `prevent_repeats` (guard naming bends when clearer)

**Complete example** (`app/controllers/messages_controller.rb:1-6,48-55`):

```ruby
class MessagesController < ApplicationController
  include ActiveStorage::SetCurrent, RoomScoped

  before_action :set_room, except: :create
  before_action :set_message, only: %i[ show edit update destroy ]
  before_action :ensure_can_administer, only: %i[ edit update destroy ]

  # ... actions ...

  private
    def set_message
      @message = @room.messages.find(params[:id])
    end

    def ensure_can_administer
      head :forbidden unless Current.user.can_administer?(@message)
    end
end
```

**Counter-example:** before_actions that compute view data, send emails, or build forms; unscoped filters that run for every action and get `skip_before_action`-ed back out in children.

**When to break the rule:** Rendering-oriented setup that only one action needs is just a method call inside that action (`find_paged_messages` in `messages_controller.rb:58-67` is called from `index`, not filtered).

**Detection heuristic:** before_action methods >5 lines; filter names that aren't `set_*`/`ensure_*`/`verify_*`-shaped verbs; filters without `only:`/`except:` in controllers with mixed access levels.

---

## Pattern: Authorization is reach — scope every lookup through Current.user

**One-line rule:** Authorize by *querying through the user's associations* (`Current.user.rooms.find_by(id:)`) so unauthorized records are simply unfindable; layer at most one explicit question (`can_administer?`) on top.

**Why 37signals does it:** If every lookup starts from what the user can reach, authorization can't be forgotten — there's no "oops, used `Room.find`" hole. The single role question lives on the User model (`User::Role#can_administer?`), not in a parallel policy-object hierarchy.

**Citations:**
- `app/controllers/concerns/room_scoped.rb:10-11` — `Current.user.memberships.find_by!(room_id: params[:room_id])`
- `app/controllers/rooms_controller.rb:23` — `Current.user.rooms.find_by(id: ...)`
- `app/controllers/messages/boosts_controller.rb:26` — `Current.user.reachable_messages.find(params[:message_id])` (note the named *reach* association, `app/models/user.rb:7`)
- `app/channels/room_channel.rb:12` — same scoping inside ActionCable: `current_user.rooms.find_by(id: params[:room_id])`
- `app/models/user/role.rb:8-10` — the entire authz "framework": `administrator? || self == record&.creator || record&.new_record?`

**Complete example** (`app/models/user/role.rb`, entire file):

```ruby
module User::Role
  extend ActiveSupport::Concern

  included do
    enum role: %i[ member administrator bot ]
  end

  def can_administer?(record = nil)
    administrator? || self == record&.creator || record&.new_record?
  end
end
```

**Counter-example:** `Room.find(params[:id])` followed by `authorize @room` — correctness now depends on remembering the second line. Or 40 Pundit policies for an app with one meaningful question.

**When to break the rule:** **This is the card with the honest caveat.** Campfire has one account per install and essentially two roles — its 6-line authz is *proportionate to that scope*, not a claim that Pundit is always wrong. When you have multiple roles × resources × tenancy rules, or need to render permission matrices, graduate to policy objects — but keep the scoped-lookup habit underneath them; it's your defense-in-depth regardless.

**Detection heuristic:** Bare `Model.find(params[:id])` in authenticated controllers for user-owned resources; `current_user` passed into finder methods instead of being the receiver of the association chain.

---

## Pattern: Don't roll your own auth — but insist on this *shape*

**One-line rule:** Use `rails generate authentication` (Rails 8+) — it *is* this pattern productized; what you must preserve is the shape: a `Session` DB record per login, a signed httponly cookie holding its token, `Current.user` set in one concern, and declarative class-method overrides.

**Why 37signals does it:** Campfire's ~95-line `Authentication` concern predates and prefigured the Rails 8 generator. DB-backed sessions are individually revocable and auditable (IP, user agent, last-active). The concern shape gives controllers a one-line vocabulary: `allow_unauthenticated_access only: %i[ new create ]`. **Do not hand-copy this code into a new app today — generate it**, then extend in this style.

**Citations:**
- `app/controllers/concerns/authentication.rb:13-26` — `allow_unauthenticated_access`, `allow_bot_access`, `require_unauthenticated_access` class methods wrapping skip_before_action
- `app/models/session.rb:1-27` — `has_secure_token`, `Session.start!`, throttled `resume` (`ACTIVITY_REFRESH_RATE = 1.hour`)
- `app/controllers/concerns/authentication.rb:70-74` — `cookies.signed.permanent[:session_token] = { value: session.token, httponly: true, same_site: :lax }`
- `app/controllers/sessions_controller.rb:2-3` — `allow_unauthenticated_access` + `rate_limit to: 10, within: 3.minutes` on create
- `app/controllers/sessions_controller.rb:11` — `User.active.authenticate_by(email_address:, password:)` (timing-safe, framework-provided)
- `app/channels/application_cable/connection.rb:3-18` — the same `SessionLookup` module reused for WebSocket auth

**Complete example** — the declarative surface controllers actually touch:

```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: %i[ new create ]
  rate_limit to: 10, within: 3.minutes, only: :create, with: -> { render_rejection :too_many_requests }

  def create
    if user = User.active.authenticate_by(email_address: params[:email_address], password: params[:password])
      start_new_session_for user
      redirect_to post_authenticating_url
    else
      render_rejection :unauthorized
    end
  end
end
```

**Counter-example:** Devise — 20 modules, engine views, config DSL, and `current_user` magic you can't read in one sitting — for apps that need exactly: sign in, sign out, maybe reset. Equally bad: hand-rolling in 2026 what the generator emits.

**When to break the rule:** Real SSO/SAML/OIDC/enterprise-MFA requirements justify a dedicated library — that's a different problem, not a reason to give up on readable sessions for everything else.

**Detection heuristic:** `gem "devise"` in apps using ~2 of its modules; session state in `session[:user_id]` with no Session model (unauditable, unrevocable); auth logic spread across ApplicationController instead of one concern.

---

## Pattern: Variants get subclass controllers, not conditionals

**One-line rule:** When a resource has variants (STI types, API-vs-web entry points), subclass the controller and override the private hooks — strong params, guards, defaults — instead of branching inside actions.

**Why 37signals does it:** `Rooms::OpensController`, `ClosedsController`, `DirectsController` all inherit `RoomsController`'s `set_room`/`ensure_can_administer`/`room_params` and override only what differs. The bot API isn't a parallel `Api::V1::MessagesController` — it's `Messages::ByBotsController < MessagesController` overriding `message_params` to accept a raw request body. Template methods over if-statements.

**Citations:**
- `app/controllers/rooms/opens_controller.rb:1` / `closeds_controller.rb:1` / `directs_controller.rb:1` — all `< RoomsController`
- `app/controllers/rooms/directs_controller.rb:31-34` — overrides `ensure_can_administer` to allow all members
- `app/controllers/messages/by_bots_controller.rb:1-23` — `< MessagesController`, overrides `message_params`, calls `super` then `head :created, location:`
- `app/controllers/rooms/opens_controller.rb:27-30` — `force_room_type` + `becomes!` for type conversion on edit

**Complete example** (`app/controllers/messages/by_bots_controller.rb`, entire file):

```ruby
class Messages::ByBotsController < MessagesController
  allow_bot_access only: :create

  def create
    super
    head :created, location: message_url(@message)
  end

  private
    def message_params
      if params[:attachment]
        params.permit(:attachment)
      else
        reading(request.body) { |body| { body: body } }
      end
    end

    def reading(io)
      io.rewind
      yield io.read.force_encoding("UTF-8")
    ensure
      io.rewind
    end
  end
```

Note what this buys: bots POST plain text or a file — no JSON envelope ceremony — and the entire "API" is one subclass. There is no `app/controllers/api/` namespace in the codebase.

**Counter-example:** `if params[:room_type] == "direct"` branches inside `RoomsController#create`; or a versioned API namespace duplicating every controller for one integration.

**When to break the rule:** A public, versioned, documented API for third parties deserves its own namespace and serializers. Subclass-controllers shine for *internal* variants of the same resource.

**Detection heuristic:** `case`/`if` on a type param inside controller actions; near-duplicate controllers differing in params/auth only; `respond_to` blocks where a subclass would isolate the format.

---

## Pattern: Speak HTTP — head, fresh_when, and status-bearing redirects

**One-line rule:** Respond with the protocol: `head :forbidden/:no_content/:created`, `fresh_when` for conditional GETs, `redirect_to` with flash for navigation — skip `respond_to` ceremony unless formats genuinely diverge.

**Why 37signals does it:** The HTTP vocabulary already covers most responses; using it keeps actions to 1-3 lines and makes clients (including Turbo and bots) behave correctly for free. Conditional GET via `fresh_when` makes message-history pagination cacheable with zero extra infrastructure.

**Citations:**
- `app/controllers/messages_controller.rb:10-18` — `fresh_when @messages` / `head :no_content`
- `app/controllers/messages/by_bots_controller.rb:6` — `head :created, location: message_url(@message)`
- guard responses: `head :forbidden` (×4 across `messages_controller.rb:54`, `authentication.rb:85`, `authorization.rb:4`, `rooms_controller.rb:31`); `head :not_found` (`users_controller.rb:28`)
- `app/controllers/sessions_controller.rb:30-33` — `render :new, status: :too_many_requests` / `:unauthorized` with `flash.now`
- `app/controllers/accounts_controller.rb:11` — `redirect_to edit_account_url, notice: "✓"` (yes, the notice is a checkmark)

**Complete example** (`app/controllers/messages_controller.rb:10-18`):

```ruby
def index
  @messages = find_paged_messages

  if @messages.any?
    fresh_when @messages
  else
    head :no_content
  end
end
```

**Counter-example:** `render json: { error: "forbidden" }, status: 403` from an HTML controller; `respond_to do |format| format.html; format.turbo_stream; end` blocks where Rails would have negotiated identically without them.

**When to break the rule:** When an action genuinely renders different *content* per format, `respond_to` is the right tool — Campfire just rarely needs it because Turbo Stream responses get their own template files (`create.turbo_stream.erb`).

**Detection heuristic:** JSON error envelopes in server-rendered apps; numeric status codes instead of symbols; missing `fresh_when`/etag on heavy index actions polled by clients.
