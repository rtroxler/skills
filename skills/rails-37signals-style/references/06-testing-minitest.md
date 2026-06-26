# Dimension F — Testing with Minitest & Fixtures

## Pattern: Fixtures are a small named world, not factories

**One-line rule:** Maintain one internally-consistent cast of fixtures — real-sounding named people, realistic relationships, ERB for computed values and relative times — loaded once (`fixtures :all`) and referenced by name in every test.

**Why 37signals does it:** Factories rebuild the world per test, slowly, and every test re-states setup it doesn't care about. Fixtures invert it: `users(:david)` is an administrator, `users(:jz)` a designer in the `designers` room, `bender` a bot — a shared universe every test reads from, instantly (fixtures insert once per suite, transactions roll back per test). The cast is deliberately legible: the people are 37signals folks; the rooms tell you their type by name.

**Citations:**
- `test/fixtures/users.yml` — `<% password_digest = BCrypt::Password.create("secret123456") %>` once at top; five users incl. a bot with `bot_token: <%= User.generate_bot_token %>` (fixtures call real model code)
- `test/fixtures/messages.yml` — every message has `created_at: <%= 1.hour.ago %>`-style relative times (pagination tests depend on realistic ordering)
- `test/fixtures/memberships.yml` — join-table fixtures named `david_designers`, `jason_watercooler` — relationship legibility
- `test/fixtures/rooms.yml` — `type: Rooms::Open` STI types set explicitly
- `test/test_helper.rb:19` — `fixtures :all`

**Complete example** (`test/fixtures/users.yml`, head):

```yaml
<% password_digest = BCrypt::Password.create("secret123456") %>

david:
  name: David
  email_address: david@37signals.com
  password_digest: <%= password_digest %>
  role: administrator

jz:
  name: JZ
  email_address: jz@37signals.com
  password_digest: <%= password_digest %>
  bio: Designer

bender:
  name: Bender Bot
  bot_token: <%= User.generate_bot_token %>
  role: bot
```

One shared password ("secret123456") across the cast — sign-in helpers hardcode it; computing bcrypt once keeps suite boot fast.

**Counter-example:** `create(:user, :admin, rooms_count: 3)` per test — FactoryBot traits sprawl, per-test inserts dominate runtime, and no two tests agree on what the world looks like.

**When to break the rule:** Combinatorial edge-case records (a user in 40 states) don't belong in the shared world — build those inline in the specific test with `Model.create!`. Keep the fixture cast small; when a fixture exists only for one test file, it's noise.

**Detection heuristic:** `factory_bot` in the Gemfile of an app team that controls its own conventions; fixtures with `user1/user2/test_user` names (a world nobody can narrate); fixture files with absolute timestamps that rot.

---

## Pattern: Test helpers form a domain language

**One-line rule:** Put suite-wide verbs in `test/test_helpers/*.rb` modules included into `ActiveSupport::TestCase` — `sign_in`, `join_room`, `send_message`, `assert_message_text` — and make auth helpers exercise the real endpoint, not a backdoor.

**Why 37signals does it:** Tests should read at the same altitude as the product. `sign_in :david` in an integration test does a real `post session_url` and asserts the cookie landed — so every test continuously re-verifies the auth stack instead of stubbing around it. Domain assertions (`assert_room_unread`) encode *how the UI expresses state* (a CSS class) exactly once.

**Citations:**
- `test/test_helpers/session_test_helper.rb:6-10` — `sign_in` posts the real session endpoint, asserts `cookies[:session_token]`
- `test/test_helpers/system_test_helper.rb:2-55` — `sign_in`, `join_room`, `send_message`, `within_message`, `assert_message_text`, `assert_room_read/unread`, `dismiss_pwa_install_prompt`
- `test/test_helpers/mention_test_helper.rb:2-6` — `mention_attachment_for(:david)` renders the *real partial* via `ApplicationController.render` to build ActionText attachment HTML
- `test/test_helper.rb:21` — `include SessionTestHelper, MentionTestHelper`
- `test/application_system_test_case.rb:8` — `include SystemTestHelper`

**Complete example** (`test/test_helpers/session_test_helper.rb`, entire file):

```ruby
module SessionTestHelper
  def parsed_cookies
    ActionDispatch::Cookies::CookieJar.build(request, cookies.to_hash)
  end

  def sign_in(user)
    user = users(user) unless user.is_a? User
    post session_url, params: { email_address: user.email_address, password: "secret123456" }
    assert cookies[:session_token].present?
  end
end
```

