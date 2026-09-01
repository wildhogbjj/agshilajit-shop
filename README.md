# agshilajit-shop

Static Jekyll site for **shop.wildhogbjj.co.uk**, hosted on GitHub Pages (build from branch, no Actions workflow). Plain HTML + CSS, one hand-written `styles.css`, no framework and no build step beyond Jekyll itself.

---

## Before this goes to customers — outstanding placeholders

Two kinds of placeholder are live in the source right now and **must be replaced**.

### 1. `YOUR_PIXEL_ID` — Meta Pixel

Appears **twice per file** (the `fbq('init', ...)` call and the `<noscript>` tracking image) in **six files**:

| File | Covers |
|---|---|
| `index.html` | homepage / shop |
| `ag-shilajit.html` | product page |
| `altai-shilajit.html` | product page (duplicate of the above for the "altai" keyword) |
| `ashwagandha-shilajit.html` | coming-soon page |
| `lions-mane-shilajit.html` | coming-soon page |
| `_layouts/default.html` | `/privacy/`, `/terms/`, `/app/` |

Find them all with:

```bash
grep -rn YOUR_PIXEL_ID . --include=*.html
```

### 2. `[TO CONFIRM]` — product and legal copy

Values that must come off the physical product label or be verified before publication:

- **Product pages** (9 markers each): ingredient declaration, recommended daily dose, servings per jar, storage conditions, allergen/extra warnings, batch number, best-before date, manufacturer name and registered address.
- **`terms.md`** (2 markers): estimated delivery times for standard and tracked delivery.

```bash
grep -rn "TO CONFIRM" . --include=*.html --include=*.md
```

**Batch number and best-before date must be updated on every product page for each new batch dispatched**, and must match what is printed on the jar the customer receives.

---

## Meta Pixel — what fires where

Standard base code sits in the `<head>` of every page, so **PageView** fires site-wide automatically.

| Event | Where | Trigger | Payload |
|---|---|---|---|
| `PageView` | all pages | base code, on load | — |
| `ViewContent` | 5 product/coming-soon pages | on load | `content_name`, `content_ids`, `content_type`, `value`, `currency: GBP` |
| `InitiateCheckout` | `index.html` | first `focusin` or `input` on `#order-form`, fires once per page load | `num_items`, live cart `value`, `currency: GBP` |
| `Lead` | `index.html` | after the Formspree `fetch` resolves with `res.ok` | `num_items`, `value`, `currency: GBP`, `order_reference` |

### Why `Lead` and not `Purchase`

No payment is taken on this Site — the customer submits the order form, then receives a payment link by email afterwards. At the moment the form succeeds, **no money has changed hands**, so firing `Purchase` there would report revenue that may never arrive and would train Meta's optimisation on the wrong signal.

If you later take payment on-site, move a `Purchase` event to the point where payment is actually confirmed. The `Lead` call in `index.html` is commented to that effect.

On the two coming-soon pages, `ViewContent` sends `value: 0.00` because those products have no price yet. Update it when pricing is set.

---

## Conversions API — NOT implemented

**The Conversions API is not wired up, and cannot be from this repository as it currently stands.**

The Conversions API requires server-side code: events are sent from a server to Meta's Graph API using an access token. That token is a secret and **cannot** live in this repository or in any client-side JavaScript — anything committed here is served publicly and anything in the browser is readable by visitors.

This site is fully static:

- GitHub Pages, building from branch — no Actions workflow, no server runtime.
- No serverless or edge function capability of any kind.
- No backend that could hold a secret or receive a webhook.

So there is nowhere to put the server-side send. Realistic options, in rough order of effort:

1. **Meta's Conversions API Gateway** — a hosted, no-code option. Meta runs the server-side piece; you point it at your pixel. Least work, has a cost.
2. **A single serverless function** on Cloudflare Workers, Netlify Functions, or Vercel — receives the order submission, forwards it to both Formspree and the Conversions API with the access token held as an environment variable. This is the natural fit if the site stays where it is; the domain is already likely to be behind Cloudflare.
3. **Move the form to a backend that supports it** — larger change, only worth it alongside real on-site checkout.

Whichever route is taken, browser-side and server-side events must share an `event_id` so Meta can deduplicate them. The pixel calls in `index.html` do not currently set one — that will need adding at the same time.

---

## Order references

Generated **client-side at submit time** in `index.html`, format:

```
WHAGS-20260901-143022-A7F3
      |        |      |
      date     time   3 random bytes (crypto.getRandomValues)
```

This replaced a public third-party counter service (`countapi`) that produced sequential `WHAGS-00001` numbers. That counter was a single point of failure in the checkout path, was rate-limited, and — being a public endpoint keyed by a guessable name — could be read or incremented by anyone, which also leaked total order volume.

The trade-off: references are no longer sequential, so **they cannot be used to count orders**. Uniqueness comes from the timestamp plus 24 bits of randomness.

---

## Order form fallback

`#order-form` carries a real `method="POST"` and `action="https://formspree.io/f/xljrqpjj"`. The JavaScript path (`preventDefault` + `fetch`) is still primary and is what normally runs.

The `action`/`method` exist **only** as a failure fallback: if the JS throws or is blocked, the browser performs a native POST to Formspree instead of falling back to a `GET` of the current page with the customer's name, address and phone number exposed in the URL query string.

Two consequences of the no-JS path, by design:

- No order reference is generated (that is JS-side), so those submissions arrive without one and need a reference assigning by hand.
- No `Lead` event fires.

If the Formspree endpoint ever changes, update it in **both** places: the `action` attribute and `FORMSPREE_ENDPOINT` in the script.

---

## Layouts

`_layouts/default.html` renders the Markdown pages (`privacy.md`, `terms.md`, `app.md`) with the shared header and footer. The `.html` pages at the repository root are standalone and do **not** use it — the header and footer markup is duplicated in each. Changing shared chrome means editing all of them.

## Known issues

- `_config.yml` contains **two `defaults:` keys**. YAML keeps only the last, so the first block — `sitemap: false` on the Google site-verification file — is silently discarded.
- Asset paths must be **root-relative** (`/styles.css`, `/images/...`). Sub-pages use directory-style permalinks, so a relative path resolves against the subdirectory and 404s.
