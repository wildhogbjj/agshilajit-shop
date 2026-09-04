# agshilajit-shop — working agreement

**This repo is worked on from more than one device and more than one Claude session** (Windows PC, phone/cloud sessions, occasionally two local sessions at once). Sessions share nothing automatically. **The git remote is the only shared state.** Treat it that way.

---

## Start of every session — do this before editing anything

```bash
git fetch origin
git status --short                          # uncommitted work from another session?
git log --oneline origin/main..HEAD         # local commits never pushed?
git log --oneline HEAD..origin/main         # commits from another device?
ls .git/*.lock 2>/dev/null                  # stale lock from a crashed session?
```

Report what you find **before** starting new work. Specifically:

- **Unpushed local commits** → say so and ask whether to push before building on them. This has already caused an hour of invisible work.
- **Uncommitted changes you didn't make** → do not overwrite. Show them and ask.
- **A `.git/*.lock` file** → check it's stale (`ls -la` for size/age, plus no `git.exe` in `tasklist`) before removing it. Never delete a lock while another git process is live.

## End of every session — push

Do not leave commits sitting locally. A commit that isn't pushed does not exist as far as any other device is concerned. If the user hasn't asked for a push, **say explicitly** that there are unpushed commits so they can decide.

---

## Deployment

- GitHub Pages, **building from branch `main`** — no Actions workflow. Pushing to `main` deploys.
- Confirm the build actually succeeded, don't assume:
  ```bash
  gh api repos/wildhogbjj/agshilajit-shop/pages/builds/latest --jq '{status,error:.error.message,commit}'
  ```
- Live at https://shop.wildhogbjj.co.uk — verify changes with `curl` after the build reports `built`.
- **There is no Ruby/Jekyll toolchain on the Windows PC.** You cannot build locally there; a broken Liquid tag will only surface as a failed Pages build. Prefer core Liquid over plugin tags, and check the build after pushing.

## Repo gotchas — these have each broken the live site once

- **All asset paths must be root-relative** (`/styles.css`, `/images/…`). Sub-pages use directory-style permalinks (`/ag-shilajit/`), so a relative path resolves against the subdirectory and 404s, leaving the page unstyled.
- **The root `.html` pages do not use `_layouts/default.html`.** Header and footer markup is duplicated in each. Changing shared chrome means editing all five. Only the `.md` pages (`privacy`, `terms`, `app`) use the layout.
- **`index.html` has no front matter**, so Jekyll does not process Liquid in it. Don't add `{% … %}` there without also adding front matter.
- `_config.yml` has **two `defaults:` keys**; YAML keeps only the last, so the first block is silently dead. Still unfixed.

## Content rules

- **Never invent product information** — ingredients, dosage, servings, storage, batch numbers, expiry dates. These are regulated. If a value isn't confirmed, say it will be published before launch; do not guess.
- Certification and origin claims (GMP Certified, Third-Party Tested, independently lab tested, Made in the UK) and the lab certificate image are **on hold pending a new certificate**. Do not add, edit, or remove them without an explicit instruction.
- No Stripe/PayPal checkout. The site uses an email-a-payment-link flow via Formspree.
- The Meta Pixel ID is still the placeholder `YOUR_PIXEL_ID` in six files.

See `README.md` for pixel event details, the Conversions API situation, and order-reference format.
