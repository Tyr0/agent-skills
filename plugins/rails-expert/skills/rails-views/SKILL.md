---
name: rails-views
description: Use this skill whenever the user asks about the Rails view layer, including ERB templates, partials, layouts, content_for/yield, view helpers, ViewComponent, Phlex, or how to structure and organize frontend code in a Rails app. Also use it for questions about rendering collections, passing locals to partials, form helpers, slot-based component APIs, component previews, or choosing between ERB/ViewComponent/Phlex for a given project. Triggers on 'how do I extract a partial', 'what is ViewComponent', 'how do I use content_for', 'should I use Phlex', 'how do I render a collection', or any question about organizing or reusing view code in Rails.
---

# Rails View Layer Reference

A dense reference for structuring and organizing the view layer in a Rails application — covering ERB, partials, layouts, helpers, ViewComponent, and Phlex.

---

## View Approach Decision Matrix

| Approach | Best for | Testability | Learning curve |
|---|---|---|---|
| **ERB + partials** | Prototypes, small teams, MVPs | Low (integration only) | Minimal |
| **ViewComponent** | Medium-large apps, reusable UI, design systems | High (unit-testable) | Moderate |
| **Phlex** | Teams preferring pure Ruby, maximum flexibility | High (unit-testable) | Steep |

Start with ERB. Migrate to ViewComponent where reuse and testability justify the overhead. Use Phlex only when the team actively prefers a Ruby DSL over templating.

---

## ERB Partials

### Extracting a partial

```erb
<%# app/views/posts/_post.html.erb %>
<article class="post">
  <h2><%= post.title %></h2>
  <p><%= post.body %></p>
</article>
```

```erb
<%# Render with an explicit local %>
<%= render partial: "posts/post", locals: { post: @post } %>

<%# Shorthand — infers partial name and local from the object %>
<%= render @post %>

<%# Inline shorthand with explicit path %>
<%= render "posts/post", post: @post %>
```

### Rendering collections

```erb
<%# Renders _post.html.erb for each item; passes `post` as local %>
<%= render partial: "posts/post", collection: @posts %>

<%# Shorthand — Rails infers partial from the collection element class %>
<%= render @posts %>

<%# With a spacer partial between items %>
<%= render partial: "posts/post", collection: @posts, spacer_template: "posts/divider" %>
```

Collection rendering issues one query per batch (not N queries) and batches fragment cache lookups when `cached: true` is added.

### Collection caching

```erb
<%= render partial: "posts/post", collection: @posts, cached: true %>
```

Rails issues a single `multi_read` to the cache store for all items. Ensure the partial's cache key is stable (i.e., the model has `updated_at`).

### Partial-local `object` variable

When rendering a collection, Rails sets a magic local named after the partial. `_post.html.erb` gets a `post` local automatically. You can also access it as `post_counter` (zero-based index):

```erb
<%# _post.html.erb %>
<%= post_counter + 1 %>. <%= post.title %>
```

---

## Layouts and `content_for` / `yield`

### Basic yield

```erb
<%# app/views/layouts/application.html.erb %>
<html>
  <head>
    <title><%= yield(:title) || "MyApp" %></title>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :title, @post.title %>

<h1><%= @post.title %></h1>
```

### Named content blocks

```erb
<%# In a view: push content into a named slot %>
<% content_for :sidebar do %>
  <nav>...</nav>
<% end %>

<%# In the layout: render it %>
<aside><%= yield(:sidebar) %></aside>
```

### Nested layouts

```erb
<%# app/views/layouts/admin.html.erb — inherits from application %>
<% content_for :body do %>
  <div class="admin-wrapper">
    <%= yield %>
  </div>
<% end %>

<%= render template: "layouts/application" %>
```

---

## View Helpers

Keep helpers stateless and pure. Extract complex logic to presenter/decorator objects rather than helpers.

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def post_status_badge(post)
    css = post.published? ? "badge-green" : "badge-gray"
    content_tag(:span, post.status.humanize, class: "badge #{css}")
  end
