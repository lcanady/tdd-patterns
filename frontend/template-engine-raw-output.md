---
id: frontend-template-engine-raw-output
domain: frontend
severity: high
stack: "node.js, javascript"
date_added: 2026-03-26
project: tdd-audit
---

# Template Engine Raw / Unescaped Output (XSS)

## Problem
Template engines provide an "unescaped" output syntax that renders raw HTML. When user-controlled data flows through these sinks, attackers inject arbitrary scripts.

```pug
//- Pug WRONG
p!= userComment

//- EJS WRONG
<p><%- userComment %></p>

//- Handlebars WRONG
<p>{{{userComment}}}</p>

//- Vue WRONG
<div v-html="userComment"></div>
```

## Root Cause
Developers need to render rich content (HTML emails, markdown-converted text) and reach for unescaped output. They forget that the data originates from users or external sources.

## Fix
```pug
//- Pug — safe (auto-escaped)
p= userComment
```

```ejs
<%- /* only for trusted, sanitised content */ sanitize(userComment) %>
```

For rich HTML: sanitise with DOMPurify (browser) or `sanitize-html` (server) before rendering through the raw sink.

```javascript
const sanitizeHtml = require('sanitize-html');
const safe = sanitizeHtml(userComment, { allowedTags: ['b','i','em','strong'] });
```

## Test
```javascript
test('user comment with <script> is escaped in output', async () => {
  const res = await request(app)
    .post('/comment')
    .send({ body: '<script>alert(1)</script>' });
  const html = await renderPage('/comments');
  expect(html).not.toContain('<script>alert(1)</script>');
  expect(html).toContain('&lt;script&gt;');
});
```

## Detection
```
!=\s+(?:user|input|body|req\.|param)         # Pug unescaped
<%-\s*(?!sanitize)(?:user|input|body|req\.)  # EJS unescaped
\{\{\{(?:user|input|body|req\.)              # Handlebars triple-stache
v-html\s*=\s*"(?:user|input|body)           # Vue v-html
\|s(?:\s|$)                                  # Dust unescaped
```
