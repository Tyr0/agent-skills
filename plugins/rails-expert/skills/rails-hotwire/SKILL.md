---
name: rails-hotwire
description: Use this skill whenever the user asks about Hotwire in Rails, including Turbo Drive (page navigation without full reloads), Turbo Frames (partial page updates), Turbo Streams (real-time or form-response HTML updates), Stimulus (lightweight JavaScript controllers), or morphing with Turbo 8. Also use it for questions about broadcasting Turbo Streams from models, responding to forms with streams, adding interactivity without a JavaScript framework, data-turbo attributes, Stimulus targets/values/actions/outlets, or SPA-like behavior in a server-rendered Rails app. Triggers on 'how do I use Turbo Frames', 'how do I broadcast updates', 'how do I add a Stimulus controller', 'how do I avoid a full page reload', or any question about Hotwire, Turbo, or Stimulus in a Rails context.
---

# Rails Hotwire Reference

A dense reference for building fast, interactive Rails applications using Turbo Drive, Turbo Frames, Turbo Streams, and Stimulus — without a separate JavaScript frontend.

---

## Hotwire Overview

| Component | Purpose | Transport |
|---|---|---|
| **Turbo Drive** | SPA-style page navigation (replaces full-page reloads) | HTTP |
| **Turbo Frames** | Scoped partial page updates | HTTP |
| **Turbo Streams** | Targeted DOM mutations from server | HTTP response or WebSocket |
| **Stimulus** | Lightweight JS controllers for behavior | Browser |

Hotwire ships by default in Rails 7+. Install with `bin/rails hotwire:install` on existing apps.

---

## Turbo Drive

Turbo Drive intercepts link clicks and form submissions, fetches pages via fetch(), and swaps only the `<body>` — giving native-like navigation speed.

### Opting out

```erb
<%# Disable Turbo for a single link %>
<%= link_to "Download", report_path(@report), data: { turbo: false } %>

<%# Disable Turbo for a form %>
<%= form_with model: @post, data: { turbo: false } do |f| %>
  ...
<% end %>

<%# Disable Turbo for an entire section %>
<div data-turbo="false">
  ...
</div>
```

### Progress bar

Turbo Drive shows a progress bar on slow navigations automatically. Customize in CSS:

```css
.turbo-progress-bar {
  height: 3px;
  background-color: theme('colors.indigo.500');
}
```

### Page refresh (Rails 7.1 / Turbo 8)

`turbo:morph` replaces only changed DOM nodes instead of swapping the full body. Enable per-page:

```erb
<%# In a view or layout, request morphing on refresh %>
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

Trigger a page refresh from the server:

```ruby
# In a controller action
redirect_to request.url, status: :see_other
# Turbo Drive will morph the page on return
```

### Caching

Turbo Drive caches the last visited page and shows it instantly on back-navigation (preview). Annotate pages that should not be cached:

```erb
<meta name="turbo-cache-control" content="no-cache">
```

---

## Turbo Frames

Turbo Frames scope navigation to a region of the page. Clicking a link inside a frame updates only that frame.

### Defining a frame

```erb
<%# app/views/posts/show.html.erb %>
<%= turbo_frame_tag @post do %>
  <h1><%= @post.title %></h1>
  <%= link_to "Edit", edit_post_path(@post) %>
<% end %>
```

The edit page must have a matching frame:

```erb
<%# app/views/posts/edit.html.erb %>
<%= turbo_frame_tag @post do %>
  <%= render "form", post: @post %>
<% end %>
```

Rails matches frames by ID. `turbo_frame_tag @post` generates `id="post_1"`.

### Lazy-loading frames

```erb
<%# The frame's src is fetched on page load %>
<%= turbo_frame_tag "recent_posts", src: posts_path(recent: true), loading: :lazy %>
```

Use lazy frames to defer expensive content below the fold without blocking the initial render.

### Targeting outside a frame

By default, navigation inside a frame stays in the frame. Use `data-turbo-frame` to break out:

```erb
<%# Navigate to a different frame %>
<%= link_to "Open in sidebar", post_path(@post), data: { turbo_frame: "sidebar" } %>

