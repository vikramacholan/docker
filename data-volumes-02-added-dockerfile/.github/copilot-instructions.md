# Data Volumes (feedback) — Copilot instructions

This document captures the essential, discoverable knowledge an AI coding agent needs to work productively on the
`data-volumes-02-added-dockerfile` sample app. Keep it focused and actionable — reference real file names and concrete
commands.

## Big-picture
- Purpose: a small Express.js app that saves user-submitted text files into `./feedback/` and serves them from that
  directory. It demonstrates Docker volumes and bind mounts.
- Runtime: Node.js (Dockerfile uses `node:14`) and an app listening on port `80` inside the container.

## Key files
- `Dockerfile` — standard layered Node build: `COPY package.json` → `RUN npm install` → `COPY .` → `CMD npm start`.
- `server.js` — core logic: serves `pages/feedback.html`, `pages/exists.html`, serves static `public/` and `feedback/`.
  - POST `/create` writes to `temp/<title>.txt` then moves the file into `feedback/<title>.txt`.
- `pages/feedback.html` — the HTML form that posts `title` and `text` to `/create`.
- `feedback/` — where saved documents live (mounted as a Docker volume in many run commands).

## Important behaviors & gotchas (discoverable)
- Cross-filesystem rename (EXDEV): `server.js` originally used `fs.rename(temp, final)`. When `feedback/` is a Docker *named
  volume* and the rest of the app is a host bind-mount, rename can fail with `EXDEV: cross-device link not permitted`.
  - Solution in repo: server now copies (`fs.copyFile`) then unlinks the temp file; this avoids EXDEV.
  - Alternative approaches: write directly into the target filesystem, ensure both paths are on the same mount, or use
    host bind-mounts consistently.
- `fs.exists` usage: repository uses the callback `exists(...)`. Prefer `fs.access`/`fs.stat` or use promisified checks to
  avoid subtle deprecation/edge cases.
- In-memory state / persistence: files in `feedback/` are persisted via Docker volumes; if you bind-mount the project
  folder, files are persisted on the host.

## Build & run (exact commands used during debugging)
Run from project root (`data-volumes-02-added-dockerfile`):

```bash
# build
docker build -t feedback:2.0 .

# run (example used while diagnosing):
docker run -d --rm -p 3000:80 --name feedback-app \
  -v feedback:/app/feedback \
  -v "$(pwd)":/app \
  -v /app/node_modules \
  feedback:2.0
```

Notes:
- `-v feedback:/app/feedback` creates/uses a named volume for `feedback/` (host-managed). Combining this with a host
  bind for `/app` can put `temp/` and `feedback/` on different filesystems, triggering EXDEV on `rename`.
- `-v /app/node_modules` is used to prevent the host's `node_modules` from shadowing the container's installed
  dependencies when bind-mounting the project directory.

Safer alternatives:
- Avoid mixing named volumes and a host bind of `/app`. Either:
  - Use only a host bind for `feedback` (e.g. `-v "$(pwd)/feedback":/app/feedback`) or
  - Use only a named volume for the full app (do not bind-mount the project into `/app`).

## Debugging quick-hits
- View logs:
  - `docker logs -f feedback-app`
- Inspect mount points and container config:
  - `docker inspect feedback-app --format '{{json .Mounts}}' | jq` (or plain `docker inspect feedback-app`)
- Rebuild after code changes:
  - `docker build -t feedback:2.0 . && docker restart feedback-app` (or stop & run again)

## Suggested micro-improvements (focused, low-risk)
- Sanitize `title` before using it as a filename (strip `../`, slashes, non-printable chars).
- Replace `fs.exists` callback style with `fs.access`/`fs.stat` and promises.
- Add explicit error pages / JSON error responses rather than letting Node unhandled rejections crash the app.

## Example code note (what an agent may change)
- If implementing an EXDEV-safe move, prefer this pattern already applied in `server.js`:
  - `await fs.copyFile(tempPath, finalPath); await fs.unlink(tempPath);`

---
If anything here is unclear or you'd like me to add a small test or filename sanitization patch, tell me which item to
prioritize and I'll implement it next.

## Useful run & cleanup commands

Run with a host bind for only the `feedback` directory (recommended to avoid cross-device issues):

```bash
docker run -d --rm -p 3000:80 --name feedback-app \
  -v "$(pwd)/feedback":/app/feedback \
  -v /app/node_modules \
  feedback:2.0
```

Run with a host bind for the whole project (convenient for editing, but can mix filesystems):

```bash
docker run -d --rm -p 3000:80 --name feedback-app \
  -v "$(pwd)":/app \
  -v /app/node_modules \
  feedback:2.0
```

Stop / remove containers (use with care):

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers (stopped or running after stopping)
docker rm $(docker ps -aq)

# One-liner: stop then remove all containers
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
```
