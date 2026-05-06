---
name: rails-activerecord-queries
description: Use this skill whenever the user asks about writing efficient ActiveRecord queries, avoiding N+1 problems, eager loading associations, batch processing large datasets, bulk insert/update/delete, or query optimization in Rails. Also use it for questions about includes/eager_load/preload differences, pluck, exists?, find_each, update_all, delete_all, to_sql, explain, or the Bullet gem. Triggers on 'how do I avoid N+1', 'what is the difference between includes and eager_load', 'how do I batch process records', 'why is my query slow', or any question about making ActiveRecord queries faster or correct.
---

# ActiveRecord Query Optimization Reference

A dense reference for writing correct, efficient ActiveRecord queries — covering N+1 prevention, batch processing, bulk operations, and query analysis.

---

## N+1 Queries

An N+1 query fires one query to load a collection, then one query per record to load an association — N+1 total queries for N records.

```ruby
# BAD — fires 1 query for posts + 1 per post for author = N+1
posts = Post.all
posts.each { |p| puts p.author.name }

# GOOD — 2 queries total (posts + users)
posts = Post.includes(:author)
posts.each { |p| puts p.author.name }
```

### `includes` vs `eager_load` vs `preload`

| Method | SQL Strategy | Use when |
|---|---|---|
| `includes` | Rails decides (preload or LEFT JOIN) | Default choice; let Rails optimize |
| `preload` | Always separate queries | Avoid Cartesian product with `has_many` + conditions |
| `eager_load` | Always LEFT OUTER JOIN | Need to `where` or `order` on the association |

```ruby
# includes — Rails chooses the strategy
Post.includes(:author, :comments)

# eager_load — forces JOIN, required when filtering on the association
Post.eager_load(:author).where(users: { active: true })

# preload — forces separate queries, avoids row duplication on has_many
Post.preload(:comments).where(published: true)
```

**When `includes` silently becomes `eager_load`:** If you add a `where` or `order` referencing the associated table, Rails switches to a JOIN. This can produce duplicate records when loading `has_many` associations with conditions. Use `preload` explicitly to force separate queries and avoid the duplication.

### Nested associations

```ruby
Post.includes(comments: :author)
Post.includes(:author, comments: [:author, :likes])
```

### Detecting N+1 with Bullet

```ruby
# Gemfile
group :development do
  gem "bullet"
end

# config/environments/development.rb
config.after_initialize do
  Bullet.enable = true
  Bullet.rails_logger = true
  Bullet.add_footer = true     # shows alerts in browser
  Bullet.raise = true          # raises an error in test
end
```

---

## Selecting Only Needed Columns

```ruby
# Avoids instantiating full AR objects — significant memory savings on large tables
User.select(:id, :email, :name).where(active: true)
```

Use `pluck` when you only need raw values (skips AR object instantiation entirely):

```ruby
User.where(active: true).pluck(:id, :email)
# => [[1, "alice@example.com"], [2, "bob@example.com"]]

User.pluck(:email)
# => ["alice@example.com", "bob@example.com"]
```

Use `ids` as a shorthand for `pluck(:id)`:

```ruby
Post.where(published: true).ids
```

---

## Presence Checks

```ruby
# Bad — fires COUNT(*) and loads the count into Ruby
if User.where(email: email).count > 0
if User.where(email: email).any?    # also fires a COUNT in most adapters

# Good — fires SELECT 1 ... LIMIT 1
if User.where(email: email).exists?
```

---

## Batch Processing Large Datasets

Never load an entire large table into memory. Use batching.

```ruby
# find_each — yields one record at a time
User.where(active: true).find_each(batch_size: 500) do |user|
  user.send_weekly_digest
end

# find_in_batches — yields arrays of records
User.find_in_batches(batch_size: 1000) do |batch|
  report.add_rows(batch)
end

# in_batches — yields ActiveRecord::Relation (Rails 5+)
User.in_batches(of: 200) do |relation|
  relation.update_all(notified: true)
end
```

Default `batch_size` for `find_each` / `find_in_batches` is 1000.

**Do not add `.order` to a `find_each` query** — it overrides the internal `ORDER BY id` that batching relies on. Rails 7+ raises if you try.

---

## Bulk Operations (no callbacks)

These are significantly faster for large updates/deletes, but **skip ActiveRecord callbacks and validations**.

