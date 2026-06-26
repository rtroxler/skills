# Dimension C — Views, Turbo, Stimulus, CSS

## Pattern: Broadcast explicitly from the controller, not from model callbacks

**One-line rule:** Fire Turbo Stream broadcasts as explicit statements at the point of action (`@message.broadcast_create` in the controller), keeping broadcast *vocabulary* in a model concern — never implicit `broadcasts_to` callbacks.

**Why 37signals does it:** The grep says it all: 31 `broadcast_*` calls in `app/controllers/`, 4 in `app/models/`. UI delivery is a controller decision — the same model create may broadcast differently per entry point (web vs bot vs webhook reply), and update/destroy broadcasts carry view-specific targets and attributes that models have no business knowing. Models keep durable side effects (unread tracking, push jobs) in callbacks; *DOM delivery* stays at the edge.

**Citations:**
- `app/controllers/messages_controller.rb:24` — `@message.broadcast_create` after create
- `app/controllers/messages_controller.rb:39` — update: `@message.broadcast_replace_to @room, :messages, target: [ @message, :presentation ], partial: "messages/presentation", attributes: { maintain_scroll: true }`
- `app/controllers/messages_controller.rb:45` — destroy: `@message.broadcast_remove_to @room, :messages`
- `app/models/message/broadcasts.rb:1-6` — the named vocabulary: `broadcast_create` = append + ActionCable ping
- `app/controllers/rooms/closeds_controller.rb:53-60` — render-once optimization: `render_to_string` the partial, then `broadcast_prepend_to user, :rooms, html: new_room_html` per user
- `app/controllers/rooms/involvements_controller.rb:16-25` — conditional broadcasts based on state transition
- contrast `app/models/message.rb:12` — the callback handles `room.receive(self)` (unread + push), *not* DOM updates

**Complete example** (`app/models/message/broadcasts.rb` + call site):

```ruby
module Message::Broadcasts
  def broadcast_create
    broadcast_append_to room, :messages, target: [ room, :messages ]
    ActionCable.server.broadcast("unread_rooms", { roomId: room.id })
  end
end

# app/controllers/messages_controller.rb
def create
  set_room
  @message = @room.messages.create_with_attachment!(message_params)

  @message.broadcast_create
  deliver_webhooks_to_bots
rescue ActiveRecord::RecordNotFound
  render action: :room_not_found
end
```

**Counter-example:** `after_create_commit -> { broadcast_append_to room }` on the model — now every console fixture, data migration, and webhook reply broadcasts to the UI; tests must stub broadcasting; and there's no way to vary the broadcast per entry point.

**When to break the rule:** Truly uniform notification-feed appends with one render shape can use `broadcasts_to` defensibly. The moment you write `target:`/`partial:` options or a second broadcast call, move it to the controller.

**Detection heuristic:** `broadcasts_to`/`broadcast_*` inside `after_*` callbacks in models; broadcast options duplicated in 3+ call sites without a named model method (the inverse failure — extract a `Message::Broadcasts`-style concern).

---

## Pattern: The dom_id contract — render and broadcast must agree on composite ids

**One-line rule:** Give every replaceable fragment an id built by `dom_id(record, purpose)` and target broadcasts with the array form `target: [ record, :purpose ]` — the two construct identical strings, and that symmetry is the entire correctness model of Turbo Streams.

**Why 37signals does it:** Hotwire's reliability isn't magic — it's a naming discipline. `dom_id(message, :presentation)` in the partial and `target: [ @message, :presentation ]` in the broadcast both produce `presentation_message_123`. Adopt the discipline and stream updates "just work"; improvise ids and you debug ghost updates forever.

**Citations:**
- `app/views/messages/_presentation.html.erb:1` — `id="<%= dom_id(message, :presentation) %>"` ↔ `app/controllers/messages_controller.rb:39` — `target: [ @message, :presentation ]`
- `app/views/messages/_message.html.erb:11` — `<turbo-frame id="<%= dom_id(message, :edit) %>">`
- `app/helpers/messages_helper.rb:16` — `id: dom_id(room, :messages)` ↔ `app/views/messages/create.turbo_stream.erb:1` — `turbo_stream.append dom_id(@message.room, :messages), @message`
- `app/views/users/sidebars/rooms/_shared.html.erb:2` — `id: dom_id(room, :list)` ↔ `app/controllers/rooms_controller.rb:49` — `broadcast_remove_to :rooms, target: [ @room, :list ]`
- 29 `dom_id` occurrences across views/helpers

