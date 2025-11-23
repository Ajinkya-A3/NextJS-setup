# 🚀 Next.js Production Environment Configuration Guide

## 📘 Table of Contents  
- [1. Environment Types in Next.js](#1-environment-types-in-nextjs)  
- [2. Server-side Environment Variables (`processenv`)](#2-server-side-environment-variables-processenv)  
- [3. Public Build-Time Env (`next_public`)](#3-public-build-time-env-next_public)  
- [4. Runtime Browser Env (`window_env_`)](#4-runtime-browser-env-window_env_)  
- [5. Calling Server APIs From Client Safely](#5-calling-server-apis-from-client-safely)  
- [6. Production Folder Structure](#6-production-folder-structure)  
- [7. Dockerfile Setup](#7-dockerfile-setup)  
- [8. Docker Compose Environment Setup](#8-docker-compose-environment-setup)  
- [9. Nginx + Certbot (Production HTTPS)](#9-nginx--certbot-production-https)  
- [10. Common Production Issues](#10-common-production-issues)  
- [11. Best Practices Summary](#11-best-practices-summary)  

---

# 1. Environment Types in Next.js

| Type | Usage | Runtime? | Safe for secrets? | Client-accessible? |
|------|--------|----------|--------------------|---------------------|
| `process.env.*` | Server only | ✔ Yes | ✔ Yes | ❌ No |
| `NEXT_PUBLIC_*` | Client build-time | ❌ No | ❌ No | ✔ Yes |
| `window._env_` | Client runtime | ✔ Yes | ❌ No | ✔ Yes |

---

# 2. Server-side Environment Variables (`processenv`)

### ✔ For secrets  
### ✔ Safe  
### ✔ Reconfigurable at runtime via Docker / K8s / Compose  

```js
export async function GET() {
  const api = process.env.INTERNAL_API_URL;
  const key = process.env.INTERNAL_API_KEY;

  const res = await fetch(`${api}/user/info`, {
    headers: { Authorization: `Bearer ${key}` }
  });

  const user = await res.json();

  return Response.json({
    username: user.username,
    plan: user.plan
  });
}
```

---

# 3. Public Build-Time Env (`next_public`)

```jsx
"use client";

export default function Page() {
  return <p>Environment: {process.env.NEXT_PUBLIC_ENV}</p>;
}
```

---

# 4. Runtime Browser Env (`window_env_`)

## Step 1 — `/public/env.js`

```js
window._env_ = {
  PUBLIC_API_URL: "default",
  APP_ENV: "default",
};
```

## Step 2 — Load in layout

```jsx
<html>
  <head>
    <script src="/env.js"></script>
  </head>
  <body>{children}</body>
</html>
```

## Step 3 — Use in client

```jsx
"use client";
const api = window._env_.PUBLIC_API_URL;
```

## Step 4 — Overwrite in Docker

```sh
cat <<EOF > /app/public/env.js
window._env_ = {
  PUBLIC_API_URL: "${PUBLIC_API_URL}",
  APP_ENV: "${APP_ENV}"
};
EOF
```

---

# 5. Calling Server APIs From Client Safely

## Client

```jsx
"use client";

export default function Page() {
  async function load() {
    const res = await fetch("/api/user");
    const data = await res.json();
    console.log(data);
  }

  return <button onClick={load}>Load User</button>;
}
```

## Server

```js
export async function GET() {
  const url = process.env.INTERNAL_API_URL;
  const token = process.env.INTERNAL_API_KEY;

  const res = await fetch(`${url}/user`, {
    headers: { Authorization: `Bearer ${token}` }
  });

  const raw = await res.json();

  return Response.json({
    name: raw.name,
    plan: raw.plan,
  });
}
```

---

# 6. Production Folder Structure

```
/app
  /api
  /components
  layout.js
/public
  env.js
Dockerfile
entrypoint.sh
docker-compose.yml
nginx.conf
```

---

# 7. Dockerfile Setup

```Dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:18 AS runner
WORKDIR /app
COPY --from=builder /app ./
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["npm", "start"]
```

---

# 8. Docker Compose Environment Setup

```yaml
services:
  next:
    image: next-app:latest
    environment:
      INTERNAL_API_URL: "http://internal-api:8080"
      INTERNAL_API_KEY: "uat-secret"
      PUBLIC_API_URL: "https://uat.example.com/api"
      APP_ENV: "uat"
    ports:
      - "3000:3000"
```

---

# 9. Nginx + Certbot (Production HTTPS)

```nginx
server {
    listen 80;
    server_name app.example.com;

    location /.well-known/acme-challenge/ { root /var/www/html; }
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    location / {
        proxy_pass http://nextjs:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

# 10. Common Production Issues

| Issue | Fix |
|-------|------|
| Env not updating | Restart container |
| OAuth failing | Use `window._env_` for callback |
| Mixed HTTP/HTTPS | Add forwarded headers |
| Secret leaked | Avoid NEXT_PUBLIC for private data |
| API failing | Fix backend CORS config |

---

# 11. Best Practices Summary

✔ Use `process.env` for secrets  
✔ Use `/api` routes as server proxy  
✔ Use `window._env_` for runtime client config  
✔ Use `NEXT_PUBLIC_*` for public build-time config  
✔ Build once → deploy everywhere  
✔ Use Nginx + Certbot for TLS  
✔ Validate env before boot  