Note the symbol-or-record flexibility (`sign_in :david` or `sign_in some_user`) and the embedded assertion — a failed sign-in fails loudly at the helper, not three asserts later.

**Counter-example:** `user.stub(:authenticated?, true)` or session-injection backdoors (`session[:user_id] = user.id`) — tests pass while the actual sign-in flow is broken; every test file reinvents its own login incantation.

**When to break the rule:** A unit test of pure model logic needs no sign-in at all — helpers are for the integration/system layers. If real-endpoint sign-in dominates suite time, a signed-cookie shortcut *plus one real-flow test* is a measured tradeoff.

**Detection heuristic:** No `test/test_helpers/` (or `test/support/`) directory; repeated multi-line login/setup sequences across files; auth stubbing in integration tests.

---

## Pattern: The integration seam is the primary altitude — assert behavior, not implementation

**One-line rule:** Carry most coverage in controller (`ActionDispatch::IntegrationTest`) tests that hit real routes against the fixture world and assert *observable* outcomes — response semantics, page membership, enqueued jobs, broadcasts — with model tests reserved for real domain logic.

**Why 37signals does it:** The integration test exercises routing, auth, scoping, the action, the view, and the broadcast in one go — maximal confidence per line. Assertions stay behavioral: `assert_response :no_content`, "page contains messages 1-2 but not 3-5", `assert_enqueued_jobs 1, only: [Room::PushMessageJob]`, `assert_broadcasts "unread_rooms", 1`. The suite is ~60 files for the whole product; nothing tests that a method called another method.

**Citations:**
- `test/controllers/messages_controller_test.rb:10-39` — pagination semantics tested through GET params (`before:`/`after:`), asserting which messages appear
- `test/controllers/messages_controller_test.rb:51-55` — `assert_broadcasts "unread_rooms", 1` around a real POST
- `test/models/message_test.rb:6-10` — `assert_enqueued_jobs 1, only: [ Room::PushMessageJob ]` — the callback's *effect*, not its existence
- `test/models/room_test.rb:4-36` — model tests target the association-extension API (`grant_to`, `revise`) — real logic, no AR boilerplate tests
- `test/test_helper.rb:12-13` — `include ActiveJob::TestHelper, Turbo::Broadcastable::TestHelper` suite-wide (with a `FIXME: Why isn't this included by default?`)
- `test/test_helper.rb:23-33` — setup resets cable pubsub + web-push pool; `WebMock.disable_net_connect!` (no test talks to the internet)

**Complete example** (`test/controllers/messages_controller_test.rb:17-27`):

```ruby
test "index returns a page before the specified message" do
  get room_messages_url(@room, before: @messages.third)

  assert_response :success
  ensure_messages_present @messages.first, @messages.second
  ensure_messages_not_present @messages.third, @messages.fourth, @messages.fifth
end
```

(`ensure_messages_present` is a file-local private helper asserting DOM membership — custom assertions are made freely, named in domain terms.)

**Counter-example:** `expect(MessageCreator).to receive(:call).with(...)` — mock-the-collaborator tests that pass through refactors that break production and fail on refactors that don't. Or validation-by-validation model specs restating the schema.

**When to break the rule:** Algorithmic logic (parsing, formatting, pagination math) earns focused unit tests — see `message_test.rb`'s `all_emoji?` cases. Mocha appears for *boundary* expectations (`Turbo::StreamsChannel.expects(:broadcast_replace_to).once`) — stubbing your own internals is still off-limits.

**Detection heuristic:** Heavy `mock`/`stub` of app-internal classes; one-assertion-per-AR-macro model specs; controller tests asserting `assigns(:foo)` or instance variables instead of responses.

---

## Pattern: System tests are few, multi-session, and cover money paths only

**One-line rule:** Reserve Capybara system tests for the handful of journeys that define the product — and use `using_session` to drive *two users at once* whenever the feature is real-time.

**Why 37signals does it:** System tests are the slowest, flakiest layer; Campfire keeps exactly **three** (`sending_messages`, `boosting_messages`, `unread_rooms`) — but they're the right three: the product *is* messages arriving on other people's screens, so the tests literally run two browser sessions and assert cross-user delivery through real WebSockets. Stability comes from helpers that wait on explicit signals (`wait_for_cable_connection` asserts `turbo-cable-stream-source[connected]`) rather than sleeps.

