# AGENTS.md

Compact guide for OpenCode sessions working in this repo. Kutt is a URL shortener built on Express 4 + Handlebars + Knex (no ORM, no build step, no TypeScript). Pure CommonJS.

## Commands

- `npm run dev` — start dev server. Uses Node's built-in `--watch-path` (not nodemon) and only watches `server/` and `custom/`. Editing files outside those dirs requires a manual restart.
- `npm start` — production. Passes `--production`, which is parsed in `server/env.js:25` to set `NODE_ENV=production`. Do not assume `cross-env` or similar.
- `npm run migrate` — apply Knex migrations.
- `npm run migrate:make -- <name>` — create a new migration (note the `--`).
- `npm run docs:build` — regenerate the Redoc API site into `docs/api/static/` (gitignored). Requires the `redoc` devDependency to be installed.

There are **no** `test`, `lint`, `format`, or `typecheck` scripts and no test framework. Do not invent or run them; verify changes by booting the app and exercising the relevant route.

## Architecture

- Entry point: `server/server.js`.
- **`server/models/*.model.js` only define table-creation helpers — they are NOT the data-access layer.** All DB reads/writes go through `server/queries/*.queries.js` (barrel: `server/queries/index.js`). Do not add query logic to `models/`.
- Routes: `server/routes/routes.js` wires sub-routers. The same API router is mounted at **both `/api` and `/api/v2`** — there is one API version, not two.
- Migrations live in `server/migrations/` (configured in `knexfile.js`), not the Knex default `./migrations`. Docker images run `npm run migrate && npm start` automatically on boot (`Dockerfile`).
- Visit tracking goes through `server/queues/`: Bull when Redis is enabled, an inline fallback shim otherwise. Both paths are driven by `env.REDIS_ENABLED`.
- `server/cron.js` (link expiry sweep) runs only when `NODE_APP_INSTANCE === 0` to support pm2 cluster mode. If clustering with something else, set this env var on a single instance.
- Cookie-session middleware is installed **only when `OIDC_ENABLED=true`** (`server/server.js:46`).

## Multi-database queries

Queries must stay compatible with SQLite, Postgres, and MySQL/MariaDB. Use the dialect helpers rather than writing raw SQL by hand:

- `server/knex.js` attaches `db.isSQLite`, `db.isPostgres`, `db.isMySQL`, and `db.compatibleILIKE` (`andWhereILike` on Postgres, `andWhereLike` elsewhere).
- `server/utils/knex.js#truncatedTimestamp(column, precision)` emits the correct date-trunc SQL per driver.
- When writing migrations, remember `IF NOT EXISTS` is not supported on MySQL — see `server/migrations/20241223062111_indexes.js` for the pattern.

## Configuration

- All env vars are declared and defaulted in `server/env.js` (validated with `envalid`). Trust that file over the README if they disagree.
- `JWT_SECRET` is required in production (envalid `str({ devDefault })`) and falls back to `securekey` in dev.
- Any env var supports a `<NAME>_FILE` variant that reads the value from a file path (`server/env.js:79`).
- The `/custom` folder overrides built-in assets: `custom/css` and `custom/images` are served at `/css` and `/images`; `custom/views` is searched **before** `server/views` by hbs. `styles.css` in `custom/css` replaces the default stylesheet.

## Branches / release flow

- Default branch is `main`. Upstream also maintains `develop`, which the CI workflow tags as `development` on Docker Hub. Releases publish versioned + `latest` tags. When in doubt about PR target branch, ask.