<%# Navigate the full page from inside a frame %>
<%= link_to "Full page", post_path(@post), data: { turbo_frame: "_top" } %>
```

### Frame-aware controller responses

If a request comes from inside a Turbo Frame, Rails can detect this:

```ruby
def edit
  @post = Post.find(params[:id])
  # request.headers["Turbo-Frame"] contains the frame ID if request is from a frame
end
```

---

## Turbo Streams

Turbo Streams deliver targeted DOM mutations using `<turbo-stream>` elements. Each stream element specifies an action and a target.

### Stream actions

| Action | Effect |
|---|---|
| `append` | Appends content inside the target element |
| `prepend` | Prepends content inside the target element |
| `replace` | Replaces the target element entirely |
| `update` | Replaces the content inside the target element |
| `remove` | Removes the target element |
| `before` | Inserts content before the target element |
| `after` | Inserts content after the target element |
| `refresh` | Triggers a Turbo Drive page refresh (Turbo 8) |

### Responding from a controller

```ruby
# app/controllers/posts_controller.rb
def create
  @post = Post.new(post_params)

  if @post.save
    respond_to do |format|
      format.turbo_stream  # renders app/views/posts/create.turbo_stream.erb
      format.html { redirect_to @post }
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts", partial: "posts/post", locals: { post: @post } %>
<%= turbo_stream.update "flash", partial: "shared/flash" %>
```

### Inline stream helpers

```erb
<%= turbo_stream.append "posts" do %>
  <%= render @post %>
<% end %>

<%= turbo_stream.remove dom_id(@post) %>

<%= turbo_stream.replace dom_id(@post) do %>
  <%= render @post %>
<% end %>
```

### Broadcasting from models (Action Cable)

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  after_create_commit -> { broadcast_prepend_to "posts" }
  after_update_commit -> { broadcast_replace_to "posts" }
  after_destroy_commit -> { broadcast_remove_to "posts" }

  # Or combine all three:
  broadcasts_to ->(post) { "posts" }
end
```

```erb
<%# Subscribe to the channel in the view %>
<%= turbo_stream_from "posts" %>

<%# The target container — Rails will broadcast into this element %>
<div id="posts">
  <%= render @posts %>
</div>
```

### Broadcasting with custom partial or target

```ruby
after_create_commit -> {
  broadcast_prepend_to "posts",
    target: "posts_list",
    partial: "posts/post_card",
    locals: { post: self }
}
```

### Broadcasting from a controller (explicit)

```ruby
def update
  @post = Post.find(params[:id])
  if @post.update(post_params)
    Turbo::StreamsChannel.broadcast_replace_to(
      "posts",
      target: dom_id(@post),
      partial: "posts/post",
      locals: { post: @post }
    )
    head :ok
  end
end
```

### `dom_id` helper

`dom_id` generates a predictable DOM ID string from an ActiveRecord object:

```ruby
dom_id(Post.find(1))       # => "post_1"
dom_id(Post.new)           # => "new_post"
dom_id(@post, :edit)       # => "edit_post_1"
```

Use `dom_id` as Turbo Stream targets to keep views and streams in sync.

---

## Stimulus

Stimulus is a lightweight JavaScript framework that attaches behavior to HTML elements via `data-controller`, `data-action`, and `data-target` attributes. No virtual DOM; no component lifecycle.

### Setup

Stimulus is included with Hotwire. Controllers live in `app/javascript/controllers/`.

```bash
bin/rails generate stimulus dropdown
# Creates app/javascript/controllers/dropdown_controller.js
# Registers it in app/javascript/controllers/index.js
```

### Controller anatomy

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged(value) {
    this.menuTarget.hidden = !value
  }
}
```

```erb
<%# Attach in HTML %>
<div data-controller="dropdown">
  <button data-action="click->dropdown#toggle">Menu</button>
  <ul data-dropdown-target="menu" hidden>
    <li>Item one</li>
    <li>Item two</li>
  </ul>
</div>
```

### Targets

```javascript
static targets = ["input", "output"]

// Access in methods:
this.inputTarget          // First matching element
this.inputTargets         // All matching elements
this.hasInputTarget       // Boolean — exists?
```

```erb
<input data-my-controller-target="input" type="text">
```

### Values

Values observe changes and call `<name>ValueChanged` callbacks automatically:

```javascript
static values = {
  url: String,
  count: { type: Number, default: 0 },
  active: { type: Boolean, default: false }
}

