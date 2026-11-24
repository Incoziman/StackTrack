# StockTrack

Stock & asset register: React + TypeScript frontend, Fastify + TypeScript API, PostgreSQL.

Any item can be **part of** another item (rack › server › drives). Owners, locations and categories are hierarchies. Every change is recorded in a movement history. See **[docs/DATABASE.md](docs/DATABASE.md)** for the data model.

```
server/   API (Fastify, zod, pg) + SQL migrations + tests
web/      React app (Vite)
docs/     database design
```

## Run it for development (no Postgres needed)

Requires Node 20+.

```bash
npm install
npm run seed      # optional: example data (rack/server/drives, NVR kit, cable lots…)
npm run dev       # API on :3000, web on :5173 (proxies /api)
```

Without `DATABASE_URL`, the API uses **PGlite**, a full Postgres engine compiled to WASM, and stores data in `server/data/`. It's the same schema, triggers and SQL as production, with nothing to install.

To use a real Postgres in dev:
```bash
DATABASE_URL=postgres://user:pass@localhost:5432/stocktrack npm run dev
```

## Run it in production (Docker)

```bash
cp .env.example .env        # set POSTGRES_PASSWORD
docker compose up -d --build
# open http://<server>:3000
```
The app container serves both the API and the built web app, and runs migrations automatically on start.
Optional example data (only loads into an empty database): `docker compose exec app node server/dist/seed.js`

Without Docker: `npm run build`, then `DATABASE_URL=postgres://… npm start`.

**Backups:** `docker compose exec db pg_dump -U stocktrack stocktrack > backup.sql`

## Tests

```bash
npm test
```
Runs the API test suite twice: against a real PostgreSQL 17 server (started temporarily from the `embedded-postgres` package) and against PGlite.

## API overview

All JSON. Send an `X-Actor: <name>` header to have changes attributed in the log. Any create or update body may include `"reason": "…"`, which is stored as the log note.

| Method | Path | |
|---|---|---|
| GET/POST | `/api/products`, `/api/locations`, `/api/parties`, `/api/categories` | list (with paths / stock figures), create |
| GET/PATCH/DELETE | `/api/{resource}/:id` | read, update, delete (409 if still in use) |
| GET | `/api/items` | filters: `q, status, tracking, product_id, category_id, location_id, owner_id, assignee_id, parent_id (uuid or 'root'), hide_retired=1, sort, dir, limit, offset`. Location, owner and category filters include descendants. |
| POST | `/api/items` | create. Serialized: one unit. Bulk: merges into the existing lot at that place. Give `location_id` *or* `parent_id`. |
| GET | `/api/items/:id` | item + `ancestors` + direct `children` |
| GET | `/api/items/:id/tree` | full subtree |
| GET | `/api/items/:id/history` | the item's log (`?subtree=1` adds its components and what went in or out) |
| PATCH | `/api/items/:id` | tags, serial, status, owner, assignee, purchase info, notes (+ `reason`) |
| POST | `/api/items/:id/move` | `{location_id \| parent_id, quantity?, reason?}`. Moves a unit with its contents, or transfers part of a bulk lot. |
| POST | `/api/items/:id/adjust` | `{delta, reason?}`: receive or consume bulk stock |
| DELETE | `/api/items/:id?reason=` | for mistakes only; use status `retired` for disposals (history is kept either way) |
| GET | `/api/summary` | dashboard totals, low-stock list |
| GET | `/api/activity` | unified change log. Filters: `q, entity_type, entity_id, action, actor, since, until` (ISO times), `item_id` + `subtree=1`, `location_id` (in/out of it or anything below it), `party_id`, `limit, offset` |
| GET | `/api/activity/actors` | everyone who appears in the log |
| GET | `/api/activity/export.csv` | same filters, as a CSV file (opens in Excel) |

## Not included yet
* **Authentication.** Anyone who can reach the server can edit, and the name in the header is trusted. Put it behind your VPN or reverse-proxy auth, or add login before exposing it.
* CSV import of items, barcode/QR labels, attachments.