**Citations:**
- `test/system/` — three files total
- `test/system/sending_messages_test.rb:10-27` — `using_session("Kevin") { sign_in ...; join_room ... }` interleaved with the default session; message asserted on the *other* user's screen
- `test/test_helpers/system_test_helper.rb:12-14` — `wait_for_cable_connection` (condition-based waiting, count: 3 streams)
- `test/test_helpers/system_test_helper.rb:36-41` — `assert_room_read/unread` use `wait: 5` and class-presence semantics
- `test/application_system_test_case.rb:3-9` — `WebMock.disable!` for system runs; headless Chrome at 1400×1400

**Complete example** (`test/system/sending_messages_test.rb:9-27`):

```ruby
test "sending messages between two users" do
  using_session("Kevin") do
    sign_in "kevin@37signals.com"
    join_room rooms(:designers)
  end

  join_room rooms(:designers)
  send_message "Is this thing on?"

  using_session("Kevin") do
    join_room rooms(:designers)
    assert_message_text "Is this thing on?"

    send_message "👍👍"
  end

  join_room rooms(:designers)
  assert_message_text "👍👍"
end
```

**Counter-example:** A 200-file system suite re-walking every CRUD form (the integration layer already covers those), taking 40 minutes and failing 5% of runs — so everyone retries CI instead of reading failures.

**When to break the rule:** Heavily client-rendered features (rich autocomplete, drag-drop) may warrant a few more system tests since lower layers can't see them. The cap isn't "three" — it's *journeys that would page you at 2am*.

**Detection heuristic:** System tests asserting things an integration test could (status codes, record creation); `sleep` calls in Capybara tests; zero `using_session` in a product with real-time features.

---

## Pattern: Test what can break; let the framework test the framework

**One-line rule:** Don't test associations exist, validations validate, or scopes call `where` — spend the suite on domain methods, response semantics, side effects, and boundary handling; name every test as a behavioral sentence.

**Why 37signals does it:** ~60 test files cover a shipped product because nothing is ceremonial. `room_test.rb` is 37 lines: grant, revoke, revise, create_for, type predicates, default involvement — every test states a behavior a human asked for. There's no coverage tooling in the Gemfile; the metric is "do the things that matter have tests," answered by reading. Supporting cast: Mocha for boundaries, WebMock locked down globally, Faker only for the dev `populate` script (not tests), and a dedicated `performance` environment + `test/performance/` for when speed *is* the behavior.

**Citations:**
- `test/models/room_test.rb:26-31` — `test "type"` covers the predicate trio in 5 lines; nothing tests `has_many :messages`
- `test/models/message_test.rb:12-17` — `all_emoji?` edge cases (the regex is the risk; the association isn't)
- `Gemfile:48-60` — test group is capybara/mocha/selenium/webmock — no simplecov, no rspec, no factory_bot
- `test/test_helper.rb:32` — `WebMock.disable_net_connect!` in global setup
- `config/environments/performance.rb` + `test/performance/` — performance treated as a testable concern, separately
- test naming throughout: `test "creating a message enqueues to push later"`, `test "create for users by giving them immediate membership"` — sentences, not method names

**Complete example** (`test/models/room_test.rb:20-31` — what model tests look like when boilerplate is banned):

```ruby
test "create for users by giving them immediate membership" do
  room = Rooms::Closed.create_for({ name: "Hello!", creator: users(:david) }, users: [ users(:kevin), users(:david) ])
  assert room.users.include?(users(:kevin))
  assert room.users.include?(users(:david))
end

test "type" do
  assert Rooms::Open.new.open?
  assert_not Rooms::Open.new.direct?
  assert Rooms::Direct.new.direct?
  assert Rooms::Closed.new.closed?
end
```

**Counter-example:** `it { should belong_to(:room) }` shoulda-matcher walls — restating the model file with worse syntax; 95% coverage mandates that breed assertion-free tests run for the counter.

**When to break the rule:** Public-library code (gems) warrants exhaustive surface testing — this philosophy is for *applications you operate*, where the integration layer catches wiring mistakes. Regulated domains may genuinely require coverage evidence; collect it without letting the number set the agenda.

**Detection heuristic:** shoulda-matchers/`have_many` macro tests; test names like `test_user_1`; tests with no assertion on behavior (only "doesn't raise"); coverage gates in CI with no review of *what's* covered.
