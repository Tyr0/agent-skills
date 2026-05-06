---
name: rails-tailwind
description: Use this skill whenever the user asks about Tailwind CSS in a Ruby on Rails application, including setup, configuration, utility class patterns, extracting reusable styles, integrating Tailwind with ViewComponent or partials, responsive design, dark mode, custom theme values, or the tailwindcss-rails gem. Also use it for questions about purging unused styles, using @apply, building design systems with Tailwind in Rails, or Tailwind v3/v4 upgrade considerations. Triggers on 'how do I set up Tailwind in Rails', 'how do I extract a Tailwind component', 'how do I customize my Tailwind theme', 'how do I do dark mode in Rails', or any question about using Tailwind CSS in a Rails project.
---

# Rails Tailwind CSS Reference

A dense reference for integrating, configuring, and scaling Tailwind CSS in a Ruby on Rails application.

---

## Setup

### New Rails app

```bash
rails new myapp --css tailwind
```

Installs the `tailwindcss-rails` gem, generates `config/tailwind.config.js`, adds `app/assets/stylesheets/application.tailwind.css`, and configures the build process.

### Existing app

```bash
bundle add tailwindcss-rails
bin/rails tailwindcss:install
```

### Development build watcher

```bash
bin/dev   # starts both Rails server and Tailwind CSS watcher via Procfile.dev
```

The watcher rebuilds `app/assets/builds/tailwind.css` whenever a template changes.

### Production build

Tailwind compiles automatically during `assets:precompile`:

```bash
bin/rails assets:precompile   # runs tailwindcss:build as part of the pipeline
```

---

## Configuration

### `config/tailwind.config.js`

```javascript
const defaultTheme = require('tailwindcss/defaultTheme')

module.exports = {
  content: [
    './public/*.html',
    './app/helpers/**/*.rb',
    './app/javascript/**/*.js',
    './app/views/**/*.{erb,haml,html,slim}',
    './app/components/**/*.{erb,rb}',   // include ViewComponent templates
  ],
  theme: {
    extend: {
      fontFamily: {
        sans: ['Inter var', ...defaultTheme.fontFamily.sans],
      },
      colors: {
        brand: {
          50:  '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a5f',
        },
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('@tailwindcss/aspect-ratio'),
  ],
}
```

**Do:** Add `./app/components/**/*.{erb,rb}` to `content` when using ViewComponent — otherwise Tailwind purges classes used only in component templates.
**Don't:** Use string interpolation to build class names dynamically (`"text-#{color}-500"`) — Tailwind's content scanner is a text search, not a Ruby evaluator. Dynamic classes get purged.

### Safe-listing dynamic classes

```javascript
// config/tailwind.config.js
module.exports = {
  safelist: [
    'text-red-500',
    'text-green-500',
    'text-yellow-500',
    { pattern: /bg-(red|green|blue)-(100|500|900)/ },
  ],
}
```

Or use a lookup map in Ruby instead of dynamic interpolation:

```ruby
STATUS_CLASSES = {
  "active"   => "text-green-600 bg-green-50",
  "inactive" => "text-gray-500 bg-gray-50",
  "banned"   => "text-red-600 bg-red-50",
}.freeze

def status_classes(status)
  STATUS_CLASSES.fetch(status, "text-gray-500")
end
```

---

## Utility Class Patterns

### Consistent spacing scale

Use Tailwind's spacing scale consistently; avoid arbitrary values (`w-[137px]`) unless absolutely required for pixel-perfect third-party integration.

```erb
<%# Good — uses scale values %>
<div class="px-4 py-3 mt-6 mb-2">

<%# Avoid unless necessary %>
<div class="px-[17px] py-[11px]">
```

### Grouping classes by concern

Group utility classes in a consistent order to aid scanning:

1. Layout (`flex`, `grid`, `block`, `hidden`)
2. Positioning (`relative`, `absolute`, `top-*`, `z-*`)
3. Box model (`w-*`, `h-*`, `p-*`, `m-*`)
4. Typography (`text-*`, `font-*`, `leading-*`)
5. Visual (`bg-*`, `border-*`, `rounded-*`, `shadow-*`)
6. Interactive (`hover:`, `focus:`, `active:`)
7. Responsive (`sm:`, `md:`, `lg:`)

Many teams use the `prettier-plugin-tailwindcss` formatter to auto-sort classes.

### Extracting repeated patterns with partials

The primary extraction mechanism in Rails is a partial, not `@apply`:

```erb
<%# app/views/shared/_button.html.erb %>
<button class="inline-flex items-center px-4 py-2 border border-transparent
               text-sm font-medium rounded-md shadow-sm text-white bg-indigo-600
               hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2
               focus:ring-indigo-500 <%= local_assigns[:extra_classes] %>"
        <%= tag.attributes(local_assigns.except(:extra_classes, :label)) %>>
  <%= label %>
</button>
```

```erb
<%= render "shared/button", label: "Save", type: "submit" %>
<%= render "shared/button", label: "Cancel", extra_classes: "bg-gray-200 text-gray-700" %>
```

### Extracting with ViewComponent

ViewComponent is the preferred extraction mechanism for complex, interactive UI elements:

```ruby
# app/components/button_component.rb
class ButtonComponent < ViewComponent::Base
  VARIANTS = {
    primary:   "bg-indigo-600 text-white hover:bg-indigo-700",
    secondary: "bg-white text-gray-700 border border-gray-300 hover:bg-gray-50",
    danger:    "bg-red-600 text-white hover:bg-red-700",
  }.freeze

  def initialize(label:, variant: :primary, **html_options)
    @label = label
    @variant = variant
    @html_options = html_options
  end

  def variant_classes
    VARIANTS.fetch(@variant)
  end
end
```

