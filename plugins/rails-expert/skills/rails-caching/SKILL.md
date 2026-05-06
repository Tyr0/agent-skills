---
name: rails-caching
description: Use this skill whenever the user asks about caching in Ruby on Rails, including fragment caching, Russian doll caching, low-level caching, HTTP caching, cache stores (Redis, Memcached, memory), cache expiration, cache invalidation, counter caches, Action Controller caching, or cache key design. Also use it for questions about when to cache vs when not to, avoiding stale data, cache stampedes, cache warming, or debugging cache misses. Triggers on questions like 'how do I cache this', 'why is my cache stale', 'should I use Redis or Memcached', or 'how does Russian doll caching work'.
---

# Rails Caching Best Practices

A dense reference for all caching layers in a Rails application — from HTTP headers down to low-level key/value caching.

---

## Caching Layers Overview

| Layer | Mechanism | Best for |
|---|---|---|
| HTTP / reverse proxy | `Cache-Control`, `ETag`, `Last-Modified` | Public pages, API responses, CDN edge caching |
| Page caching | Static file written to disk | Fully public, no auth, no personalization (rare today) |
| Action caching | Removed in Rails 4 | N/A — use HTTP caching instead |
| Fragment caching | `cache do` in views | Expensive rendered partials, collections |
| Low-level caching | `Rails.cache.fetch` | Database query results, computed values, external API calls |
| Counter cache | `counter_cache: true` on `belongs_to` | Association `.count` on frequently rendered parent records |
| Memoization | `@ivar ||= ...` | Per-request in-process deduplication |

Cache at the highest level that is correct. HTTP caching scales to the CDN; fragment/low-level caching only helps your app servers.

---

## Cache Stores

### Redis (recommended for production)

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  expires_in: 1.hour,
  race_condition_ttl: 10.seconds,  # prevents cache stampede
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error: #{exception.message}"
  }
}
```

**Do:** Set `race_condition_ttl` — it allows one request through on expiry while others serve stale data, preventing the thundering herd.
**Don't:** Point your cache Redis and your Sidekiq/session Redis at the same instance without separate databases or namespaces — cache eviction (`maxmemory-policy allkeys-lru`) will evict job queue data.

### Memcached

```ruby
config.cache_store = :mem_cache_store, "cache1.example.com", "cache2.example.com",
  { expires_in: 1.hour, compress: true }
```

Use Memcached when you need a pure LRU cache with simple multi-threaded performance. Use Redis when you need persistence, pub/sub, sorted sets, or data structures beyond key/value.

### Memory Store (development / testing only)

```ruby
config.cache_store = :memory_store, { size: 64.megabytes }
```

**Don't** use in production with multiple processes — each process has its own isolated store. Cached values will not be shared across Puma workers or dynos.

### Null Store (testing)

```ruby
# config/environments/test.rb
config.cache_store = :null_store
```

Disable caching in tests by default so cached values don't pollute assertions. Enable per-test with `Rails.cache.clear` and a real store only when testing caching behavior explicitly.

---

## Fragment Caching

Cache rendered HTML partials directly in views.

```erb
<% cache post do %>
  <%= render post %>
<% end %>
```

Rails derives a cache key from the model's `cache_key_with_version`, which includes `id` and `updated_at`. When the record is `touch`ed or updated, the key changes and the fragment is regenerated.

### Russian Doll Caching

Nest `cache` blocks so inner fragments can be reused when only an outer record changes.

```erb
<%# Outer: cache the post and all its comments %>
<% cache post do %>
  <h1><%= post.title %></h1>

  <%# Inner: each comment cached independently %>
  <% post.comments.each do |comment| %>
    <% cache comment do %>
      <%= render comment %>
    <% end %>
  <% end %>
<% end %>
```

When a comment is updated, only that comment's fragment is invalidated. The outer post key must also change — ensure the post `touches` its association:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true
end
```

**Do:** Use `touch: true` on `belongs_to` to propagate cache invalidation up the nesting hierarchy.
**Don't:** Forget to `touch` parent records — stale outer fragments will silently serve outdated inner content even after the inner record changes.

### Collection Caching

```erb
<%= render partial: "products/product", collection: @products, cached: true %>
```

Rails batches the cache lookups for the entire collection into a single `multi_read` call. Far more efficient than caching each item individually in a loop.

**Do:** Use `cached: true` on collection renders for large lists.
**Don't:** Use `cached: true` without ensuring the partial's cache key is stable and the model has `updated_at`.

### Cache Key Design

Rails auto-generates keys from `ActiveRecord::Base#cache_key_with_version`:

```
posts/42-20240315123456789
```

For compound or custom keys:

```ruby
cache [current_user, post, "sidebar"] do
  # ...
end
```

