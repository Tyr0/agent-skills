---
name: rails-db-best-practices
description: Use this skill whenever the user asks about Rails or Postgres database schema design, indexing strategies, association best practices, join tables, counter caches, or database anti-patterns. Also use it for questions about Postgres-specific features like JSONB, partial indexes, GIN/GiST indexes, generated columns, advisory locks, full-text search, or upsert. Triggers on code reviews of migrations or models where schema quality is a concern, or when a user asks 'is this schema efficient' or 'when should I add an index'. For N+1 queries, eager loading, and ActiveRecord query optimization use the rails-activerecord-queries skill instead.
---

# Rails & Postgres Database Best Practices

A dense reference for schema design, indexing, ActiveRecord associations, and Postgres-specific capabilities.

---

## Indexing

### Do's

- **Index every foreign key.** Rails does not add these automatically. A missing FK index causes full table scans on every `JOIN` and `WHERE` on the FK column.

  ```ruby
  # Good migration
  add_reference :posts, :user, null: false, foreign_key: true, index: true
  ```

- **Use `algorithm: :concurrently` on Postgres for zero-downtime index creation.** Standard `CREATE INDEX` holds a write lock; concurrent does not.

  ```ruby
  class AddIndexOnOrdersStatus < ActiveRecord::Migration[7.2]
    disable_ddl_transaction!

    def change
      add_index :orders, :status, algorithm: :concurrently
    end
  end
  ```

- **Prefer composite indexes when queries always filter on multiple columns together.** Column order matters — put the highest-cardinality or equality-filtered column first.

  ```ruby
  # Covers WHERE user_id = ? AND status = ?
  add_index :orders, [:user_id, :status]
  ```

- **Use partial indexes to keep indexes small and targeted.**

  ```sql
  -- Only index active users
  CREATE INDEX idx_users_active ON users (email) WHERE deleted_at IS NULL;
  ```

  Rails equivalent:
  ```ruby
  add_index :users, :email, where: "deleted_at IS NULL"
  ```

- **Add `unique: true` at the database level, not just with ActiveRecord validations.** Validations have a TOCTOU race condition under concurrent requests.

  ```ruby
  add_index :users, :email, unique: true
  ```

- **Cover frequently queried columns together to enable index-only scans.** If a query only touches columns `a` and `b`, a composite index `(a, b)` can satisfy it without hitting the heap.

### Don'ts

- **Don't index low-cardinality columns in isolation** (e.g., a boolean `active` on a large table). The planner will often prefer a seq scan; the index wastes space and slows writes.

- **Don't index every column "just in case."** Each index adds write overhead on `INSERT`, `UPDATE`, and `DELETE`. Only add indexes backed by real query patterns.

- **Don't forget to drop unused indexes.** Use `pg_stat_user_indexes` to find indexes with zero scans:

  ```sql
  SELECT relname, indexrelname, idx_scan
  FROM pg_stat_user_indexes
  WHERE schemaname = 'public'
  ORDER BY idx_scan ASC;
  ```

- **Don't use `add_index` without `algorithm: :concurrently` on production tables.** The default acquires a full table lock.

- **Don't rely on a leading composite index for single-column lookups on a non-leading column.** An index on `(user_id, created_at)` cannot efficiently satisfy `WHERE created_at = ?` alone.

---

## Associations

### Do's

- **Declare `belongs_to` with `optional: false` (the Rails 5+ default) to enforce presence at the model level.** Pair it with a database-level `NOT NULL` constraint and a foreign key.

  ```ruby
  belongs_to :user  # optional: false by default in Rails 5+
  ```

- **Use `has_many :through` for many-to-many with extra attributes on the join record.** Use `has_and_belongs_to_many` only when the join is a pure pivot with no extra data (rare in practice).

  ```ruby
  # Preferred
  class Enrollment < ApplicationRecord
    belongs_to :student
    belongs_to :course
    # extra columns: enrolled_at, grade
  end

  class Student < ApplicationRecord
    has_many :enrollments
    has_many :courses, through: :enrollments
  end
  ```