countValueChanged(value, previousValue) {
  this.element.textContent = `Count: ${value}`
}
```

```erb
<div data-controller="counter"
     data-counter-count-value="5"
     data-counter-url-value="<%= posts_path %>">
</div>
```

### Actions

Actions wire DOM events to controller methods:

```erb
<%# Syntax: event->controller#method %>
<button data-action="click->form#submit">Submit</button>
<input data-action="input->search#query keydown.enter->search#query">
<form data-action="submit->form#validate">
```

Common shorthand: `click` is the default action for buttons. `input` is the default for text inputs. `submit` is the default for forms. Omit the event when using defaults:

```erb
<button data-action="modal#open">Open</button>
```

### Outlets

Outlets connect controllers to other controllers on the page:

```javascript
// parent_controller.js
static outlets = ["child"]

childOutletConnected(outlet, element) {
  outlet.activate()
}
```

```erb
<div data-controller="parent" data-parent-child-outlet=".child-wrapper">
  <div class="child-wrapper" data-controller="child"></div>
</div>
```

### Lifecycle callbacks

```javascript
connect()    // Element connected to DOM (controller initialized)
disconnect() // Element removed from DOM
initialize() // Called once on first connect
```

Use `connect` / `disconnect` to set up and tear down event listeners or observers:

```javascript
connect() {
  this.observer = new IntersectionObserver(this.handleIntersection.bind(this))
  this.observer.observe(this.element)
}

disconnect() {
  this.observer.disconnect()
}
```

---

## Turbo + Stimulus Integration Patterns

### Flash messages via Turbo Stream

```erb
<%# app/views/shared/_flash.html.erb %>
<% flash.each do |type, message| %>
  <div data-controller="flash"
       data-flash-delay-value="3000"
       class="alert alert-<%= type %>">
    <%= message %>
  </div>
<% end %>
```

```javascript
// app/javascript/controllers/flash_controller.js
export default class extends Controller {
  static values = { delay: { type: Number, default: 5000 } }

  connect() {
    setTimeout(() => this.element.remove(), this.delayValue)
  }
}
```

```erb
<%# In a Turbo Stream response, update flash without full page reload %>
<%= turbo_stream.update "flash" do %>
  <%= render "shared/flash" %>
<% end %>
```

### Inline editing with Turbo Frames + Stimulus

```erb
<%# Show mode %>
<%= turbo_frame_tag dom_id(@post) do %>
  <div data-controller="inline-edit">
    <h2 data-inline-edit-target="display"><%= @post.title %></h2>
    <button data-action="click->inline-edit#startEditing">Edit</button>
  </div>
<% end %>
```

```erb
<%# edit.html.erb — same frame ID enables swap-in-place %>
<%= turbo_frame_tag dom_id(@post) do %>
  <%= render "form", post: @post %>
<% end %>
```

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Returning a `redirect_to` from a Turbo Stream format | Turbo ignores redirects on `text/vnd.turbo-stream.html` responses | Use `format.html { redirect_to ... }` separately; streams replace, not redirect |
| Missing matching `turbo_frame_tag` on the destination page | Frame navigation silently fails (no content swap) | Ensure the response contains a frame with the same ID |
| Broadcasting from a synchronous model callback | Broadcasts fire before transaction commits; subscribers may get stale or missing data | Use `_commit` callbacks (`after_create_commit`, etc.) |
| One Stimulus controller per element type | Proliferation of single-purpose micro-controllers | Design controllers around behavior patterns (toggle, form, reveal) not element names |
| Storing application state in Stimulus values on unrelated elements | Implicit coupling between distant DOM nodes | Use events (`this.dispatch`) or outlets for intentional cross-controller communication |
| Large amounts of JavaScript in Stimulus controllers | Defeats the purpose; hard to test | Keep controllers thin; move complex logic to imported modules |
| Using `turbo_stream_from` without authentication | Any authenticated user can subscribe to the stream | Scope broadcast streams to the current user or resource: `broadcast_to(current_user, ...)` |
| Forgetting `status: :unprocessable_entity` on failed form renders | Turbo treats all 2xx responses as success and replaces the frame | Always render with `status: :unprocessable_entity` on validation failure |
