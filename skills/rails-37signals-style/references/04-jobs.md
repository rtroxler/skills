# Dimension D — Jobs & Background Work

> **Era note:** the audited snapshot ran Resque + resque-pool on Redis (Feb 2024,
> pre-Solid). Today the answer is **Solid Queue — the Rails 8 default** (DB-backed, no
> Redis). Nothing below depends on the backend; the pattern is the *shape* of jobs, which
> transfers across any ActiveJob adapter. Do not cargo-cult Resque from this codebase.

## Pattern: Jobs are three-line async transports; the domain owns the logic

**One-line rule:** A job's `perform` is a single delegation to a domain method or PORO; the model exposes a synchronous method and its `_later` enqueuing twin (`deliver_webhook`/`deliver_webhook_later`), the `_later` suffix mirroring ActiveJob's own vocabulary — and when the sync and async halves live on the *same* object and would otherwise collide, 37signals' `STYLE.md` names the sync half `_now` (`relay_now`/`relay_later`). The default is *synchronous*: enqueue only for slow, fan-out, or failure-isolated external I/O.

**Why 37signals does it:** Campfire ships an entire chat product with **two** job classes. The logic in a job class is untestable without the queue, invisible to the domain, and unusable synchronously. Keeping jobs as transports means: the console can run `bot.deliver_webhook(message)` directly; tests assert behavior on the model; retry/queue concerns stay in one layer. And most work simply never becomes a job — only webhooks (slow third-party HTTP) and push notifications (fan-out to thousands of subscriptions) cross the async line.

**Citations:**
- `app/jobs/bot/webhook_job.rb:1-5` — entire job: `def perform(bot, message) = bot.deliver_webhook(message)`
- `app/jobs/room/push_message_job.rb:1-5` — entire job: `Room::MessagePusher.new(room:, message:).push`
- `app/models/user/bot.rb:51-57` — the sync/async pair: `deliver_webhook_later` (guarded `perform_later`) above `deliver_webhook`
- `app/models/room.rb:72-74` — `push_later(message)` private method wraps the enqueue; called from the domain reaction `Room#receive`
- `app/jobs/application_job.rb` — untouched generator defaults (no retry framework accreted)
- jobs namespaced under their domain owner: `Bot::`, `Room::` — not a flat `app/jobs` of verbs
- the `_later`/`_now` convention: `fizzy STYLE.md §8 (Run async operations in jobs)`. Honest caveat: `_now` is the *documented* rule, but their actual code still uses a bare sync verb (`deliver` + `deliver_later`) far more than `_now` — reserve `_now` for the genuine same-object collision; a bare verb is fine otherwise.

**Complete example** (the full async stack for webhooks — all of it):

```ruby
# app/jobs/bot/webhook_job.rb
class Bot::WebhookJob < ApplicationJob
  def perform(bot, message)
    bot.deliver_webhook(message)
  end
end

# app/models/user/bot.rb
def deliver_webhook_later(message)
  Bot::WebhookJob.perform_later(self, message) if webhook
end

def deliver_webhook(message)
  webhook.deliver(message)
end
```

The guard (`if webhook`) lives at enqueue time — don't enqueue work that will no-op. The actual HTTP, timeouts, and reply handling live in `Webhook#deliver` (`app/models/webhook.rb:9-19`), which also handles its own failure mode by posting a timeout notice back into the room — *domain-level* error handling, not queue-level retry.

**Counter-example:** A 150-line `SendWebhookJob` containing HTTP client setup, payload building, retry conditionals, and response parsing — logic trapped in the queue layer. Or "async everything": enqueueing trivial DB writes, adding queue latency and failure modes to operations that were fine inline.

**When to break the rule:** Genuinely queue-shaped concerns (batching, scheduled recurrence, rate-limited drains) can justify logic-bearing jobs. High-volume pipelines may need explicit `retry_on`/`discard_on` tuning on the job — keep it declarative at the top, still delegating the work. Note Campfire also uses a third async mechanism deliberately: an in-process thread pool for web-push delivery (`config/initializers/web_push.rb`, `Rails.configuration.x.web_push_pool`) — when thousands of HTTP posts per message would swamp a queue, a dedicated pool with persistent connections is the honest tool.

**Detection heuristic:** Jobs >15 lines; business branching inside `perform`; models with no synchronous counterpart for job-invoked behavior (can you run it in the console without the queue?); `perform_later` called from deep inside model callbacks for non-external work.