- **Use `dependent:` deliberately.** Choose based on the ownership semantics:
  - `dependent: :destroy` — load each record and call its `destroy` callbacks (safe but slow for large sets)
  - `dependent: :delete_all` — single `DELETE` SQL, skips callbacks (fast, use when no callbacks needed)
  - `dependent: :nullify` — sets FK to NULL, for optional ownership
  - `dependent: :restrict_with_error` — prevent parent deletion when children exist

- **Add `inverse_of` when Rails can't infer it** (non-standard naming, `:through` associations). Prevents unnecessary extra queries and ensures in-memory object identity.

  ```ruby
  has_many :authored_posts, class_name: "Post", foreign_key: :author_id, inverse_of: :author
  belongs_to :author, class_name: "User", inverse_of: :authored_posts
  ```

- **Use `counter_cache` for frequently displayed child counts.** Avoids a `COUNT(*)` query on every render.

  ```ruby
  class Comment < ApplicationRecord
    belongs_to :post, counter_cache: true
  end
  # Requires a `comments_count` integer column on posts with default: 0
  ```

  Migration:
  ```ruby
  add_column :posts, :comments_count, :integer, null: false, default: 0
  ```

- **Use polymorphic associations sparingly.** They break referential integrity (no real FK is possible) and complicate indexing. Prefer STI or separate join tables when the relationship set is bounded.

### Don'ts

- **Don't use `has_and_belongs_to_many` when you may need to add columns to the join table later.** Migrating to `has_many :through` is painful. Default to `has_many :through` with an explicit join model.

- **Don't call `.count` on an association when a counter cache exists.** `.size` uses the cache; `.count` always hits the database.

  ```ruby
  post.comments.size   # uses counter_cache if present
  post.comments.count  # always fires a COUNT query
  ```

- **Don't omit `null: false` on foreign key columns.** A nullable FK means a record that references nothing — almost always a data quality bug.

---

## Join Tables

### Do's

- **Name join tables in alphabetical order by convention** when using `has_and_belongs_to_many` (e.g., `assemblies_parts`, not `parts_assemblies`). For `has_many :through`, name the join model semantically (e.g., `enrollments`, not `courses_students`).

- **Always add a composite unique index on join tables** to prevent duplicate pivot rows.

  ```ruby
  add_index :enrollments, [:student_id, :course_id], unique: true
  ```

- **Add individual indexes on each FK in the join table** for efficient reverse lookups.

  ```ruby
  add_index :enrollments, :course_id
  # student_id is covered by the composite unique index
  ```

- **Use `insert_all` / `upsert_all` for bulk join record creation** to avoid N individual inserts.

  ```ruby
  Enrollment.insert_all(
    students.map { |s| { student_id: s.id, course_id: course.id, enrolled_at: Time.current } }
  )
  ```

### Don'ts

- **Don't omit a primary key from join tables without a reason.** Rails assumes a primary key; its absence breaks many ActiveRecord helpers. Only omit it if you're using the composite FK pair as a natural PK and you know the implications.

- **Don't allow duplicate rows in a join table without an explicit reason.** Missing the unique index will cause phantom duplication bugs when callbacks fire multiple times.

---

## Schema Design Best Practices

### Do's

- **Add `null: false` constraints to columns that must always have a value.** Database-level constraints are authoritative; model validations are a convenience layer.

- **Use `timestamps` on every table** (`created_at`, `updated_at`). The cost is two columns; the benefit is an audit trail and trivial cache key generation.

- **Prefer `string` with a `limit` or `text` intentionally.** `string` maps to `varchar(255)` by default; `text` is unbounded. Use `text` when you don't want a length limit; use `string` with `limit:` when you want the database to enforce it.

- **Use `bigint` (the Rails default) for primary keys.** If you anticipate > 2B rows, move to `uuid` from the start.

- **Use UUIDs for primary keys when records are created across multiple systems or when exposing IDs externally** (prevents enumeration attacks).

  ```ruby
  create_table :users, id: :uuid, default: "gen_random_uuid()" do |t|
    t.string :email, null: false
    t.timestamps
  end
  ```