**Complete example** — one fragment, render side and broadcast side:

```erb
<%# app/views/messages/_presentation.html.erb %>
<div id="<%= dom_id(message, :presentation) %>" data-reply-target="body">
  <%= message_presentation(message) %>
</div>
```

```ruby
# app/controllers/messages_controller.rb — update action
@message.broadcast_replace_to @room, :messages,
  target: [ @message, :presentation ], partial: "messages/presentation",
  attributes: { maintain_scroll: true }
```

**Counter-example:** Hand-built ids — `id="message-#{message.id}-body"` in one place, `target: "msg_#{id}_body"` in another. Every rename is a silent production bug.

**When to break the rule:** Ids keyed by something other than a record (Campfire targets boosts with `"boosts_message_#{client_message_id}"`, `messages/boosts_controller.rb:35`, because optimistic client-side messages don't have server ids yet). Fine — but then define a helper so both sides share the constructor.

**Detection heuristic:** String-interpolated DOM ids in views or `target:` options; `dom_id` on one side of a render/broadcast pair but not the other.

---

## Pattern: Name streams [scope, :collection] and subscribe at the layout seam

**One-line rule:** Stream names are `turbo_stream_from parent, :collection` (e.g. `@room, :messages`) or a global symbol (`:rooms`), declared in the view that owns the region — with per-user streams as `user, :collection`.

**Why 37signals does it:** Three streams cover the whole app: `[room, :messages]` for in-room updates, `:rooms` for everyone's sidebar, `[user, :rooms]` for personal sidebar entries. The convention makes the broadcast call site (`broadcast_append_to room, :messages`) and the subscription readable as the same sentence.

**Citations:**
- `app/views/rooms/show.html.erb:20` — `turbo_stream_from @room, :messages`
- `app/views/users/sidebars/show.html.erb:2-3` — `turbo_stream_from :rooms` and `turbo_stream_from Current.user, :rooms` side by side
- broadcasters matching: `app/models/message/broadcasts.rb:3` (`room, :messages`), `app/controllers/rooms/opens_controller.rb:46` (`:rooms`), `app/controllers/rooms/directs_controller.rb:27` (`membership.user, :rooms`)

**Complete example** (`app/views/users/sidebars/show.html.erb:1-3` + a matching broadcast):

```erb
<%= sidebar_turbo_frame_tag do %>
  <%= turbo_stream_from :rooms %>
  <%= turbo_stream_from Current.user, :rooms %>
```

```ruby
# app/controllers/rooms/directs_controller.rb — personal stream, per member
room.memberships.each do |membership|
  membership.broadcast_prepend_to membership.user, :rooms, target: :direct_rooms, partial: "users/sidebars/rooms/direct"
end
```

**Counter-example:** Stream names invented ad hoc (`"room_#{id}_updates"`, `"sidebar-#{user.id}"`) — unsigned strings drifting between subscriber and broadcaster, with no grep-able convention.

**When to break the rule:** Raw `ActionCable.server.broadcast("unread_rooms", ...)` with a hand-named channel is used for *data* (JSON) pushes consumed by Stimulus, vs Turbo Streams for *HTML* — keep that distinction (see `app/models/message/broadcasts.rb:4` and `app/javascript/controllers/` consumers).

**Detection heuristic:** Interpolated stream-name strings; `turbo_stream_from` deep inside partials rendered multiple times per page (duplicate subscriptions).

---

## Pattern: Helpers own the data-attribute wiring; views read like prose

**One-line rule:** Wrap any element carrying Stimulus/Turbo wiring in a tag-builder helper (`message_area_tag`, `link_to_room`, `composer_form_tag`) so the `data-controller`/`data-action` contract lives in exactly one Ruby method — and pass page chrome through `content_for` slots.

**Why 37signals does it:** Stimulus wiring is *configuration*, and configuration scattered across ERB rots. Campfire's views are visually clean because helpers like `message_tag` centralize 8 data-attributes (controllers, targets, outlets, sort keys) into one place that both the partial and any broadcast-rendered context reuse. Helpers — 24 files, namespaced like controllers (`Users::SidebarHelper`) — are the presentation layer; there are no decorators or component classes.

**Citations:**
- `app/helpers/messages_helper.rb:2-42` — `message_area_tag`, `messages_tag`, `message_tag` each bundling controller/action/target wiring
- `app/helpers/rooms_helper.rb:2-6, 50-53` — `link_to_room` (merging caller data), `composer_form_tag`
- `app/helpers/messages_helper.rb:65-79` — action strings extracted to named private methods (`messages_actions`, `refresh_room_actions`)
- layout slots: `app/views/layouts/application.html.erb:34-61` — `yield :nav`, `yield :sidebar`, `yield :footer`; filled via `content_for` in `app/views/searches/index.html.erb:4,31,61` and ivars `@page_title`/`@body_class` (`rooms/show.html.erb:1-2`)

**Complete example** (`app/helpers/messages_helper.rb:15-23` and its call site):

```ruby
def messages_tag(room, &)
  tag.div id: dom_id(room, :messages), class: "messages", data: {
    controller: "maintain-scroll refresh-room",
    action: [ maintain_scroll_actions, refresh_room_actions ].join(" "),
    messages_target: "messages",
    refresh_room_loaded_at_value: room.updated_at.to_fs(:epoch),
    refresh_room_url_value: room_refresh_url(room)
  }, &
end
```

```erb
<%# app/views/rooms/show.html.erb — the entire room body %>
<%= message_area_tag(@room) do %>
  <%= render "messages/template" %>

  <%= messages_tag(@room) do %>
    <%= render "rooms/show/invitation", room: @room %>
    <%= render partial: "messages/message", collection: @messages, cached: true %>
  <% end %>

  <%= turbo_stream_from @room, :messages %>
  <%= button_to_jump_to_newest_message %>
<% end %>
```

**Counter-example:** 15 raw `data-*` attributes inline in ERB, duplicated wherever the element renders; or ViewComponent classes adding a render-object layer to get the same encapsulation helpers already provide.

**When to break the rule:** One-off wiring used in a single view can stay inline — extract on the second use or when attributes exceed ~3. Helpers that accumulate unrelated methods need splitting by namespace, not abandonment.

**Detection heuristic:** The same `data-controller` value with multi-attribute wiring appearing in 2+ templates; ERB lines >120 chars dominated by `data:` hashes.

---

## Pattern: A fleet of tiny, generic Stimulus controllers + plain-JS models

**One-line rule:** Default to single-purpose, reusable Stimulus controllers configured entirely through `static values/targets/classes/outlets`; when one grows real logic, extract plain JS classes into `javascript/models/` and let the controller orchestrate.

**Why 37signals does it:** Of Campfire's 38 controllers, most are app-agnostic (`copy_to_clipboard`, `element_removal`, `toggle_class`, `auto_submit`, `sorted_list`, `local_time`) — HTML composes behaviors by listing controllers. The one big feature (the message area) doesn't become a 500-line controller; it's `messages_controller.js` orchestrating `ClientMessage`, `MessageFormatter`, `MessagePaginator`, `ScrollManager` — testable classes with no Stimulus dependency. Cross-controller talk uses `this.dispatch` events and outlets, never reaching into other controllers.

**Citations:**
- `app/javascript/controllers/element_removal_controller.js:1-7` — entire controller is 7 lines
- `app/javascript/controllers/copy_to_clipboard_controller.js` — `static values = { content: String }`, `static classes = [ "success" ]`, `#private` methods
- `app/javascript/controllers/messages_controller.js:1-45` — imports 4 models from `models/`, wires them in `connect()`
- `app/javascript/models/{client_message,message_formatter,message_paginator,scroll_manager,typing_tracker,file_uploader}.js` — the JS domain layer
- `app/javascript/controllers/presence_controller.js:21` — `this.dispatch("present", { detail: { roomId } })`; consumed via `data-action` strings in helpers
- `app/javascript/controllers/sorted_list_controller.js:7-9` — `itemTargetConnected` lifecycle callback (reacting to streamed DOM arrivals)
- directory layout: `app/javascript/{controllers,models,helpers,lib,initializers}` pinned via `pin_all_from` (`config/importmap.rb:11-16`)

**Complete example** (`app/javascript/controllers/sorted_list_controller.js`, entire file — keeps streamed lists sorted with zero server coordination):

```js
import { Controller } from "@hotwired/stimulus"
import { throttle } from "helpers/timing_helpers"

export default class extends Controller {
  static targets = [ "item" ]

  itemTargetConnected(target) {
    this.#throttledSort()
  }

  updateItem({ detail: { targetId }}) {
    const itemTargetForUpdate = this.itemTargets.find(itemTarget => itemTarget.id == targetId)

    if (itemTargetForUpdate) {
      if (itemTargetForUpdate.dataset.sortedListNumber) {
        itemTargetForUpdate.dataset.sortedListNumber = new Date().getTime()
      }

      this.sort()
    }
  }

  sort() {
    const sortedItemTargets = this.itemTargets.sort((a, b) => {
      if (a.dataset.sortedListNumber) {
        return b.dataset.sortedListNumber - a.dataset.sortedListNumber
      } else {
        return a.dataset.sortedListName.toLowerCase().localeCompare(b.dataset.sortedListName.toLowerCase())
      }
    })

    sortedItemTargets.forEach(item => this.element.appendChild(item))
  }

  #throttledSort = throttle(this.sort.bind(this))
}
```

Servers broadcast prepends in any order; this controller makes order a client concern. Note `#private` fields, target-connected callbacks, and configuration via `data-sorted-list-name`/`-number` on items.

**Counter-example:** One `app_controller.js` god-controller; React-style state management bolted on; controllers querying `document.querySelector` into DOM they don't own instead of targets/outlets/events.

**When to break the rule:** Truly rich client features (Campfire's autocomplete) graduate to `javascript/lib/` with custom elements — still framework-free. If you need client-side *state synchronization* beyond DOM, you've left Hotwire's sweet spot; decide deliberately.

**Detection heuristic:** Stimulus controllers >150 lines without extracted models; controller names describing pages (`dashboard_controller`) rather than behaviors; `querySelector` calls crossing controller boundaries.

---

## Pattern: Cache at the record fragment; render collections by convention

**One-line rule:** Wrap record partials in `<% cache record do %>`, render lists with `render partial: ..., collection:, cached: true`, and rely on `touch: true` chains to invalidate.

**Why 37signals does it:** Russian-doll caching is the performance story for server-rendered HTML — the message list renders thousands of cached fragments in one multi-get. It only works because of two disciplines elsewhere: `belongs_to :room, touch: true` (`message.rb:4`) and `belongs_to :message, touch: true` (`boost.rb:2`) bust parent caches; and every `update_all` sets `updated_at` explicitly (`room.rb:69`, `membership/connectable.rb:13`) because bulk writes skip touching.

**Citations:**
- `app/views/messages/_message.html.erb:3` — `<% cache message do %>`
- `app/views/rooms/show.html.erb:17` — `render partial: "messages/message", collection: @messages, cached: true`
- `app/views/users/sidebars/rooms/_direct.html.erb:1` — `<% cache membership do %>` + collection-cached at `sidebars/show.html.erb:22`
- `app/models/message.rb:4`, `app/models/boost.rb:2` — the `touch: true` chain
- `config/routes.rb:28-30, 57-59` — cache-busting URL helpers: `direct :fresh_user_avatar` appends `?v=updated_at` so avatars are immutable-cacheable

**Complete example** (cache-busted asset URLs — the underrated half of the pattern, `config/routes.rb:57-59`):

```ruby
direct :fresh_user_avatar do |user, options|
  route_for :user_avatar, user, v: user.updated_at.to_fs(:number)
end
```

Used everywhere an avatar renders (`image_tag fresh_user_avatar_path(member)`), so `Cache-Control: public, max-age=30.days` (`production.rb:50-52`) is safe — changing your avatar changes the URL.

**Counter-example:** Caching whole pages (auth-dependent, uninvalidatable), or no caching plus N+1 partial renders; `update_all` without `updated_at` leaving stale fragments forever.

**When to break the rule:** Fragments rendering `Current.user`-dependent content can't share a record-keyed cache — either key on `[record, current_user-role]` or leave uncached. Campfire renders unread state via *classes injected by the caller* (`local_assigns[:unread]`, `_shared.html.erb:3`) partly to keep the cacheable part user-independent.

**Detection heuristic:** Record partials without `cache`; `update_all`/`insert_all` near views with fragment caching but no explicit `updated_at`; `touch: true` missing on belongs_to of things rendered inside parent fragments.

---

## Pattern: Optimistic UI — a server-rendered client template plus client-generated identity

**One-line rule:** For latency-sensitive creates, render a `<script type="text/template">` version of the partial with `$placeholder$` tokens for client-side instant insertion, and give records a client-generated id (`client_message_id`) that the server respects as identity.

**Why 37signals does it:** Chat must feel instant. The client inserts the message immediately from the template, then the Turbo Stream broadcast replaces/reconciles it — matched by `client_message_id`, which the model treats as its DOM key (`def to_key = [ client_message_id ]`). The template is ERB-rendered server-side, so avatar, name, and markup come from the same partial vocabulary as the real thing — with an explicit comment contract to keep them in sync.

**Citations:**
- `app/views/messages/_template.html.erb:1-31` — the client template with `$clientMessageId$`/`$body$` tokens
- `app/views/messages/_message.html.erb:1` — `<%# Be sure to check/update messages/_template.html.erb when changing this file %>`
- `app/models/message.rb:11` — `before_create -> { self.client_message_id ||= Random.uuid } # Bots don't care`
- `app/models/message.rb:21-23` — `def to_key; [ client_message_id ]; end` → `dom_id(message)` is client-stable
- `app/javascript/models/client_message.js` + `app/javascript/controllers/messages_controller.js:31` (`new ClientMessage(this.templateTarget)`)
- `db/structure.sql:32` — `client_message_id varchar NOT NULL` (identity enforced at the schema)

**Complete example** — the identity trick that makes reconciliation free:

```ruby
class Message < ApplicationRecord
  before_create -> { self.client_message_id ||= Random.uuid } # Bots don't care

  def to_key
    [ client_message_id ]
  end
end
```

Because `to_key` feeds `dom_id`, the optimistic client node (`id="message_<uuid>"`) and the broadcast server render produce the *same element id* — Turbo's append/replace then deduplicates naturally.

**Counter-example:** Spinner-until-server-responds UIs; or optimistic inserts keyed by nothing, producing duplicate messages when the broadcast lands.

**When to break the rule:** Most CRUD doesn't need it — Turbo Stream round-trips are fast enough for non-chat UX. Reach for this only where perceived latency is the product.

**Detection heuristic:** (For adopting, not violating.) Duplicate-element bugs after stream broadcasts = missing shared identity; hand-written client HTML diverging from the server partial = missing the sync comment/discipline.

---

## Pattern: Plain CSS, two-layer custom properties, and a handcrafted utility vocabulary

**One-line rule:** Ship vanilla CSS through Propshaft — one file per component, a `colors.css` defining raw values (`--lch-*`) that semantic tokens (`--color-*`) consume, dark mode by redefining only the raw layer, plus a small hand-rolled utility sheet.

**Why 37signals does it:** No build step, no framework churn, full use of modern CSS (oklch, nesting, `:where`, logical properties, view transitions). The two layers mean dark mode is ~12 lines — semantics like `--color-border` never change, only the raw palette flips. Utilities exist (`.flex`, `.gap`, `.txt-small`, `.for-screen-reader`) but as a *vocabulary they wrote and can read* — ~25 component files like `messages.css`, `composer.css` carry the real styling with BEM-ish names (`message__body`, `composer__input`).

**Citations:**
- `app/assets/stylesheets/colors.css:1-45` — raw `--lch-*` layer, semantic `--color-*` layer, dark-mode override of raw layer only
- `app/assets/stylesheets/utilities.css` — spacing tokens (`--inline-space: 1ch; --block-space: 1rem`) + utility classes
- `app/assets/stylesheets/base.css:20-30` — `:where(a:not([class]))` low-specificity defaults, CSS nesting
- one-file-per-component listing: `messages.css`, `composer.css`, `sidebar.css`, `boosts.css`, … (25 files)
- `app/views/layouts/application.html.erb:23` — `stylesheet_link_tag :all` (Propshaft: just load them all)
- customization hook: `app/helpers/application_helper.rb:15-19` — account-provided `custom_styles` injected as a `<style>` tag, viable *because* everything keys off custom properties

**Complete example** (`app/assets/stylesheets/colors.css`, the architecture in miniature):

```css
:root {
  /* Named color values */
  --lch-black: 0% 0 0;
  --lch-white: 100% 0 0;
  --lch-blue: 54% 0.23 255;

  /* Abstractions */
  --color-bg: oklch(var(--lch-white));
  --color-text: oklch(var(--lch-black));
  --color-link: oklch(var(--lch-blue));

  /* Redefine named color values for dark mode */
  @media (prefers-color-scheme: dark) {
    --lch-black: 100% 0 0;
    --lch-white: 0% 0 0;
    --lch-blue: 72.25% 0.16 248;
  }
}
```

**Counter-example:** Tailwind-everywhere (class soup in templates, a build dependency, dark-mode variants on every element) or SCSS with deep nesting and `@extend` chains nobody can trace. Both trade CSS literacy for tooling.

**When to break the rule:** Teams already fluent and invested in Tailwind shouldn't churn for style points — the transferable core is the *token layering* and one-file-per-component organization, which work under any system. Very large design systems may justify build-time tooling.

**Detection heuristic:** Hex colors inline in component CSS (should reference `--color-*`); a single 3,000-line `application.css`; dark mode implemented as per-component overrides instead of token redefinition.
