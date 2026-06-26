# Dimension G — Security

## Pattern: Guard every user-supplied URL fetch against SSRF — including redirects

**One-line rule:** Any server-side fetch of a user-provided URL must resolve the host and reject private/loopback/zero-net IPs, re-validate on **every redirect hop**, cap redirect count, allowlist content types, and enforce a max body size while streaming.

**Why 37signals does it:** Link unfurling means "fetch whatever URL a user pastes" — the canonical SSRF hole (hit the cloud metadata endpoint, scan the internal network, OOM the worker with a 10GB response). Campfire treats this as a first-class subsystem: a reusable `RestrictedHTTP` guard in `lib/`, used by an `Opengraph::Fetch` client that distrusts the response at every step. This is the most transferable security code in the codebase — most apps that unfurl/webhook/import-by-URL have this hole.

**Citations:**
- `lib/restricted_http/private_network_guard.rb:1-30` — DNS-resolving IP check; **fails closed**: unparseable URL → treated as private
- `app/models/opengraph/fetch.rb:29-33` — `follow_redirect` re-runs the guard per hop, `MAX_REDIRECTS = 10`
- `app/models/opengraph/fetch.rb:39-50` — streams body in chunks, bails past `MAX_BODY_SIZE = 5.megabytes` even when Content-Length lied (with a comment explaining exactly that)
- `app/models/opengraph/fetch.rb:52-66` — `response_valid?` = status AND content-type allowlist AND length
- `app/models/opengraph/location.rb:8-9` — the URL itself is an ActiveModel with `validate :validate_url, :validate_url_is_public` (validation at the trust boundary — see `01-models.md`)

**Complete example** (`lib/restricted_http/private_network_guard.rb`, entire module):

```ruby
require "resolv"

module RestrictedHTTP
  class Violation < StandardError; end

  module PrivateNetworkGuard
    extend self

    def enforce_public_ip(url)
      if private_ip?(url)
        raise Violation.new("Attempt to access private IP #{url}")
      end
    end

    def resolvable_public_ip?(url)
      !private_ip?(url)
    rescue Resolv::ResolvError
      false
    end

    private
      LOCAL_IP = IPAddr.new("0.0.0.0/8") # "This" network

      def private_ip?(url)
        ip = IPAddr.new(Resolv.getaddress(URI.parse(url).host))
        ip.private? || ip.loopback? || LOCAL_IP.include?(ip)
      rescue URI::InvalidURIError, IPAddr::InvalidAddressError, ArgumentError
        true
      end
  end
end
```

Note the shape of fail-closed: the rescue returns `true` ("yes, private, block it") for anything malformed.

**Counter-example:** `Net::HTTP.get(URI(params[:url]))` — one line, four vulnerabilities (internal fetch, redirect bypass, unbounded body, no type check). Also bad: checking the IP once but following redirects blindly — `http://innocent.example` 302→ `http://169.254.169.254/` defeats the first check.

