# NULGlow (website + API)

One project, one server. The React website lives in `web/`, the API in `src/`, and in production the API server also serves the built website, so there is a single thing to host.

Stack: React 19 + Vite + Tailwind v4 (web) · Node 20+ · Express 5 · TypeScript · SQLite · zod (api).

The API does two jobs:

1. **Serves the catalog** that used to be hard-coded in the frontend's `lib/data.ts` (products, brands, routines, concerns, deals, plans, stats).
2. **Handles what a static site can't:** the contact form, newsletter sign-up, student sign-up dialog, and student reviews (previously `localStorage`).

## Run it locally

```bash
npm run install:all          # installs api + web dependencies
cp .env.example .env         # optional for local use

# Option A: development (hot reload). Two terminals:
npm run dev                  # API on http://localhost:3001
npm run dev:web              # website on http://localhost:5173 (proxies /api to the API)

# Option B: production-style, ONE server (this is what you will host):
npm run build                # builds web/ then compiles the API
npm start                    # website + API on http://localhost:3001
```

With no SMTP configured, emails are **printed to the server console** (including confirm links), so every flow works locally with zero setup. `npm test` runs the 29 API tests.

## Endpoints

All under `/api`. Errors are JSON: `{ "error": "...", "fields": { "email": "Enter a valid email address." } }`. Validation wording matches the frontend's own messages.

### Content (GET, cached 5 min)
| Endpoint | Notes |
|---|---|
| `/products` | Filters: `step`, `brand`, `q`, `minPrice`, `maxPrice`, `sort=price_asc\|price_desc\|name`, `limit` |
| `/brands` | Includes `productCount` and Maloti `priceRange` (what the Brands page computes) |
| `/routines?skin=Oily&budget=Under M300` | Steps + server-computed `total`. Add `&concerns=Acne and breakouts,Vitiligo` to include advice and resolved product picks |
| `/routines/options` | Valid skin types, budgets, concern labels |
| `/concerns` | Concern advice with picks resolved to full products |
| `/deals` | Optional `store`, `limit` |
| `/plans`, `/stats` | |

### Reviews
| Endpoint | Notes |
|---|---|
| `GET /reviews` | Approved reviews: newest student reviews first, then the starter four |
| `POST /reviews` | `{ name, course?, quote (15–300), rating (1–5) }` |

### Forms (POST, rate-limited to 15 / 15 min / IP)
| Endpoint | Body | What happens |
|---|---|---|
| `/contact` | `name, email, topic, message` | Saved + emailed to `CONTACT_INBOX` with Reply-To set to the sender |
| `/newsletter/subscribe` | `email` | Creates a *pending* subscriber, emails a confirm link (double opt-in) |
| `GET /newsletter/confirm?token=` / `/unsubscribe?token=` | | Small HTML result pages |
| `/signup` | `name, email, skinType?, concerns?` | Upserts the student, sends one welcome email that includes the newsletter confirm link |

All form bodies accept a hidden `website` field as a honeypot: bots that fill it get a fake success and nothing is stored.

### Admin (`Authorization: Bearer $ADMIN_TOKEN`)
`GET /admin/summary`, `/admin/contacts`, `/admin/subscribers`, `/admin/students`, `/admin/reviews` (all paginated with `limit`/`offset`; add `?format=csv` to export),
`PATCH /admin/contacts/:id {handled}`, `PATCH /admin/reviews/:id {status}`, `DELETE /admin/reviews/:id`.
If `ADMIN_TOKEN` is empty the admin API is disabled (503).

```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" "http://localhost:3001/api/admin/subscribers?format=csv" > subscribers.csv
```

## How the website talks to the API

`web/src/lib/api.ts` is a small typed client. It uses same-origin URLs (`/api/...`): in dev the Vite server proxies them to the API, and in production the API serves the site, so **no CORS or URL settings are needed**. Set `VITE_API_URL` only if you ever host the API somewhere else.

The four forms are already wired: contact form, footer newsletter, sign-up dialog, and the reviews carousel (starter reviews show instantly, then the live list loads from the API). Each form disables its button while sending and shows the server's error message if something fails, including "could not reach the server" when offline.

The routine builder, product shelf and deals still read the local `lib/data.ts`, so the site keeps working even if the API is down. They can be switched to `api.routine()`, `api.products()` and `api.deals()` later.

If you set `REVIEWS_AUTO_APPROVE=false`, new reviews wait for approval via the admin API, and the form tells the student so.

## Hosting the website on GitHub Pages

GitHub Pages is free and serves **static files only**: it can host `web/` but cannot run the API or keep the database. So the site goes on Pages and the API needs a separate home (a small server host, or a replacement for the forms). `.github/workflows/deploy-pages.yml` builds and publishes the site automatically on every push to `main`.

One-time setup:

1. Create a **public** repository on GitHub (Pages is free for public repos only) and push this project to it. `.env` and the database are git-ignored, so no secrets are published.
2. In the repo: **Settings → Pages → Source: GitHub Actions**.
3. Push to `main`. After a minute the site is live at `https://<your-username>.github.io/<repo-name>/`.
4. Once the API has an address, add it in **Settings → Secrets and variables → Actions → Variables** as `API_URL` (for example `https://nulglow-api.example.com`, no trailing slash), then re-run the workflow.
5. On the API host set `CORS_ORIGINS=https://<your-username>.github.io` (the origin only, no `/<repo-name>`) and `SITE_URL=https://<your-username>.github.io/<repo-name>`.

The site uses `#/page` routes, so refreshing or sharing a link to any page works on Pages with no extra setup.

Pages' terms don't allow sites whose main purpose is processing commercial transactions. NULGlow as built (information, deals, free sign-ups) is fine, but taking payments for the Glow/Hall plans would need to happen elsewhere.

## Configuration

See `.env.example`. The important ones: `PUBLIC_API_URL` and `SITE_URL` (both become your public address; used in email links), `ADMIN_TOKEN`, `SMTP_*`, `TRUST_PROXY=1` when behind a reverse proxy (otherwise rate limiting sees the proxy's IP for everyone), and `DATABASE_PATH` (put it on a persistent volume). The server serves `web/dist` automatically when it exists.

## Where things live

```
web/                  the React website (Vite). web/src/lib/api.ts talks to the API
src/data/catalog.ts   the API's copy of the catalog (from lib/data.ts); edit prices/deals in BOTH places for now
src/routes/           catalog, reviews, forms, admin
src/schemas.ts        all input validation
src/db.ts             SQLite schema (+ seeds the 4 starter reviews on first run)
src/services/mailer.ts  SMTP or console
tests/api.test.ts     integration tests
```

## Design decisions worth knowing

- **Catalog is code, not database.** Prices and deals change a few times a term; a deploy is fine for that, and it keeps the frontend and API from drifting. Moving it to tables later only touches `services/catalog.ts`.
- **Double opt-in** for the newsletter, and the subscribe endpoint answers identically for new and existing addresses so it can't be used to discover who is subscribed.
- **Sign-up implies interest in the weekly drop**, per the dialog's own copy, but the subscription still has to be confirmed via the link in the welcome email.
- **Submissions are saved before email is attempted.** An SMTP outage never loses a message.
- **Out of scope:** user accounts/login, payments for the Glow/Hall plans (those buttons just link to the contact page today), and image hosting.