**Do:** Include all inputs that affect the rendered output in the cache key (locale, user role, feature flags if relevant).
**Don't:** Include volatile or high-cardinality values in cache keys (e.g., `Time.current`, request IDs) — you'll create unbounded cache growth and zero reuse.

---

## Low-Level Caching

Use `Rails.cache.fetch` for anything below the view layer.

```ruby
def expensive_data
  Rails.cache.fetch("expensive_data/#{id}", expires_in: 5.minutes) do
    ExternalApi.fetch(id)  # only called on cache miss
  end
end
```

### Options

| Option | Purpose |
|---|---|
| `expires_in:` | TTL; after this, the entry is treated as missing |
| `race_condition_ttl:` | On expiry, serve stale to all but one request while one recalculates |
| `force: true` | Bypass cache and recompute (useful in admin tasks) |
| `skip_nil: true` | Don't cache a nil return value (prevents negative caching) |
| `compress: true` | Gzip large values before storing |

```ruby
# Prevent caching nil (e.g., when an external API is temporarily down)
Rails.cache.fetch("user_profile/#{id}", expires_in: 10.minutes, skip_nil: true) do
  UserProfileService.fetch(id)
end
```

### Write and Read Explicitly

```ruby
Rails.cache.write("key", value, expires_in: 1.hour)
Rails.cache.read("key")
Rails.cache.delete("key")
Rails.cache.exist?("key")
Rails.cache.fetch_multi("key1", "key2") { |key| compute(key) }
```

`fetch_multi` issues one `multi_read` and only computes values for missing keys — use it instead of calling `fetch` in a loop.

---

## HTTP Caching

### `Cache-Control`

```ruby
class ArticlesController < ApplicationController
  def show
    @article = Article.find(params[:id])
    expires_in 1.hour, public: true
  end
end
```

- `public: true` — cacheable by CDNs and shared proxies
- `public: false` (default) — only cacheable by the browser (`private`)
- `no_store` — must not be cached anywhere
- `no_cache` — must revalidate with the server before using cached copy (not "don't cache")

### Conditional GET (`ETag` / `Last-Modified`)

Lets the browser or CDN ask "has this changed?" — the server returns `304 Not Modified` with no body if not.

```ruby
def show
  @article = Article.find(params[:id])
  if stale?(@article, public: true)
    respond_to do |format|
      format.html
      format.json { render json: @article }
    end
  end
end
```

`stale?` sets `ETag` and `Last-Modified` headers automatically from the model and short-circuits rendering if the client's copy is still fresh.

For manual control:

```ruby
fresh_when(@article, public: true)
```

**Do:** Use `stale?` / `fresh_when` on all public show actions. The savings from avoided renders and reduced payload transfer are free.
**Don't:** Use `public: true` on responses that vary by user identity, role, or session state — you will serve one user's data to another user through a shared cache.

---

## Memoization

In-process, per-request caching using instance variables.

```ruby
def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
```

For methods that may legitimately return `nil` or `false`, `||=` will re-execute on every call. Use `defined?` instead:

```ruby
def current_user
  return @current_user if defined?(@current_user)
  @current_user = User.find_by(id: session[:user_id])
end
```

**Do:** Memoize helper methods and service objects that are called multiple times per request with the same inputs.
**Don't:** Memoize across requests (e.g., in class variables or class-level instance variables on controllers) — you will share state between requests, causing data leaks.

---

## Cache Invalidation

Cache invalidation is hard. Design for it from the start.

### Expiry-Based (TTL)

Set an `expires_in` and accept brief staleness. Best for data that changes occasionally and where a short window of stale data is acceptable (product listings, counts, non-critical stats).

### Key-Based Expiration (preferred)

Embed a version signal in the cache key so old entries are naturally abandoned rather than explicitly invalidated.

```ruby
Rails.cache.fetch("user_#{user.id}_profile_#{user.updated_at.to_i}") do
  render_profile(user)
end
```

Old keys are never deleted — they expire naturally by TTL. This is the strategy Rails uses with `cache_key_with_version`.

**Do:** Prefer key-based expiration over explicit `delete` calls. Explicit deletes require synchronized deploys and are prone to race conditions.
**Don't:** Use keys without a version component for data that must be accurate after writes.

### Explicit Invalidation

Use when you cannot embed a version in the key (e.g., aggregated data, cross-model dependencies).

```ruby
Rails.cache.delete("dashboard_stats")
Rails.cache.delete_matched("user_#{user.id}_*")  # use sparingly — expensive on Redis
```

**Don't** rely on `delete_matched` in hot paths — it requires a full key scan on Redis and is O(N) on keyspace size.

### `touch`

Update `updated_at` without saving other attributes, causing key-based caches to invalidate:

```ruby
post.touch                        # updates updated_at
post.touch(:cached_at)            # updates a custom timestamp column
post.comments.each(&:touch)       # propagate down
```

Or automatically via associations:
```ruby
belongs_to :post, touch: true
```

---

## Anti-Patterns

### Caching Too Early

**Don't** add caching before profiling. Adding a cache introduces complexity, eventual consistency, and a new failure mode. Measure first; cache the specific bottleneck.

### Caching User-Specific Data in a Shared Cache Key

```ruby
# Bad — serves any user's dashboard to any other user hitting this key
Rails.cache.fetch("dashboard") { Dashboard.build(current_user) }

# Good — scoped to user
Rails.cache.fetch("dashboard/#{current_user.id}") { Dashboard.build(current_user) }
```

### Caching Inside a Transaction

Cache writes inside a database transaction may commit to the cache store before the transaction commits — or after a rollback, leaving stale cached data.

```ruby
# Bad
ActiveRecord::Base.transaction do
  order.save!
  Rails.cache.write("order_#{order.id}", order)  # may persist even if transaction rolls back
end

# Good — write to cache after the transaction
ActiveRecord::Base.transaction { order.save! }
Rails.cache.write("order_#{order.id}", order)
```

### Caching Entire ActiveRecord Objects

Serializing full AR objects is fragile — schema changes, method additions, or gem updates can cause deserialization errors or silently return objects missing new attributes.

```ruby
# Risky
Rails.cache.write("user_#{id}", User.find(id))

# Better — cache plain data structures
Rails.cache.write("user_#{id}", User.find(id).slice(:id, :email, :name, :role))
```

If you must cache AR objects, use short TTLs and accept that a deploy may cause a wave of cache misses while old serialized objects expire.

### Using `expires_in: nil` (No Expiry) Without a Rotation Strategy

Without a TTL, keys live until memory pressure evicts them (unpredictable) or you explicitly delete them. This causes unbounded cache growth and makes it easy to serve indefinitely stale data after bugs.

**Always** set `expires_in` unless you have an explicit key-based invalidation strategy in place.

### Cache Stampede (Thundering Herd)

When a popular cached entry expires, all concurrent requests miss simultaneously and all recompute, hammering the database.

**Fix:** Set `race_condition_ttl` on the Redis cache store (serves stale to all but one requester during recomputation):

```ruby
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  race_condition_ttl: 5.seconds
}
```

Or use `fetch` with `race_condition_ttl` per call:

```ruby
Rails.cache.fetch("key", expires_in: 10.minutes, race_condition_ttl: 5.seconds) do
  expensive_computation
end
```

---

## Cache Warming

For critical data that must be present on first request (e.g., after a deploy or cache flush):

```ruby
# lib/tasks/cache.rake
namespace :cache do
  desc "Warm critical cache entries"
  task warm: :environment do
    Product.published.find_each do |product|
      Rails.cache.fetch("product_#{product.id}", expires_in: 1.hour) do
        ProductSerializer.new(product).as_json
      end
    end
  end
end
```

Run as part of deploy (`bin/rails cache:warm`) before traffic is shifted to new instances.

---

## Debugging Cache Issues

### Enable Caching in Development

```bash
bin/rails dev:cache   # toggles caching on/off in development
```

This writes a `tmp/caching-dev.txt` flag file and restarts the server.

### Inspect Cache Keys in Logs

```ruby
# config/environments/development.rb
config.cache_store = :memory_store
config.action_controller.perform_caching = true
```

Look for `Cache read`, `Cache write`, and `Cache miss` log lines to trace hits and misses.

### Clear the Cache

```ruby
Rails.cache.clear   # in console or a Rake task
```

**Don't** run `Rails.cache.clear` in production without understanding the blast radius — it will cause a thundering herd on all downstream caches simultaneously.

### Verify Keys Manually

```ruby
Rails.cache.read("my/key")
Rails.cache.exist?("my/key")
```

---

## Quick Reference Checklist

- [ ] Cache store configured per environment (null in test, memory/file in dev, Redis in production)
- [ ] `race_condition_ttl` set on Redis store to prevent stampedes
- [ ] Fragment caches use `touch: true` on `belongs_to` to propagate invalidation
- [ ] Collection partials use `cached: true` for batch lookups
- [ ] `expires_in` set on all `Rails.cache.fetch` calls
- [ ] No user-specific data in shared cache keys
- [ ] No cache writes inside database transactions
- [ ] No full AR objects serialized into the cache (prefer plain hashes or value objects)
- [ ] `stale?` / `fresh_when` used on public controller actions
- [ ] `fetch_multi` used instead of looping over `fetch` for multi-key reads
- [ ] Cache warm task exists and runs as part of deploy for critical paths
