---
name: rails-routing
description: Use this skill whenever the user asks about Rails routing, including defining routes with resources/resource, nested routes, namespaces, scopes, route helpers (_path and _url), named routes, member/collection routes, route constraints, non-resourceful routes, or organizing large route files. Also use it for questions about resolving routing errors, inspecting routes, understanding how Rails routes a request, or REST design best practices. Triggers on 'how do I add a route', 'undefined method _path', 'no route matches', 'what does resources do', or any question about config/routes.rb.
---

# Rails Routing Reference

A dense reference for `config/routes.rb` — defining routes, generating helpers, and avoiding common mistakes.

---

## Resourceful Routes

### `resources` (plural — 7 standard routes)

```ruby
resources :articles
```

Generates: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`.

| Helper | HTTP | Path | Action |
|---|---|---|---|
| `articles_path` | GET | `/articles` | index |
| `new_article_path` | GET | `/articles/new` | new |
| `article_path(id)` | GET | `/articles/:id` | show |
| `edit_article_path(id)` | GET | `/articles/:id/edit` | edit |
| `articles_path` | POST | `/articles` | create |
| `article_path(id)` | PATCH/PUT | `/articles/:id` | update |
| `article_path(id)` | DELETE | `/articles/:id` | destroy |

### `resource` (singular — 6 routes, no `:id`)

Use when there's only ever one of the resource per context (e.g., a user's own profile, session).

```ruby
resource :profile
resource :session
```

Generates: `show`, `new`, `create`, `edit`, `update`, `destroy`. No `index` and no `:id` segment.

### `only:` / `except:`

```ruby
resources :photos, only: [:index, :show]
resources :comments, except: [:destroy]
```

Prefer `only:` — be explicit about what routes exist.

---

## Nested Routes

Use nested routes when a child resource is always accessed in the context of a parent.

```ruby
resources :articles do
  resources :comments
end
# => /articles/:article_id/comments/:id
```

**Shallow nesting** — avoids deeply nested paths for member actions. Only collection routes (index, new, create) are nested; member routes (show, edit, update, destroy) are top-level:

```ruby
resources :articles do
  resources :comments, shallow: true
end
# => GET  /articles/:article_id/comments      (index)
# => GET  /articles/:article_id/comments/new  (new)
# => POST /articles/:article_id/comments      (create)
# => GET  /comments/:id                        (show)
# => GET  /comments/:id/edit                  (edit)
# => PATCH /comments/:id                      (update)
# => DELETE /comments/:id                     (destroy)
```

**Rule: never nest more than one level deep.** Deeply nested routes produce unwieldy helper names and fragile URLs.

---

## Member and Collection Routes

### Member routes — operate on a single record (include `:id`)

```ruby
resources :articles do
  member do
    post :publish
    get  :preview
  end
end
# => POST /articles/:id/publish
# => GET  /articles/:id/preview
# Helpers: publish_article_path(article), preview_article_path(article)
```

One-liner for a single extra action:

```ruby
resources :articles do
  post :publish, on: :member
end
```

### Collection routes — operate on the collection (no `:id`)

```ruby
resources :articles do
  collection do
    get :archived
  end
end
# => GET /articles/archived
# Helper: archived_articles_path
```

---

## Namespaces and Scopes

### `namespace` — URL prefix + module + helper prefix all match

```ruby
namespace :admin do
  resources :users
end
# => /admin/users maps to Admin::UsersController
# Helper: admin_users_path
```

### `scope` — URL prefix only, no module or helper change

```ruby
scope "/api" do
  resources :users
end
# => /api/users maps to UsersController (no module)
# Helper: users_path (unchanged)
```

### `scope module:` — module only, no URL prefix

```ruby
scope module: :api do
  resources :users
end
# => /users maps to Api::UsersController
# Helper: users_path
```

### `scope as:` — helper prefix only

```ruby
scope as: :v1 do
  resources :users
end
# => /users maps to UsersController
# Helper: v1_users_path
```

### Combining all three — explicit API versioning:

```ruby
namespace :api do
  namespace :v1 do
    resources :users
  end
end
# => /api/v1/users maps to Api::V1::UsersController
# Helper: api_v1_users_path
```

---

## Named Routes

```ruby
get "/about", to: "pages#about", as: :about
# Helper: about_path, about_url
```

Named routes with `:as` on scoped `resources`:

```ruby
resources :photos, as: :images
# Helper: images_path, image_path(id)
```

---

## Route Constraints

### Format/regex constraints on parameters

```ruby
resources :articles do
  get :show, constraints: { id: /\d+/ }
end

get "/users/:username", to: "users#show", constraints: { username: /[a-z0-9_]+/ }
```

### Subdomain constraints

```ruby
constraints subdomain: "api" do
  namespace :api do
    resources :users
  end
end
```

### Request-based constraints (class-based)

```ruby
class MobileConstraint
  def matches?(request)
    request.user_agent =~ /Mobile/
  end
end

constraints MobileConstraint.new do
  root to: "mobile#index"
end
```

---

## Non-Resourceful Routes

```ruby
# Static route
get "/healthcheck", to: "health#show"

# Root
root "dashboard#index"

# Redirect
get "/old-path", to: redirect("/new-path")
get "/users/:id", to: redirect { |params, req| "/profiles/#{params[:id]}" }

# Catch-all (place last — matches everything)
get "*path", to: "errors#not_found"
```

---

## Organizing Large Route Files

Use `draw` to split routes into multiple files. Rails loads them in the order called.

```ruby
# config/routes.rb
Rails.application.routes.draw do
  draw :admin
  draw :api
  draw :webhooks
end
```

```ruby
# config/routes/admin.rb
namespace :admin do
  resources :users
  resources :reports
end
```

```ruby
# config/routes/api.rb
namespace :api do
  namespace :v1 do
    resources :articles
  end
end
```

---

## Inspecting Routes

```bash
bin/rails routes                          # all routes
bin/rails routes -g article               # filter by pattern
bin/rails routes -c ArticlesController    # filter by controller
bin/rails routes --expanded               # verbose output
```

In the console:

```ruby
Rails.application.routes.url_helpers     # access all named helpers
Rails.application.routes.recognize_path("/articles/1")
# => { controller: "articles", action: "show", id: "1" }
```

---

## Route Helpers

```ruby
# _path — relative URL (use in views and controllers)
articles_path           # => "/articles"
article_path(@article)  # => "/articles/42"
new_article_path        # => "/articles/new"

# _url — absolute URL (required in mailers, redirects from non-request contexts)
articles_url            # => "https://example.com/articles"
```

Pass query parameters:

```ruby
articles_path(page: 2, sort: :created_at)
# => "/articles?page=2&sort=created_at"
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Nesting more than one level deep | Long, fragile helper names; hard to maintain | Use `shallow: true` or flatten after the first level |
| `resources` without `only:`/`except:` | Exposes unwanted actions | Always declare intent explicitly |
| Using `get`/`post` for everything instead of `resources` | Loses RESTful conventions and generated helpers | Map to resources where possible |
| Named route conflicts (same `as:`) | Routing raises `ArgumentError` at boot | Keep `as:` names unique across the file |
| Large `config/routes.rb` (> 100 lines) | Hard to navigate, slow to grep | Use `draw` to split by domain |
| Catch-all route before other routes | Swallows legitimate routes silently | Always place catch-alls last |
| `redirect` without scope | Builds URL without host context in tests | Use `redirect { |p, req| ... }` for dynamic redirects |
| Calling route helpers in models | Models shouldn't know about URLs | Use decorators, presenters, or pass helpers explicitly |