end
```

```erb
<%= post_status_badge(@post) %>
```

**Do:** Use helpers for small markup fragments tied to a specific domain concept.
**Don't:** Put business logic, queries, or conditional chains in helpers — use a service object or presenter instead.

### `tag` and `content_tag`

```ruby
tag.div(class: "container") { tag.p("Hello") }
content_tag(:ul, class: "list") { safe_join(items.map { |i| content_tag(:li, i) }) }
```

Prefer `tag.*` (Rails 5.1+) over `content_tag` for new code — it uses keyword arguments and is less prone to XSS from unsafe HTML interpolation.

---

## ViewComponent

ViewComponent (github.com/ViewComponent/view_component) is an open-source gem originally built by GitHub. It wraps a Ruby class + template into a testable, reusable UI unit.

### Setup

```ruby
# Gemfile
gem "view_component"
```

### Generating a component

```bash
bin/rails generate component Post title body published_at
# Creates:
#   app/components/post_component.rb
#   app/components/post_component.html.erb
#   test/components/post_component_test.rb
```

### Component anatomy

```ruby
# app/components/post_component.rb
class PostComponent < ViewComponent::Base
  def initialize(post:)
    @post = post
  end

  def status_label
    @post.published? ? "Published" : "Draft"
  end
end
```

```erb
<%# app/components/post_component.html.erb %>
<article>
  <h2><%= @post.title %></h2>
  <span class="badge"><%= status_label %></span>
</article>
```

```erb
<%# Render in a view %>
<%= render(PostComponent.new(post: @post)) %>

<%# Render a collection %>
<%= render(PostComponent.with_collection(@posts)) %>
```

### Slots

Slots define injectable regions within a component — equivalent to named slots in web components.

```ruby
class CardComponent < ViewComponent::Base
  renders_one :header
  renders_many :rows
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="card">
  <% if header? %>
    <div class="card-header"><%= header %></div>
  <% end %>
  <div class="card-body">
    <% rows.each do |row| %>
      <div class="card-row"><%= row %></div>
    <% end %>
  </div>
</div>
```

```erb
<%= render CardComponent.new do |card| %>
  <% card.with_header { "Card Title" } %>
  <% card.with_row { "Row one" } %>
  <% card.with_row { "Row two" } %>
<% end %>
```

### Unit testing

```ruby
# test/components/post_component_test.rb
class PostComponentTest < ViewComponent::TestCase
  def test_renders_title
    post = posts(:published)
    render_inline(PostComponent.new(post: post))
    assert_selector "h2", text: post.title
  end

  def test_shows_published_badge
    render_inline(PostComponent.new(post: posts(:published)))
    assert_selector ".badge", text: "Published"
  end
end
```

ViewComponent tests run without a full Rails request cycle — they are significantly faster than system tests for UI assertion coverage.

### Component previews

```ruby
# test/components/previews/post_component_preview.rb
class PostComponentPreview < ViewComponent::Preview
  def default
    render(PostComponent.new(post: Post.first))
  end

  def draft
    render(PostComponent.new(post: Post.new(title: "Draft Post", published: false)))
  end
end
```

Access at `/rails/view_components` in development.

### HTML attribute forwarding

Accept arbitrary HTML attributes and pass them through to the root element using `**html_options`:

```ruby
class ButtonComponent < ViewComponent::Base
  def initialize(label:, variant: :primary, **html_options)
    @label = label
    @variant = variant
    @html_options = html_options
  end
end
```

```erb
<%# app/components/button_component.html.erb %>
<button class="btn btn-<%= @variant %>" <%= tag.attributes(@html_options) %>>
  <%= @label %>
</button>
```

```erb
<%= render ButtonComponent.new(label: "Save", type: "submit", data: { turbo_confirm: "Sure?" }) %>
```

### Polymorphic slots

Slots can accept different component types, enabling flexible composition:

```ruby
class FeedComponent < ViewComponent::Base
  renders_many :items, types: {
    post:    { as: :post,    renders: PostItemComponent },
    comment: { as: :comment, renders: CommentItemComponent },
  }
end
```

```erb
<%= render FeedComponent.new do |feed| %>
  <% @posts.each   { |p| feed.with_post(post: p) } %>
  <% @comments.each { |c| feed.with_comment(comment: c) } %>
