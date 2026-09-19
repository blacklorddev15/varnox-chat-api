# Varnox Chat API

The backend for Varnox Chat: tRPC over Express, plus a Server-Sent Events channel that pushes
"something changed in this conversation" nudges to signed-in clients.

Split out from the web app so it can run as a long-lived process. That matters: Web Push and
Server-Sent Events both need a server that stays alive, which serverless functions are not.

```text
server/           the API (Express, tRPC routers, Drizzle data access, SSE bus)
shared/           constants and types shared with the web app
drizzle/          schema + migrations
tests/            server-side tests
render.yaml       Render blueprint
```

---

## Deploy on Render

1. **New → Blueprint**, pick this repo. Render reads `render.yaml` and creates the service.
2. Fill in the four secret values it prompts for:

   | Variable | Value |
   |---|---|
   | `DATABASE_URL` | your Postgres connection string (`?sslmode=require`) |
   | `JWT_SECRET` | a long random string — changing it signs everyone out |
   | `RESEND_API_KEY` | Resend key, for verification and reset emails |
   | `RESEND_FROM` | e.g. `Varnox Chat <no-reply@your-domain.com>` |

3. Deploy. Migrations run automatically as a pre-deploy step.

Verify:

```bash
curl https://your-service.onrender.com/api/health
# {"ok":true,"timestamp":...,"realtime":true}
```

`realtime: true` means this process can hold an event stream.

### Add a domain before pointing the web app at it

Render gives you `*.onrender.com`, which is a *different registrable domain* from your frontend.
That combination breaks the event stream in a quiet way:

- Ordinary API calls still work — the client sends a `Bearer` token from `localStorage`, which the
  server prefers over the cookie.
- `/api/realtime` returns **401**. `EventSource` cannot set an `Authorization` header, and the
  session cookie is `SameSite=Lax`, so it is not sent across sites. The client backs off and falls
  back to polling — you keep the split and lose the realtime.

So add a custom domain under the same registrable domain as the frontend:

```text
varnoxchat.blacklord.tech   ->  Vercel        (frontend)
api.blacklord.tech          ->  Render        (this service)
```

Render → your service → Settings → Custom Domains → add `api.blacklord.tech`, then create the DNS
record Render shows you. TLS is issued automatically.

### Do not use the free plan

Render's free web services sleep after roughly 15 minutes of inactivity. Every open event stream
dies with them, and the first request after a sleep waits for a cold start. `starter` avoids both.

### Migrations

`pnpm db:migrate` runs on every deploy via `preDeployCommand`. To run one by hand:

```bash
DATABASE_URL="postgresql://..." pnpm db:migrate
```

`pnpm db:generate` creates a new migration after a schema change.

---

## Local development

```bash
cp .env.example .env      # then fill it in
pnpm install
pnpm dev                  # tsx watch, restarts on change
```

```bash
pnpm typecheck
pnpm test
pnpm build && pnpm start  # what Render runs
```

---

## Endpoints

| Route | Purpose |
|---|---|
| `/api/health` | liveness, plus whether this deployment supports realtime |
| `/api/trpc/*` | the tRPC API |
| `/api/realtime` | the SSE stream (auth required; 503 on serverless runtimes) |
| `/api/auth/*` | password, phone and OAuth sign-in |
| `/api/media/*`, `/api/avatar/*` | uploads and avatars |

## Notes

- **Keep this in step with the web app.** `server/`, `shared/` and `drizzle/` are the same code that
  lives in the web repository. Changes made there do not arrive here automatically, and the two
  will drift the moment a router gains a procedure. Either mirror server changes deliberately, or
  point Render at the web repository with a root directory and skip this split entirely.
- **The SSE bus is in-process.** Run exactly one instance; a second would deliver events to nobody.
  Scaling out means moving `server/realtime.ts` to Redis pub/sub or Postgres `LISTEN`/`NOTIFY` — the
  interface is two functions, so that change is contained to one file.
- **Behind a proxy, turn buffering off.** The API sends `X-Accel-Buffering: no`; Render's own proxy
  handles this, but a proxy you add yourself needs `proxy_buffering off;`.
