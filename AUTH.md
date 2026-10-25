# Authentication (Milestone 2)

Library: **Better Auth** (maintained; scrypt password hashing, signed httpOnly session cookies, DB-backed sessions) with Prisma 7 + PostgreSQL.
Two modes, chosen by `AUTH_MODE` (anything else stops the server): `dev` (default, no database) and `better-auth`.

## What it enforces
- **No self-registration**: `emailAndPassword.disableSignUp` is on in the app. Only the five seeded accounts exist.
- **Role never comes from the browser or the session payload.** The signed-in email is mapped to the server-side directory (`src/server/accounts/directory.ts`), which holds the role. Unknown emails get no account.
- Sessions expire after **8 hours**; cookies are `Secure` in production; sign-in attempts are rate limited per IP and per email (8 per 5 minutes) with a generic error message.
- In `better-auth` mode `NEXORA_DEV_ACTOR` is **ignored**. In production, `dev` mode has no actor at all.
- All pages and actions still check permissions on the server (see `docs/MARKET_MAKER.md` for the matrix).

## Seeded accounts (development only)
| Email | Role |
|---|---|
| `user1@nexora.invalid`, `user2@nexora.invalid` | USER |
| `mm@nexora.invalid`, `mm2@nexora.invalid` | MM |
| `admin@nexora.invalid` | ADMIN |

`.invalid` is a reserved TLD, so these can never receive mail. The password for all five is **the value of `SEED_DEV_PASSWORD` that you choose** (12+ characters). It is not in the repository and the seed never prints it. There is no production admin account.

## Setup (Codespaces or local)
1. Database: `docker run --name nexora-pg -e POSTGRES_PASSWORD=nexora -e POSTGRES_DB=nexora_demo -p 5432:5432 -d postgres:16`
2. Packages: `npm i better-auth @prisma/client @prisma/adapter-pg pg` and `npm i -D @types/pg`
3. `.env`: `AUTH_MODE=better-auth`, `DATABASE_URL=postgresql://postgres:nexora@localhost:5432/nexora_demo`, `BETTER_AUTH_SECRET=<openssl rand -base64 32>`, `BETTER_AUTH_URL=<public URL>` (Codespaces: `https://<name>-3000.app.github.dev`, no trailing slash).
4. `npx prisma validate` then `npx prisma generate` then `npx prisma migrate dev --name auth`
5. Seed: `ALLOW_DEV_SEED=true SEED_DEV_PASSWORD='your-long-dev-password' npm run db:seed`
6. `npm run dev`, open `/login`.

## Not verified yet (authored without the libraries or a database)
The schema in `prisma/schema.prisma` follows Better Auth's core tables from memory: compare it with `npx @better-auth/cli generate` and prefer the CLI output if they differ. Option names were checked against the Better Auth and Prisma 7 docs, but nothing has run against a real database. Tests cover the pure logic (mode, role mapping, env validation, seed guard) and the wiring against stubs.

## Troubleshooting
- "Invalid origin"/CSRF errors in Codespaces: `BETTER_AUTH_URL` must equal the browser URL exactly; `https://*.app.github.dev` is trusted automatically when `CODESPACES=true`.
- Server won't start: the error lists exactly which variable is wrong (secrets are never echoed).