<% end %>
```

### Sidecar assets (scoped CSS/JS)

Place CSS or JavaScript next to the component file for scoped, co-located styling:

```
app/components/
  card_component.rb
  card_component.html.erb
  card_component.css          # scoped CSS
  card_component.js           # component-specific JS
```

Enable sidecar assets in the initializer:

```ruby
# config/initializers/view_component.rb
Rails.application.config.view_component.use_component_path_prefix = true
```

### i18n support

ViewComponent automatically scopes `t()` calls to the component's namespace:

```ruby
class AlertComponent < ViewComponent::Base
  # t("title") looks up en.components.alert_component.title
end
```

```yaml
# config/locales/en.yml
en:
  components:
    alert_component:
      title: "Alert"
      dismiss: "Dismiss"
```

### Stimulus integration

Components pair naturally with Stimulus. Define the `data-controller` in the template:

```erb
<%# app/components/dropdown_component.html.erb %>
<div data-controller="dropdown"
     data-dropdown-open-value="false">
  <button data-action="click->dropdown#toggle">
    <%= @label %>
  </button>
  <ul data-dropdown-target="menu" hidden>
    <%= content %>
  </ul>
</div>
```

---

## Phlex

Phlex (phlex.fun) lets you write views as pure Ruby classes using a DSL that mirrors HTML structure. No separate template file.

### Setup

```ruby
# Gemfile
gem "phlex-rails"
```

### Component anatomy

```ruby
# app/views/components/post_card.rb
class PostCard < Phlex::HTML
  def initialize(post:)
    @post = post
  end

  def view_template
    article(class: "post-card") do
      h2(class: "post-title") { @post.title }
      p(class: "post-body") { @post.excerpt }
      span(class: badge_class) { @post.status.humanize }
    end
  end

  private

  def badge_class
    @post.published? ? "badge badge-green" : "badge badge-gray"
  end
end
```

```erb
<%# Render in ERB %>
<%= render PostCard.new(post: @post) %>
```

All standard Ruby control flow works naturally:

```ruby
def view_template
  ul do
    @items.each do |item|
      li { item.name }
    end
  end
end
```

**Phlex vs ViewComponent:** Phlex has no template file (pure Ruby); ViewComponent keeps ERB templates. Phlex is more test-friendly for logic-heavy components; ViewComponent is easier to adopt incrementally from existing ERB.

---

## Forms

### `form_with`

```erb
<%= form_with model: @post, local: true do |f| %>
  <div>
    <%= f.label :title %>
    <%= f.text_field :title, class: "input" %>
  </div>

  <div>
    <%= f.label :body %>
    <%= f.text_area :body, rows: 8 %>
  </div>

  <%= f.submit "Save", class: "btn-primary" %>
<% end %>
```

`form_with` defaults to remote (Turbo-compatible) in Rails 6+. Pass `data: { turbo: false }` to opt out.

### Strong parameters reminder

```ruby
# Always whitelist in the controller, not in the view
def post_params
  params.require(:post).permit(:title, :body, :published)
end
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Business logic in ERB templates | Untestable, unmaintainable | Move to helper, presenter, or component method |
| Deep partial nesting (`_a` renders `_b` renders `_c`) | Hard to trace rendering path; no clear ownership | Flatten or use ViewComponent slots |
| Querying the database in partials | N+1 queries; partials fire per-record | Eager load all data in the controller before rendering |
| `content_for` for JavaScript snippets per partial | Unpredictable script ordering; breaks CSP | Use Stimulus controllers instead of inline JS |
| Fat helpers with instance variable access | Hidden dependencies; helpers break in isolation | Pass all data as arguments; use presenters for complex logic |
| Rendering a ViewComponent in a loop without `with_collection` | Creates N component instances without batched cache lookups | Use `ComponentClass.with_collection(array)` |
| Using `raw` or `html_safe` on user-supplied content | XSS vulnerability | Sanitize with `sanitize` helper or use `tag.*` helpers |
| Global layout state via `@instance_variables` in helpers | Implicit coupling between controllers and views | Use `content_for` or pass locals explicitly |
