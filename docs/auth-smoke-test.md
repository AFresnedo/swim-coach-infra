# Auth Smoke Test

End-to-end test of the register → logout → sign-in flow. Requires both servers running.

## Prerequisites

Start the FastAPI backend (from `back-end/`):

```bash
source $HOME/.local/bin/env
cd /Users/andres/Workspace/SwimCoach/back-end
uv run uvicorn app.main:app --reload
```

Start the Next.js frontend (from `swim-coach/`):

```bash
cd /Users/andres/Workspace/SwimCoach/swim-coach
npm run dev
```

Confirm both are up:

```bash
curl http://localhost:8000/health   # {"status":"ok"}
curl http://localhost:3000          # HTML response
```

---

## 1. Register

```bash
curl -s -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Swimmer","email":"test@swimcoach.com","password":"password123"}' \
  -c /tmp/cookies.txt -v
```

**Expected response:**
```
HTTP/1.1 200 OK
set-cookie: access_token=<jwt>; Path=/; HttpOnly; SameSite=strict

{"ok":true}
```

The JWT is stored as an httpOnly cookie. It is never exposed in the response body.

---

## 2. Logout

```bash
curl -s -X POST http://localhost:3000/api/auth/logout \
  -b /tmp/cookies.txt -c /tmp/cookies.txt -v
```

**Expected response:**
```
HTTP/1.1 200 OK

{"ok":true}
```

The `access_token` cookie is deleted server-side.

---

## 3. Sign In

```bash
curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@swimcoach.com","password":"password123"}' \
  -c /tmp/cookies2.txt -v
```

**Expected response:**
```
HTTP/1.1 200 OK
set-cookie: access_token=<jwt>; Path=/; HttpOnly; SameSite=strict

{"ok":true}
```

A fresh httpOnly cookie is issued.

---

## Notes

- All auth requests go to Next.js (`localhost:3000`), which proxies them to FastAPI (`localhost:8000`) server-to-server. The browser never calls FastAPI directly.
- The `-c` flag saves cookies to a file; `-b` sends them on the next request. This mimics browser cookie behavior in curl.
- To test a wrong password, change the password in the sign-in payload — expect `401 Unauthorized` with `{"detail":"Incorrect email or password"}`.