```erb
<%# app/components/button_component.html.erb %>
<button class="inline-flex items-center px-4 py-2 text-sm font-medium
               rounded-md shadow-sm focus:outline-none focus:ring-2
               focus:ring-offset-2 focus:ring-indigo-500 <%= variant_classes %>"
        <%= tag.attributes(@html_options) %>>
  <%= @label %>
</button>
```

```erb
<%= render ButtonComponent.new(label: "Save", variant: :primary, type: "submit") %>
<%= render ButtonComponent.new(label: "Delete", variant: :danger, data: { confirm: "Sure?" }) %>
```

---

## Responsive Design

Tailwind is mobile-first. Unprefixed utilities apply at all breakpoints; prefixed utilities apply at that breakpoint and up.

```erb
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
  <%= render @posts %>
</div>
```

### Breakpoints

| Prefix | Min-width |
|---|---|
| `sm:` | 640px |
| `md:` | 768px |
| `lg:` | 1024px |
| `xl:` | 1280px |
| `2xl:` | 1536px |

Customize breakpoints in `tailwind.config.js`:

```javascript
theme: {
  screens: {
    tablet: '640px',
    laptop: '1024px',
    desktop: '1280px',
  },
},
```

---

## Dark Mode

### Configuration

```javascript
// config/tailwind.config.js
module.exports = {
  darkMode: 'class',  // 'media' uses prefers-color-scheme; 'class' uses .dark on <html>
  // ...
}
```

### Toggling dark mode

```erb
<%# Add/remove the `dark` class on <html> to toggle %>
<html class="<%= dark_mode? ? 'dark' : '' %>">
```

```javascript
// Stimulus controller for toggling
export default class extends Controller {
  toggle() {
    document.documentElement.classList.toggle('dark')
    localStorage.setItem('theme', document.documentElement.classList.contains('dark') ? 'dark' : 'light')
  }
}
```

Persist preference with localStorage and set the class before first paint to avoid flash:

```erb
<%# In <head> — before body renders %>
<script>
  if (localStorage.theme === 'dark' || (!('theme' in localStorage) && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
    document.documentElement.classList.add('dark')
  }
</script>
```

### Using dark variants

```erb
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
  <p class="text-gray-600 dark:text-gray-400">...</p>
</div>
```

---

## Typography Plugin

The `@tailwindcss/typography` plugin adds the `prose` class for rendering user-generated or CMS content:

```erb
<article class="prose prose-lg prose-indigo dark:prose-invert max-w-none">
  <%= @post.body_html %>
</article>
```

Useful for blog posts, documentation, and any rich text content rendered via ActionText or markdown.

---

## Forms Plugin

The `@tailwindcss/forms` plugin resets browser form styles so Tailwind utilities apply cleanly to inputs, selects, and textareas.

```bash
npm install @tailwindcss/forms
# or (with tailwindcss-rails)
```

Add to `plugins` array in `tailwind.config.js`. Then style inputs directly with utilities:

```erb
<%= f.text_field :email,
      class: "block w-full rounded-md border-gray-300 shadow-sm
              focus:border-indigo-500 focus:ring-indigo-500 sm:text-sm" %>
```

---

## Design Tokens / Theme

Define all brand values in `theme.extend` rather than using arbitrary values inline. This creates a consistent token set across the app.

```javascript
theme: {
  extend: {
    colors: {
      brand: {
        light: '#60a5fa',
        DEFAULT: '#2563eb',
        dark: '#1d4ed8',
      },
    },
    spacing: {
      18: '4.5rem',
      88: '22rem',
    },
    borderRadius: {
      '4xl': '2rem',
    },
    boxShadow: {
      card: '0 2px 8px 0 rgba(0,0,0,0.08)',
    },
  },
},
```

Reference custom tokens with the same utility class syntax:

```erb
<div class="bg-brand text-white shadow-card rounded-4xl p-18">
```

---

## `@apply` (use sparingly)

`@apply` lets you extract a group of utilities into a CSS class. Use it only for truly global, design-system-level abstractions — not as a default extraction pattern.

```css
/* app/assets/stylesheets/application.tailwind.css */
@layer components {
  .btn-primary {
    @apply inline-flex items-center px-4 py-2 text-sm font-medium rounded-md
           bg-indigo-600 text-white hover:bg-indigo-700 focus:outline-none
           focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500;
  }
}
```

**Do:** Use `@apply` for globally shared atomic components (`.btn-primary`, `.badge`, `.input`).
**Don't:** Use `@apply` to avoid repeating classes in partials — prefer ViewComponent or a partial instead. `@apply` breaks the utility-first model and makes classes opaque.

---

## Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| Dynamic class interpolation (`"text-#{color}-500"`) | Tailwind purges dynamically constructed class names; classes are missing in production | Use a lookup hash or `safelist` in config |
| Over-relying on `@apply` for everything | Defeats utility-first model; produces opaque class names; fights Tailwind's design | Extract to ViewComponent or partial instead |
| Omitting `./app/components/**/*` from `content` | ViewComponent template classes purged in production build | Add component paths to `content` array in config |
| Inline styles for values not in the scale | Inconsistent spacing/color; hard to maintain | Add custom tokens to `theme.extend` instead |
| Setting `!important` utilities everywhere | Indicates a specificity war, usually from third-party CSS | Scope third-party styles; use Tailwind's `important` strategy if needed |
| Using Tailwind without `prettier-plugin-tailwindcss` | Classes in random order; hard to scan or diff | Add the plugin; enforce with lint |
| Copying long class strings to every button/input | Duplication; one change requires find-and-replace across templates | Extract shared elements to partials or ViewComponents |
| Not running `bin/dev` during development | Tailwind build is stale; classes added to templates are missing | Always use `bin/dev` (or `foreman start`) which runs the Tailwind watcher |
