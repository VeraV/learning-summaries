# Learning Summaries

Personal learning summary pages with a dark, minimal design. Static HTML + shared CSS, no build tools.

## Project structure

```
index.html                          — Landing page (links to all summaries)
styles.css                          — Shared stylesheet for all pages
bash-zsh-summary.html               — Bash & Zsh summary
elasticsearch-kibana-summary.html   — Elasticsearch & Kibana summary
```

## Design system

### Fonts (Google Fonts, loaded in each HTML `<head>`)

- **DM Serif Display** — headings (h1, h2, takeaway h3)
- **DM Sans** — body text
- **JetBrains Mono** — code, badges, labels, table headers

```html
<link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Sans:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet" />
```

### CSS variables (defined in styles.css `:root`)

| Variable | Value | Use |
|---|---|---|
| `--bg` | `#0f1117` | Page background |
| `--surface` | `#181b24` | Card/component backgrounds |
| `--surface-raised` | `#1e222d` | Elevated surfaces, hover states |
| `--border` | `#2a2f3c` | Borders |
| `--text` | `#e4e6ed` | Primary text |
| `--text-muted` | `#8b90a0` | Secondary/body text |
| `--accent` / `--accent-soft` | `#00bcd4` | Cyan — links, highlights, default accent |
| `--green` / `--green-soft` | `#00c9a7` | Green |
| `--yellow` / `--yellow-soft` | `#fec514` | Yellow |
| `--pink` / `--pink-soft` | `#f04e98` | Pink |
| `--purple` / `--purple-soft` | `#b48eff` | Purple |
| `--orange` / `--orange-soft` | `#ff8a50` | Orange |

Each color has a `--soft` variant (low-opacity version) used for backgrounds.

### Color utility classes

Apply directly in HTML. Available for all 6 colors:

- Text: `.green-color`, `.yellow-color`, `.pink-color`, `.purple-color`, `.orange-color`, `.accent-color`
- Background: `.green-bg`, `.yellow-bg`, `.pink-bg`, `.purple-bg`, `.orange-bg`, `.accent-bg`

### Code block syntax spans

Inside `.code-block`, use these `<span>` classes for syntax highlighting:

| Class | Color | Use |
|---|---|---|
| `.cmd` | accent (cyan) | Commands |
| `.flag` | yellow | Flags, options |
| `.val` | green | Values |
| `.str` | purple | Strings |
| `.var` | orange | Variables |
| `.url` | green | URLs |
| `.json` | pink | JSON content |
| `.comment` | grey italic | Comments |

## How to create a new summary page

### 1. Pick two theme colors for the page

Each summary uses two colors from the palette for its topic titles and accents. Example: Bash page uses green + purple, ES page uses yellow + pink.

### 2. Create the HTML file

Use this skeleton:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>[Topic] - Learning Summary</title>
  <link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Sans:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap" rel="stylesheet" />
  <link rel="stylesheet" href="styles.css" />
  <style>
    /* Page-specific: header glow, title colors, takeaway gradient */
    .header::before {
      background:
        radial-gradient(circle at 30% 50%, rgba(COLOR1, 0.06) 0%, transparent 50%),
        radial-gradient(circle at 70% 50%, rgba(COLOR2, 0.04) 0%, transparent 50%);
    }
    .header h1 span.topic-a { color: var(--COLOR1); }
    .header h1 span.topic-b { color: var(--COLOR2); }
    .takeaway {
      background: linear-gradient(135deg, rgba(COLOR1, 0.08) 0%, rgba(COLOR2, 0.06) 100%);
      border: 1px solid rgba(COLOR1, 0.2);
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <div class="header-badge">Learning Session Summary</div>
      <h1><span class="topic-a">Topic A</span> & <span class="topic-b">Topic B</span></h1>
      <p>Subtitle describing what was learned</p>
    </div>

    <!-- Sections go here (see component reference below) -->

    <div class="takeaway">
      <h3>Key Takeaway Title</h3>
      <p>The most important insight from this topic.</p>
    </div>
  </div>
</body>
</html>
```

### 3. Build sections using these components

**Section with numbered icon:**
```html
<div class="section">
  <div class="section-header">
    <div class="section-icon green-bg"><span class="green-color">①</span></div>
    <h2>Section Title</h2>
  </div>
  <!-- cards, concept grids, code blocks, etc. -->
</div>
<div class="divider"></div>
```

Use circled numbers ①②③④⑤⑥⑦⑧⑨⑩⑪⑫ for section icons.

**Card:**
```html
<div class="card">
  <div class="card-title">Title</div>
  <p>Description text.</p>
</div>
```

**Card with list (arrow bullets):**
```html
<div class="card">
  <div class="card-title">Title</div>
  <ul>
    <li><strong>Term</strong> — explanation</li>
  </ul>
</div>
```

**Concept grid (2-column):**
```html
<div class="concept-grid">
  <div class="concept-card">
    <div class="label green-color">Label</div>
    <div class="value">Description text.</div>
  </div>
  <div class="concept-card">
    <div class="label purple-color">Label</div>
    <div class="value">Description text.</div>
  </div>
</div>
```

**Analogy box (left-bordered callout):**
```html
<div class="analogy-box">
  <div class="analogy-label">Key analogy</div>
  <p>Analogy or important note text.</p>
</div>
```

**Flow diagram (horizontal):**
```html
<div class="analogy-flow">
  <div class="flow-box" style="border-color: var(--green);">Step 1<br><span style="font-size:11px;color:var(--text-muted);">Detail</span></div>
  <div class="arrow">→</div>
  <div class="flow-box" style="border-color: var(--accent);">Step 2<br><span style="font-size:11px;color:var(--text-muted);">Detail</span></div>
</div>
```

**Flow diagram (vertical):** Add `style="flex-direction: column; gap: 8px;"` to `.analogy-flow` and `style="transform: rotate(90deg);"` to each `.arrow`.

**Code block:**
```html
<div class="code-block">
  <span class="comment"># Comment</span><br>
  <span class="cmd">command</span> <span class="flag">--flag</span> <span class="str">"value"</span>
</div>
```

**Reference table:**
```html
<table class="ref-table">
  <tr><th>Column A</th><th>Column B</th></tr>
  <tr><td>code-thing</td><td>Explanation</td></tr>
</table>
```

**Tags (inline badges):**
```html
<span class="tag green-bg green-color">Tag text</span>
```

### 4. Add a button to index.html

Add a new `.btn` entry inside `.buttons` in `index.html`. Pick a semantic class name for the button variant (e.g., `.btn-docker`) and add its color styles to `styles.css`:

```css
.btn-newpage {
  border-color: rgba(R, G, B, 0.25);
}
.btn-newpage:hover {
  border-color: var(--COLOR);
}
.btn-newpage .btn-icon {
  background: var(--COLOR-soft);
}
.btn-newpage .label {
  color: var(--COLOR);
}
```

## Style rules

- Do NOT add technology-specific prefixes to CSS variable names (use `--purple` not `--zsh-purple`)
- Color utility classes use generic names (`.green-color` not `.bash-color`)
- Header `<span>` classes for topic names ARE semantic (`.bash`, `.zsh`, `.es`, `.kb`) — defined per-page
- Page-specific styles (header glow gradient, title span colors, takeaway gradient) go in a small `<style>` block in the page's `<head>`, not in styles.css
- Inline `<code>` is automatically styled cyan with soft background — no extra classes needed
- The index page body must have `class="page-index"`; content pages have no body class