- **Use `check` constraints to enforce domain invariants at the database level.**

  ```ruby
  add_check_constraint :orders, "amount > 0", name: "orders_amount_positive"
  ```

- **Normalize aggressively first; denormalize deliberately for performance.** Premature denormalization adds inconsistency risk without a proven query bottleneck.

### Don'ts

- **Don't store arrays or hashes in `text` columns as serialized strings.** Use JSONB (Postgres) or a proper join table instead.

- **Don't store money as `float`.** Floating-point precision errors will corrupt financial data. Use `decimal` with explicit `precision` and `scale`, or an integer number of cents.

  ```ruby
  # Bad
  t.float :price

  # Good
  t.decimal :price, precision: 10, scale: 2
  # Or: store as integer cents
  t.integer :price_cents, null: false, default: 0
  ```

- **Don't store time zones as offsets.** Store timestamps in UTC (`timestamptz` in Postgres) and convert in the application layer.

- **Don't add `default: nil` explicitly.** It is the implicit default; writing it adds noise with no benefit.

- **Don't use EAV (Entity-Attribute-Value) tables** as a schema escape hatch. They destroy query performance and type safety. Use JSONB or STI instead.

---

## Postgres-Specific Features

Postgres offers capabilities that generic ActiveRecord abstractions do not expose. Use them deliberately.

### JSONB

Store semi-structured data with full indexability. Unlike `json`, `jsonb` is stored as a binary tree and supports GIN indexing.

```ruby
# Migration
add_column :products, :metadata, :jsonb, null: false, default: {}
add_index :products, :metadata, using: :gin
```

```ruby
# Query
Product.where("metadata @> ?", { color: "red" }.to_json)
```

**Do:** Use JSONB for truly variable or sparse attributes (e.g., per-product custom fields).
**Don't:** Use JSONB as a substitute for normalized columns on data you need to reliably `GROUP BY`, `JOIN`, or aggregate. Postgres can query JSONB but the ergonomics are worse than structured columns.

### Array Columns

```ruby
add_column :posts, :tags, :string, array: true, default: []
add_index :posts, :tags, using: :gin
```

```ruby
Post.where("? = ANY(tags)", "rails")
```

**Do:** Use for small, bounded, unordered sets of scalars that are always queried together with the parent row.
**Don't:** Use arrays as a substitute for a proper association when you need to query individual elements frequently or maintain referential integrity.

### Partial Indexes

Already covered in the Indexing section. Partial indexes are a first-class Postgres feature — use them aggressively for filtered queries (e.g., `WHERE status = 'active'`, `WHERE deleted_at IS NULL`).

### GIN and GiST Indexes

| Type | Best for |
|---|---|
| GIN | JSONB containment (`@>`), array overlap (`&&`), full-text search (`tsvector`) |
| GiST | Geometric/range types, full-text search, nearest-neighbor |

```ruby
# Full-text search with GIN
add_column :articles, :search_vector, :tsvector
add_index :articles, :search_vector, using: :gin

# Keep it updated with a trigger (or in an AR callback)
execute <<~SQL
  CREATE OR REPLACE FUNCTION articles_search_vector_update() RETURNS trigger AS $$
  BEGIN
    NEW.search_vector :=
      to_tsvector('english', coalesce(NEW.title, '') || ' ' || coalesce(NEW.body, ''));
    RETURN NEW;
  END
  $$ LANGUAGE plpgsql;

  CREATE TRIGGER articles_search_vector_update
  BEFORE INSERT OR UPDATE ON articles
  FOR EACH ROW EXECUTE FUNCTION articles_search_vector_update();
SQL
```

### Generated Columns

Postgres 12+ supports stored generated columns — computed from other columns, persisted, and indexable.

```sql
ALTER TABLE users
  ADD COLUMN full_name text GENERATED ALWAYS AS (first_name || ' ' || last_name) STORED;
```

Rails 7.1+ exposes this:
```ruby
t.virtual :full_name, type: :string, as: "first_name || ' ' || last_name", stored: true
```

