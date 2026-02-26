# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

Enterprise Chat (企业通讯) — a Next.js 16 + React 19 real-time chat application with PostgreSQL, Redis, and Socket.IO. Single repo, two runtime processes: Next.js web app (port 3000) and WebSocket server (port 3001).

### Infrastructure dependencies

PostgreSQL 16 and Redis 7 must be running before starting the app. In the Cloud VM these run as Docker containers:

```bash
sudo dockerd &>/tmp/dockerd.log &
sleep 3
sudo docker start postgres redis 2>/dev/null || {
  sudo docker run -d --name postgres -p 5432:5432 -e POSTGRES_DB=chat_app -e POSTGRES_USER=chat_user -e POSTGRES_PASSWORD=chat_pass postgres:16-alpine
  sudo docker run -d --name redis -p 6379:6379 redis:7-alpine
}
```

### Environment variables

A `.env` file must exist in the project root. Copy from `.env.example` or create with at minimum:

```
DATABASE_URL=postgresql://chat_user:chat_pass@localhost:5432/chat_app
JWT_SECRET=dev-jwt-secret-for-local-development-only
REDIS_URL=redis://localhost:6379
COOKIE_SECURE=false
INTERNAL_API_SECRET=dev-internal-secret-for-local-development-only
APP_URL=http://localhost:3000
SOCKET_PORT=3001
```

### Running services

| Service | Command | Port |
|---------|---------|------|
| Next.js (dev) | `npm run dev` | 3000 |
| WebSocket server | `npm run start:ws` (needs `REDIS_URL`, `JWT_SECRET`, `DATABASE_URL` in env) | 3001 |

After starting both, verify with: `curl http://localhost:3000/api/health` — should return `{"ok":true,"db":"ok","redis":"ok"}`.

### Database migrations

Run `npx prisma migrate deploy` after first setup or after pulling new migrations.

### Lint / Build / Test

- **Lint**: `npm run lint` — note: `eslint` is not listed in `package.json` dependencies (pre-existing issue), so this command fails with `eslint: not found`.
- **Build**: `npm run build` — runs `prisma generate && next build`. Succeeds.
- **Tests**: No automated test framework is configured in this project.

### Known issue: Middleware JWT verification in Next.js 16

The `middleware.ts` uses `jsonwebtoken` (a Node.js-only library) to verify JWT tokens. Next.js 16 runs middleware in Edge Runtime, which does not support Node.js `crypto` module. This causes `jwt.verify()` to fail silently, preventing authenticated access to protected routes after login. The login API itself works correctly and returns a valid token cookie, but the middleware cannot verify it. This is a pre-existing codebase issue — not an environment setup problem.
