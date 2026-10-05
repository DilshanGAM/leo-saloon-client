# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Project

Backend API for "Leo Saloon" built on Next.js 16 (App Router) route handlers, Prisma 7 + PostgreSQL, Zod 4, and `jose` JWTs. The UI (`app/page.tsx`) is still the create-next-app placeholder; the work so far is in `app/api/`.

## Commands

- `npm run dev` / `npm run build` / `npm run start` — no lint or test scripts are configured.
- `npx prisma migrate dev --name <name>` — create/apply a migration after editing `prisma/schema.prisma`.
- `npx prisma generate` — regenerate the client into `app/generated/prisma` (gitignored; must be generated after a fresh clone or schema change, or imports from `@/app/generated/prisma/*` fail).
- `npx prisma db seed` — runs `tsx prisma/seed.ts` (creates the admin user).
- Prisma config lives in `prisma.config.ts` (loads `.env` via `dotenv`), not in the schema's `datasource` block.

Env vars: `DATABASE_URL` (Postgres), `JOSE_SECRET` (JWT signing; falls back to a hardcoded default).

## Architecture

- **Prisma client**: always import the singleton `prisma` from `@/lib/prisma` (uses the `@prisma/adapter-pg` driver adapter, required in Prisma 7). Import generated types/enums from `@/app/generated/prisma/client` and `@/app/generated/prisma/enums` — not from `@prisma/client`.
- **Auth**: `POST /api/auth` verifies the bcrypt password and sets an httpOnly `login-token` cookie holding an HS256 JWT whose payload is `RequestUserType` (`types/requestUser.ts`), including the user's `privileges`. Because privileges are baked into the token, changes to a user's privileges take effect only after re-login.
- **Authorization** (`utils/authentication.ts`): `getUser(request)` decodes the cookie (returns `null` if invalid); `isPrivileged(request, "resource:action")` checks the token's `privileges` array. Privilege strings follow `resource:action` (e.g. `products:read`, `products:add`, `products:edit`, `products:delete`, `users:read`). Authorization is privilege-based, not role-based. **Strict rule:** no matter how a requirement is worded (e.g. "only admins can…"), never gate access on `User.role` — always map it to a `resource:action` privilege checked with `isPrivileged`, and name the new privilege so it can be granted.
- **Route handler conventions** (see `app/api/products/route.ts` as the reference):
  - One `route.ts` per resource exporting `GET`/`POST`/`PUT`/`DELETE`; item IDs come from the `?id=` query param rather than dynamic `[id]` segments (e.g. `app/api/products/byId/route.ts`).
  - Request bodies are validated with Zod schemas in `types/dto/` (each file exports the schema and an inferred type). On `z.ZodError`, respond 400 with `error.issues[0].message`.
  - Responses are `NextResponse.json({ message, ... }, { status })`; 403 for missing privilege.
  - Pagination: `getPaginationInfo(request)` (`utils/pageInfoRetrieval.ts`) reads `pageNumber`/`pageSize` (defaults 1/10); list responses include a `pagination` object `{ pageNumber, pageSize, totalPages, totalCount }`.
- **Products**: deletion is a soft delete (`status = DELETED`); DELETED products are hidden from everyone, INACTIVE products are visible only to holders of `products:read`. Updating `media` replaces all of a product's `Media` rows.
- `User.role` and `User.status` are plain strings (`status`: ACTIVE/BANNED/SUSPENDED; login rejects non-ACTIVE), while `Product.status` and `Media.type` are Prisma enums.