**Do:** Use for computed search or display fields that are expensive to recalculate and need to be indexed.

### Upsert

```ruby
# Rails 6+
User.upsert_all(
  [{ email: "a@b.com", name: "Alice" }, { email: "b@c.com", name: "Bob" }],
  unique_by: :email,
  update_only: [:name]
)
```

Postgres-native: `INSERT ... ON CONFLICT DO UPDATE`.

**Do:** Use for idempotent sync jobs, data imports, and background cache refreshes.
**Don't:** Use upsert as a substitute for proper business logic that must distinguish insert from update (e.g., when the two paths have different side effects).

### Advisory Locks

Lightweight application-level locks tied to the Postgres connection — no separate lock table needed.

```ruby
# Try to acquire a lock for a specific job class
def with_advisory_lock(key)
  result = ActiveRecord::Base.connection.execute(
    "SELECT pg_try_advisory_lock(hashtext('#{key}'))"
  ).first["pg_try_advisory_lock"]

  return unless result == "t"

  begin
    yield
  ensure
    ActiveRecord::Base.connection.execute(
      "SELECT pg_advisory_unlock(hashtext('#{key}'))"
    )
  end
end
```

**Do:** Use for distributed mutual exclusion (e.g., ensuring only one worker processes a cron job at a time).
**Don't:** Use as a general row-level lock — use `SELECT ... FOR UPDATE` for row locking.

### Range Types

Postgres has native range types (`int4range`, `tstzrange`, `daterange`, etc.) with overlap operators.

```sql
-- Find all events overlapping a period
SELECT * FROM events WHERE duration && '[2024-01-01, 2024-01-31)'::daterange;
```

**Do:** Use for scheduling, billing periods, and any domain with overlap/containment semantics.
**Don't:** Model date ranges as `starts_at` + `ends_at` columns if you frequently need overlap queries — range types give you index-backed operators for free.

### `EXPLAIN ANALYZE`

Always run `EXPLAIN (ANALYZE, BUFFERS)` on slow queries before tuning.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
```

Key things to look for:
- `Seq Scan` on large tables — indicates a missing or unusable index
- `Nested Loop` with large row estimates — may indicate a missing index on the inner relation
- High `Buffers: shared hit` vs `read` ratio — cache hit rate
- `rows=X (actual rows=Y)` mismatch — stale statistics; run `ANALYZE tablename`

---

## Migrations: Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Adding `NOT NULL` column with no default in one step | Locks the table during backfill on Postgres < 11 | Add nullable, backfill, then add constraint |
| `remove_column` without deploying code that stops using it first | Causes `ActiveRecord::StatementInvalid` in running instances | Two-phase deploy: code change first, then migration |
| Renaming a column in one migration | Old column gone before app is updated | Add new column, dual-write, backfill, remove old |
| Running `add_index` without `algorithm: :concurrently` on production | Full table write lock | Always use concurrent index creation |
| Data migrations inside schema migrations | Couples schema and data; dangerous on rollback | Use a separate Rake task or a dedicated data migration gem |
| `execute` raw SQL in a `change` migration without `reversible` | `db:rollback` will crash | Wrap in `reversible { |dir| dir.up { ... }; dir.down { ... } }` |

---

## Rails + Postgres: Integration Checklist

- [ ] `config.active_record.schema_format = :sql` — required when using Postgres-specific features (triggers, custom types, views). Commit `structure.sql`.
- [ ] All foreign keys have database-level `REFERENCES` constraints (`foreign_key: true` in migrations).
- [ ] All foreign key columns have indexes.
- [ ] All unique constraints are enforced at the database level, not just via AR validations.
- [ ] `NOT NULL` on all columns that must have a value.
- [ ] Money stored as `decimal` or integer cents — never `float`.
- [ ] Timestamps stored in UTC; `config.time_zone` and `config.active_record.default_timezone = :utc` set in `application.rb`.
- [ ] Concurrent indexes used for all production index additions.
- [ ] `EXPLAIN ANALYZE` run on all new queries touching tables > 10k rows.