**When to break the rule:** None for the guard itself. Add (Campfire doesn't): per-host timeouts you tune, and DNS-rebinding awareness if your threat model is hostile-targeted rather than drive-by (resolve once and connect to the resolved IP, not the hostname again).

**Detection heuristic:** `Net::HTTP`/`HTTParty`/`Faraday` receiving anything derived from `params`/user records without an IP guard; redirect-following clients (`open-uri`!) on user URLs; no size cap on fetched bodies.

---

## Pattern: Tokens come from Rails primitives — never hand-rolled crypto or token tables

**One-line rule:** Identity-bearing values use the framework's audited primitives: `has_secure_token` for opaque DB tokens, `signed_id(purpose:, expires_in:)` for expiring links, `cookies.signed` for client state, `SecureRandom` for invite codes — each with a *purpose* so tokens can't be replayed across features.

**Why 37signals does it:** Every primitive here ships with Rails, rotates with your `secret_key_base`, and has had more security review than anything you'd write. The QR-code device-transfer feature — sensitive! it logs you in! — is two methods, because `signed_id` already does expiry, signing, and purpose-binding. Note the *purpose* discipline: a `:transfer` signed id can't be used where some other `find_signed` expects a different purpose.

**Citations:**
- `app/models/session.rb:4` — `has_secure_token` (the session credential; unique-indexed at `db/structure.sql:63`)
- `app/models/user/transferable.rb:6-14` — `signed_id(purpose: :transfer, expires_in: 4.hours)` / `find_signed(id, purpose: :transfer)` — entire transfer-auth feature
- `app/controllers/concerns/authentication.rb:73` — `cookies.signed.permanent[:session_token] = { value: session.token, httponly: true, same_site: :lax }`
- `app/models/account/joinable.rb:13-15` — invite codes: `SecureRandom.alphanumeric(12).scan(/.{4}/).join("-")` (human-formatted, regenerable)
- `app/models/user/bot.rb:23-28` — bot keys as composite `"#{id}-#{bot_token}"` resolved via `active.find_by(id:, bot_token:)` (lookup includes the active check — deactivation revokes)
- `app/models/message.rb:11` — even idempotency keys: `Random.uuid`, no gem

**Complete example** (`app/models/user/transferable.rb`, entire file — an expiring login link feature):

```ruby
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

**Counter-example:** A `tokens` table with `token_type`, `expires_at`, manual `SecureRandom.hex` generation, sweep jobs, and a comparison that isn't constant-time — re-implementing `signed_id` with more code and at least one bug. Or JWTs for first-party session state.

**When to break the rule:** Tokens that must survive `secret_key_base` rotation, be revocable individually, or be queryable ("show my active API keys") need DB rows — which is exactly why *sessions* are a table here while *transfer links* are signed ids. Match storage to revocation needs.

**Detection heuristic:** `JWT.encode` for same-app sessions; hand-built token columns next to unused `has_secure_token`; signed/encrypted values without `purpose:`; `==` comparison on secrets (vs `secure_compare`/AR finders).

---

## Pattern: Deny by default at the edge; make exceptions declarative and auditable

**One-line rule:** Every request must pass authentication and bot-denial filters unless a controller *declares* otherwise (`allow_unauthenticated_access`, `allow_bot_access only: :create`); scoped lookups make unauthorized data unreachable even past the gate — and where 37signals relaxes a default, they document the tradeoff in place.

**Why 37signals does it:** The security posture is grep-able: search `allow_unauthenticated_access` and you have the complete public surface (sessions, first run, join-by-code, PWA assets). Bots are denied *everywhere* (`before_action :deny_bots`) and re-admitted per-action; their CSRF exemption is scoped to exactly the bot-key auth path. Defense-in-depth comes free from authorization-by-scoping (`02-controllers.md`). And honesty is part of the posture — the 20-year session cookie carries a comment explaining why (CSRF token persistence across reopened windows).

**Citations:**
- `app/controllers/concerns/authentication.rb:5-11` — `before_action :require_authentication`, `before_action :deny_bots`, and `protect_from_forgery with: :exception, unless: -> { authenticated_by.bot_key? }` (CSRF skipped *only* for bot-key requests)
- `app/controllers/concerns/authentication.rb:13-26` — the declarative escape hatches as class methods
- `app/controllers/messages/by_bots_controller.rb:2` — `allow_bot_access only: :create` (bots can post messages; nothing else)
- `app/controllers/users_controller.rb:5,27-29` — join flow still verifies `join_code`, `head :not_found` on mismatch (don't even confirm existence)
- `config/initializers/session_store.rb:1-4` — documented tradeoff: `# Persist session cookie as permament so re-opened browser windows maintain a CSRF token` / `expire_after: 20.years`
- `app/controllers/sessions_controller.rb:3` — `rate_limit to: 10, within: 3.minutes` on the credential-guessing endpoint specifically
- `Gemfile:52` + `bin/brakeman` — static analysis wired in (with a checked-in `config/brakeman.ignore`)

**Complete example** (`app/controllers/concerns/authentication.rb:5-26` — the gate and its declared exceptions):

```ruby
included do
  before_action :require_authentication
  before_action :deny_bots
  helper_method :signed_in?

  protect_from_forgery with: :exception, unless: -> { authenticated_by.bot_key? }
end

class_methods do
  def allow_unauthenticated_access(**options)
    skip_before_action :require_authentication, **options
  end

  def allow_bot_access(**options)
    skip_before_action :deny_bots, **options
  end

  def require_unauthenticated_access(**options)
    skip_before_action :require_authentication, **options
    before_action :restore_authentication, :redirect_signed_in_user_to_root, **options
  end
end
```

**Counter-example:** Opt-in auth (`before_action :authenticate!` added per controller — one forgotten controller is a breach); blanket `skip_before_action :verify_authenticity_token` for "the API"; rate limiting bolted on globally where it punishes legitimate traffic instead of guarding the one hot endpoint.

**When to break the rule (and what *not* to copy):** Campfire leaves the CSP initializer fully commented out (`config/initializers/content_security_policy.rb` — all defaults) — defensible for a self-hosted product where admins inject custom CSS/styles, but **you should enable a CSP** in a normal SaaS. The 20-year cookie trades session-fixation surface for PWA ergonomics; with a sessions table you can still revoke server-side, which is what makes it tolerable — don't copy the cookie lifetime without copying the revocable Session model.

**Detection heuristic:** Controllers adding auth filters individually rather than inheriting them; `skip_before_action :verify_authenticity_token` broader than one declared auth path; no rate limit on session/password endpoints; CSP absent with no recorded reason.
