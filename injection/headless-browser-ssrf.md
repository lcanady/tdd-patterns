---
id: injection-headless-browser-ssrf
domain: injection
severity: high
stack: "node.js"
date_added: 2026-03-26
project: tdd-audit
---

# Headless Browser SSRF (Puppeteer / Playwright / wkhtmltopdf)

## Problem
User-supplied URLs are passed directly to `page.goto()`, `chromium.launch()`, or PDF-rendering tools. An attacker supplies `http://169.254.169.254/latest/meta-data` or `file:///etc/passwd` and the headless browser fetches it from the server's network, leaking cloud credentials or local files.

```javascript
// WRONG
app.post('/screenshot', async (req, res) => {
  const browser = await puppeteer.launch();
  const page    = await browser.newPage();
  await page.goto(req.body.url);          // SSRF
  const img = await page.screenshot();
  res.send(img);
});
```

## Root Cause
Developers assume browser navigation is sandboxed from the server's internal network. It is not — the browser runs on the server and can reach any network the server can reach.

## Fix
```javascript
const ALLOWED_PROTOCOLS = new Set(['https:']);
const PRIVATE_IP = /^(127\.|10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|169\.254\.)/;

function assertSafeUrl(rawUrl) {
  const u = new URL(rawUrl);
  if (!ALLOWED_PROTOCOLS.has(u.protocol)) throw new Error('Only https:// allowed');
  if (PRIVATE_IP.test(u.hostname) || u.hostname === 'localhost') {
    throw new Error('Private/internal hosts not allowed');
  }
}

app.post('/screenshot', async (req, res) => {
  assertSafeUrl(req.body.url);
  // ... proceed
});
```

## Test
```javascript
test('rejects file:// URL with 400', async () => {
  const res = await request(app).post('/screenshot').send({ url: 'file:///etc/passwd' });
  expect(res.status).toBe(400);
});

test('rejects internal metadata IP', async () => {
  const res = await request(app).post('/screenshot')
    .send({ url: 'http://169.254.169.254/latest/meta-data' });
  expect(res.status).toBe(400);
});
```

## Detection
```
page\.goto\(\s*req\.|page\.goto\(\s*body\.|page\.goto\(\s*params\.
chromium\.launch[\s\S]{0,200}goto\(\s*(?:req|body|params|url)
wkhtmltopdf.*(?:req|body|url)
```