```ruby
# update_all — single UPDATE SQL
Post.where(published: false).update_all(archived: true, archived_at: Time.current)

# delete_all — single DELETE SQL (skips callbacks and dependent: :destroy)
Session.where("expires_at < ?", 30.days.ago).delete_all

# destroy_all — loads records and calls destroy on each (callbacks fire, slow)
User.where(banned: true).destroy_all
```

Use `destroy_all` only when callbacks and `dependent:` associations must fire. Prefer `delete_all` for large purges where callbacks are not needed.

### Bulk insert with `insert_all` / `upsert_all` (Rails 6+)

```ruby
# Insert many rows in a single statement — skips callbacks, validations, and timestamps
Post.insert_all(
  [
    { title: "Post 1", user_id: 1, created_at: Time.current, updated_at: Time.current },
    { title: "Post 2", user_id: 1, created_at: Time.current, updated_at: Time.current },
  ]
)

# Upsert — INSERT ON CONFLICT DO UPDATE
User.upsert_all(
  [{ email: "alice@example.com", name: "Alice" }],
  unique_by: :email,
  update_only: [:name]
)
```

---

## Associations: Loading vs Querying

```ruby
# association_ids — returns cached IDs without loading full records
post.comment_ids

# .size — uses counter_cache if present, or COUNT; does not load association
post.comments.size

# .count — always fires COUNT(*) regardless of cache
post.comments.count

# .length — loads all records if not already loaded, then counts in Ruby
post.comments.length   # avoid
```

---

## Raw Values vs AR Objects

| Scenario | Use |
|---|---|
| Need single column values | `pluck(:col)` |
| Need multiple column values | `pluck(:col1, :col2)` |
| Need a key-value map | `each_with_object` on `pluck` result |
| Need IDs only | `.ids` |
| Need full AR object | `find` / `where` |

```ruby
# Map user IDs to emails without loading AR objects
User.where(active: true).pluck(:id, :email).to_h
# => { 1 => "alice@example.com", 2 => "bob@example.com" }
```

---

## SQL Injection Prevention

```ruby
# VULNERABLE — user input interpolated directly into SQL
User.where("email = '#{params[:email]}'")

# SAFE — parameterized placeholders
User.where("email = ?", params[:email])
User.where(email: params[:email])       # hash syntax (preferred for equality)
User.where("created_at > ?", params[:since].to_time)
```

---

## Query Analysis

```ruby
# See the SQL without executing
User.where(active: true).order(:email).to_sql
# => "SELECT \"users\".* FROM \"users\" WHERE \"users\".\"active\" = TRUE ORDER BY \"users\".\"email\" ASC"

# Run EXPLAIN
User.where(email: "a@b.com").explain
# Prints EXPLAIN output from Postgres

# Run EXPLAIN ANALYZE (executes the query)
ActiveRecord::Base.connection.execute("EXPLAIN (ANALYZE, BUFFERS) SELECT ...")
```

Key things to look for in `EXPLAIN ANALYZE`:
- `Seq Scan` on large tables — likely a missing index
- Row estimate mismatch (`rows=X` vs `actual rows=Y`) — stale stats; run `ANALYZE tablename`
- `Nested Loop` with large row estimates — missing index on the inner relation
- High `Buffers: shared read` — cold cache or large scans

---

## Scopes and Chainability

```ruby
class Post < ApplicationRecord
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_author, ->(user) { where(author: user) }
end

# Scopes chain
Post.published.recent.limit(10)
Post.by_author(current_user).published
```

Scopes that might return no records should be avoided — prefer class methods with explicit guards if nil is possible.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `.all.each` on large tables | Loads all records into memory; OOM | Use `find_each` |
| Association methods in loops without eager loading | N+1 queries | `includes` the association before the loop |
| `.count > 0` or `.any?` for presence | Full COUNT query | `exists?` — fires `SELECT 1 LIMIT 1` |
| `.length` on unloaded relation | Loads entire dataset | `.size` (uses cache/COUNT) or `.count` |
| `destroy_all` on large tables | Loads every record, fires callbacks one-by-one | `delete_all` if callbacks aren't needed |
| `update_all` without a scope | Updates every row in the table | Always scope before `update_all` |
| String interpolation in `where` | SQL injection | Parameterized queries or hash syntax |
| `includes` when filtering on association | Unexpected JOIN and row duplication | Use `eager_load` explicitly when adding `where` on the association |
| `select *` on wide tables | Fetches unused columns; wastes memory and bandwidth | `select` only needed columns or use `pluck` |
| `Post.all.map(&:id)` | Loads full AR objects to get IDs | `Post.ids` or `Post.pluck(:id)` |
