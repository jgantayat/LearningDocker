# Docker — Day 3: Building Your Own Images & Volumes

> **Goal for today:** Stop using other people's images and start building your own — the *right* way, with a **Dockerfile** (the reproducible alternative to Day 2's opaque `docker commit`). You'll learn every key instruction, master the two concepts that trip everyone up (`CMD` vs `ENTRYPOINT`, `ENV` vs `ARG`), exploit layer caching to make builds fast, cut image size dramatically with **multi-stage builds**, and finally solve the statelessness problem from Day 1 using **volumes** for persistent data.
>
> **Time budget:** ~40 min theory → ~90 min hands-on → ~30 min review and notes. This is the heaviest day — don't rush it.
>
> **Prerequisite:** Days 1–2 complete. You should be comfortable with the layer model and copy-on-write, since a Dockerfile is literally a recipe for producing layers.

---

## Table of Contents

1. [Why Dockerfiles (Recap of `commit`)](#1-why-dockerfiles-recap-of-commit)
2. [Your First Dockerfile](#2-your-first-dockerfile)
3. [The Build Process & Layer Caching](#3-the-build-process--layer-caching)
4. [Dockerfile Instructions — The Full Tour](#4-dockerfile-instructions--the-full-tour)
5. [CMD vs ENTRYPOINT (the big one)](#5-cmd-vs-entrypoint-the-big-one)
6. [ENV vs ARG](#6-env-vs-arg)
7. [.dockerignore](#7-dockerignore)
8. [Multi-Stage Builds](#8-multi-stage-builds)
9. [Persisting Data: Volumes & Mounts](#9-persisting-data-volumes--mounts)
10. [Essential Commands Reference](#10-essential-commands-reference)
11. [Hands-On Labs](#11-hands-on-labs)
12. [Common Mistakes & Gotchas](#12-common-mistakes--gotchas)
13. [Day 3 Checklist](#13-day-3-checklist)

---

## 1. Why Dockerfiles (Recap of `commit`)

On Day 2 you turned a modified container into an image with `docker commit`. It worked — but the resulting image was **opaque**: nothing recorded *how* it was made. You couldn't review it, reproduce it, or audit it.

A **Dockerfile** fixes all of that. It's a plain text file containing the **ordered instructions** to build an image. Because it's just text:

- It lives in version control (Git) alongside your code.
- It's reviewable in a pull request.
- It's **reproducible** — anyone can rebuild the exact same image from it.
- It's self-documenting — the file *is* the record of how the image was made.

> **The contrast to keep in mind:** `commit` = snapshot a container, no record of how. **Dockerfile** = a reviewable, version-controlled recipe that produces the image deterministically. From today on, this is how you build images. `commit` was a learning tool.

Each instruction in a Dockerfile generally produces **one layer** — exactly the layers you inspected with `docker image history` on Day 2. So a Dockerfile is, quite literally, you writing the layer stack by hand.

---

## 2. Your First Dockerfile

Here's a minimal Dockerfile for a Python web app. Read it top to bottom — it reads like a recipe.

```dockerfile
# 1. Start from a base image (this becomes the bottom layers)
FROM python:3.12-slim

# 2. Set the working directory inside the image
WORKDIR /app

# 3. Copy dependency list first (caching trick — explained in §3)
COPY requirements.txt .

# 4. Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copy the rest of the application code
COPY . .

# 6. Document which port the app listens on
EXPOSE 8000

# 7. The command that runs when a container starts
CMD ["python", "app.py"]
```

You build it into an image with:

```bash
docker build -t myapp:1.0 .
```

- `-t myapp:1.0` → **tag** the resulting image with a name and version.
- `.` → the **build context**: the directory Docker sends to the daemon (where it looks for the Dockerfile and the files referenced by `COPY`).

Then run it exactly like any other image:

```bash
docker run -d -p 8000:8000 myapp:1.0
```

---

## 3. The Build Process & Layer Caching

When you run `docker build`, the daemon executes each instruction in order, and **each instruction that changes the filesystem creates a new layer** on top of the previous one — the same read-only layers you met on Day 2.

### The build cache — and why instruction order matters

Before running an instruction, Docker checks: *"Have I built this exact layer before, from the same inputs?"* If yes, it **reuses the cached layer** instead of redoing the work. The moment one instruction's inputs change, that layer **and every layer after it** must be rebuilt — the cache is invalidated from that point down.

This is why the example in §2 copies `requirements.txt` and installs dependencies *before* copying the rest of the code:

```dockerfile
COPY requirements.txt .          # changes rarely
RUN pip install -r requirements.txt   # expensive — we want this cached
COPY . .                         # changes constantly (every code edit)
```

```
   Build cache flow
   ┌──────────────────────────────┐
   │ FROM python:3.12-slim        │  cached (base rarely changes)
   ├──────────────────────────────┤
   │ COPY requirements.txt        │  cached UNLESS requirements change
   ├──────────────────────────────┤
   │ RUN pip install ...          │  cached UNLESS the line above changed
   ├──────────────────────────────┤
   │ COPY . .                     │  ← invalidated on EVERY code edit
   ├──────────────────────────────┤
   │ CMD ...                      │  rebuilt after the line above
   └──────────────────────────────┘
```

If you instead did `COPY . .` *before* installing dependencies, then **every** code change would bust the cache on the `pip install` layer and reinstall everything from scratch — turning a 2-second rebuild into a 2-minute one.

> **The golden rule of Dockerfile ordering:** put the instructions that change *least often* near the top, and the ones that change *most often* (your source code) near the bottom. This keeps the expensive layers cached.

You can force a clean build (ignore the cache) with:

```bash
docker build --no-cache -t myapp:1.0 .
```

---

## 4. Dockerfile Instructions — The Full Tour

Here's every instruction you need today, grouped by purpose.

### `FROM` — the base image

```dockerfile
FROM python:3.12-slim
```
Every Dockerfile starts with `FROM`. It sets the base layers. Choosing a smaller base (`slim`, `alpine`) means a smaller final image. You can have multiple `FROM`s in one file — that's multi-stage builds (§8).

### `WORKDIR` — set the working directory

```dockerfile
WORKDIR /app
```
Sets the directory for subsequent `RUN`, `CMD`, `COPY`, etc. Creates it if it doesn't exist. **Prefer this over `RUN cd /app`** — `cd` in a `RUN` doesn't persist to the next instruction (each `RUN` is a fresh shell), but `WORKDIR` does.

### `RUN` — execute a command at build time

```dockerfile
RUN apt-get update && apt-get install -y curl
```
Runs a command *while building the image* and commits the result as a layer. **Chain related commands with `&&`** to keep them in one layer and avoid stale-cache bugs:

```dockerfile
# Good — one layer, no stale package cache
RUN apt-get update && apt-get install -y \
        curl \
        git \
    && rm -rf /var/lib/apt/lists/*
```

> ⚠️ Splitting `apt-get update` and `apt-get install` into two separate `RUN` lines causes a classic bug: the `update` layer gets cached, and weeks later `install` runs against a stale package index. Always chain them.

### `COPY` — copy files into the image

```dockerfile
COPY requirements.txt .
COPY . .
```
Copies files/directories from the **build context** (your project folder) into the image.

### `ADD` — like COPY, but with extras

```dockerfile
ADD https://example.com/file.tar.gz /tmp/   # can fetch URLs
ADD archive.tar.gz /app/                    # auto-extracts local tarballs
```
`ADD` can fetch remote URLs and auto-extract local tar archives. **Best practice: prefer `COPY`.** Use `ADD` *only* when you specifically need auto-extraction. `COPY` is more predictable and explicit.

> **COPY vs ADD — the rule:** Use `COPY` for everything. Reach for `ADD` only when you need to auto-extract a local tarball. Don't use `ADD` to download URLs — use `RUN curl/wget` so you control caching and can clean up.

### `ENV` — set environment variables (runtime)

```dockerfile
ENV APP_PORT=8000 \
    LOG_LEVEL=info
```
Sets environment variables that exist **both during build and at runtime** inside the container. (Contrast with `ARG` — see §6.)

### `EXPOSE` — document the listening port

```dockerfile
EXPOSE 8000
```
> ⚠️ **`EXPOSE` does NOT publish the port.** It's pure documentation — a hint about which port the app uses. Actual port mapping happens at run time with `docker run -p 8000:8000`. This is a very common misconception.

### `LABEL` — add metadata

```dockerfile
LABEL maintainer="you@example.com" \
      version="1.0" \
      description="My demo app"
```
Key-value metadata attached to the image, visible via `docker inspect`.

### `USER` — drop root privileges

```dockerfile
RUN useradd -m appuser
USER appuser
```
By default a container runs as **root**, which is a security risk. `USER` switches to a non-root user for subsequent instructions and at runtime. You'll make heavy use of this on Day 5 (hardening) — but it's good to start now.

### `HEALTHCHECK` — let Docker check if the app is alive

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```
Tells Docker how to test that the container is actually working (not just running). The status shows up in `docker ps` as `healthy` / `unhealthy`. Invaluable in production and orchestration.

### `ONBUILD` — deferred instruction (rare)

```dockerfile
ONBUILD COPY . /app/src
```
Registers an instruction that runs only when *another* image uses this one as its base. Niche — know it exists, you'll rarely write it.

### `CMD` and `ENTRYPOINT`

These deserve their own section — see §5.

---

## 5. CMD vs ENTRYPOINT (the big one)

This is the single most confusing part of Dockerfiles. Take your time here.

Both define **what runs when the container starts**. The difference is how they interact with each other and with arguments you pass to `docker run`.

### `CMD` — the *default* command, easily overridden

```dockerfile
CMD ["python", "app.py"]
```
`CMD` provides a **default** that is **completely replaced** if you supply a command on the `docker run` line:

```bash
docker run myapp              # runs: python app.py
docker run myapp echo hello   # runs: echo hello  (CMD ignored entirely)
```

### `ENTRYPOINT` — the fixed command, *not* overridden by args

```dockerfile
ENTRYPOINT ["python", "app.py"]
```
`ENTRYPOINT` sets a command that **always runs**. Arguments on the `docker run` line are *appended* to it, not replaced:

```bash
docker run myapp              # runs: python app.py
docker run myapp --debug      # runs: python app.py --debug
```

### Using them *together* (the recommended pattern)

`ENTRYPOINT` sets the fixed executable; `CMD` provides **default arguments** that callers can override:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

```bash
docker run myapp                    # python app.py --port 8000
docker run myapp --port 9000        # python app.py --port 9000  (CMD replaced)
```

> **Mental model:** `ENTRYPOINT` = *what this container fundamentally is* (the program). `CMD` = *its default arguments*, which the user can swap out. Use `ENTRYPOINT` when the container is essentially one specific tool; use `CMD` alone when you want an easily-replaceable default.

### Shell form vs exec form (important!)

There are two ways to write `CMD`/`ENTRYPOINT`/`RUN`:

| Form | Syntax | What happens |
|---|---|---|
| **Exec form** ✅ | `CMD ["python", "app.py"]` | Runs the binary **directly** as PID 1. Signals (like `docker stop`'s SIGTERM) reach your app properly. |
| **Shell form** | `CMD python app.py` | Runs via `/bin/sh -c`, so your app is a *child* of the shell. Signals hit the shell, not your app — graceful shutdown can break. |

> **Always prefer the exec form** (the JSON-array `["..."]` syntax) for `CMD` and `ENTRYPOINT`. The shell form's signal-handling problem causes containers that won't stop cleanly and have to be force-killed.

---

## 6. ENV vs ARG

Both define variables, but they live at different times.

| | `ARG` | `ENV` |
|---|---|---|
| Available during **build** | ✅ Yes | ✅ Yes |
| Available at **runtime** (inside the running container) | ❌ No | ✅ Yes |
| Set from the CLI | `--build-arg NAME=value` | `-e NAME=value` |
| Typical use | build-time choices (versions, flags) | runtime config the app reads |

```dockerfile
ARG PYTHON_VERSION=3.12        # build-time only
FROM python:${PYTHON_VERSION}-slim

ARG BUILD_ENV=production       # available only while building
ENV APP_PORT=8000              # available to the running app too
```

Build-time override:
```bash
docker build --build-arg PYTHON_VERSION=3.11 -t myapp .
```

Runtime override:
```bash
docker run -e APP_PORT=9000 myapp
```

> **The rule:** Use `ARG` for things that only matter *while building* (which base version, a feature flag for the build). Use `ENV` for configuration the *running application* needs to read. **Never put secrets in either** — they end up baked into image layers and history (you'll see the secure way on Day 5).

---

## 7. .dockerignore

The build context (`.` in `docker build`) is sent to the daemon. If your project folder contains `node_modules`, `.git`, large data files, or secrets, all of it gets shipped — slowing builds and risking leaks.

A `.dockerignore` file (same idea as `.gitignore`) excludes paths from the build context:

```
# .dockerignore
.git
node_modules
__pycache__
*.pyc
.env
*.log
.vscode
Dockerfile
README.md
```

Benefits:
- **Faster builds** (less data sent to the daemon).
- **Smaller images** (you won't accidentally `COPY . .` junk in).
- **Security** — keeps `.env` files and secrets out of the image.

> Always add a `.dockerignore`. At minimum exclude `.git`, dependency folders (`node_modules`, virtualenvs), caches (`__pycache__`), and anything secret (`.env`).

---

## 8. Multi-Stage Builds

This is the technique that makes images dramatically smaller and is considered industry standard.

### The problem

To *build* an app you often need heavy tools — compilers, dev dependencies, build toolchains. But to *run* it you only need the final artifact (a binary, a bundle, the installed packages). If you build and run in one image, all that build tooling ships to production as dead weight — and as extra attack surface (Day 5).

### The solution

Use **multiple `FROM` stages** in one Dockerfile. Earlier stages do the heavy building; the final stage copies *only the finished artifact* out of them, starting from a clean, minimal base.

```dockerfile
# ---------- Stage 1: build ----------
FROM node:20 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # installs ALL deps, incl. dev/build tools
COPY . .
RUN npm run build             # produces /app/dist

# ---------- Stage 2: runtime ----------
FROM node:20-alpine           # tiny base
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev         # only production deps
COPY --from=builder /app/dist ./dist   # copy ONLY the build output
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

The magic line is `COPY --from=builder /app/dist ./dist` — it reaches into the first stage and pulls out only what's needed. The build tools, dev dependencies, and source-only files in `builder` are **discarded** — they never reach the final image.

```
   Stage 1 (builder)            Stage 2 (final, shipped)
   ┌─────────────────┐          ┌─────────────────┐
   │ node:20 (~1 GB) │          │ node:20-alpine  │
   │ + all dev deps  │  COPY    │ + prod deps     │
   │ + source        │ ──only──▶│ + dist/ only    │
   │ + dist/ output  │  dist/   │                 │
   └─────────────────┘          └─────────────────┘
     thrown away                  ~150–200 MB
```

Typical result: an image that was **1.2 GB drops to ~180 MB.** Smaller images push/pull faster, start faster, and have far less to attack.

Build a specific stage (e.g. for debugging the builder) with:
```bash
docker build --target builder -t myapp:builder .
```

> **Why this matters for Day 5:** every tool *not* in your final image is one fewer thing an attacker can exploit. Multi-stage builds are step one of image hardening. We'll go further (distroless) on Friday.

---

## 9. Persisting Data: Volumes & Mounts

Now we solve the problem you discovered on Day 1: **containers are ephemeral.** Anything written to a container's writable layer dies when the container is removed (that's copy-on-write, from Day 2). For databases, uploads, and anything that must survive, you need storage that lives **outside** the container's writable layer.

Docker offers three mount types:

```
   ┌──────────────────────────────────────────────────────┐
   │                      Container                         │
   │   /var/lib/postgresql/data  ──┐                        │
   │   /app (live code)          ──┼──┐                     │
   │   /tmp/cache                ──┼──┼──┐                  │
   └───────────────────────────────┼──┼──┼─────────────────┘
                                    │  │  │
                ┌───────────────────┘  │  └──────────────┐
                ▼                       ▼                 ▼
        ┌──────────────┐      ┌──────────────────┐  ┌───────────┐
        │ Named Volume │      │   Bind Mount      │  │  tmpfs    │
        │ (Docker-     │      │ (a host folder    │  │ (in RAM,  │
        │  managed)    │      │  you choose)      │  │  ephemeral)│
        └──────────────┘      └──────────────────┘  └───────────┘
        persistent,           your laptop's files,   fast, vanishes
        portable, prod        great for dev          on stop
```

### 1. Named volumes — the production default

Docker creates and manages a storage area; you refer to it by name. It persists independently of any container.

```bash
docker volume create pgdata
docker run -d --name db \
    -v pgdata:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:16
```

Stop and remove the container, recreate it pointing at the same `pgdata` volume, and **your data is still there.** This is how you run databases in containers.

### 2. Bind mounts — map a host folder (great for development)

Mounts a specific folder from *your machine* into the container. Changes on either side are reflected immediately — perfect for live-reloading code during development.

```bash
docker run -d --name dev \
    -v "$PWD":/app \
    -p 8000:8000 \
    myapp:1.0
```

Now editing a file on your laptop instantly changes it inside the container. (Use this for dev only; in production you bake code into the image.)

### 3. tmpfs — in-memory, ephemeral

Stores data in the host's RAM. Fast, and gone when the container stops. Good for sensitive temp data you *don't* want written to disk.

```bash
docker run -d --tmpfs /tmp/cache myapp:1.0
```

### Named volume vs bind mount — when to use which

| | Named volume | Bind mount |
|---|---|---|
| Managed by | Docker | You (it's your host path) |
| Location | Docker's storage area | Wherever you point it |
| Best for | **Production** data (DBs, uploads) | **Development** (live code editing) |
| Portability | High — not tied to host paths | Low — depends on host layout |

> **Rule of thumb:** named volumes for persistent app data in production; bind mounts for editing code live in development; tmpfs for sensitive, throwaway data.

### Managing volumes

```bash
docker volume ls               # list volumes
docker volume inspect pgdata   # where it lives, when created
docker volume rm pgdata        # delete a volume (data gone!)
docker volume prune            # remove all unused volumes (careful!)
```

> ⚠️ Removing a container does **not** remove its named volumes — that's the whole point (data survives). But it means volumes can quietly accumulate; `docker volume ls` and `docker volume prune` keep things tidy. Conversely, `docker volume rm` **permanently deletes data** — there's no undo.

---

## 10. Essential Commands Reference

### Building images

```bash
docker build -t myapp:1.0 .              # build & tag from ./Dockerfile
docker build -t myapp:1.0 -f Custom.Dockerfile .   # custom filename
docker build --no-cache -t myapp:1.0 .   # ignore the build cache
docker build --target builder -t x .     # build a specific stage
docker build --build-arg VER=3.11 -t x . # pass a build-time ARG
docker build --progress=plain .          # verbose build output (debugging)
```

### Volumes

```bash
docker volume create <name>
docker volume ls
docker volume inspect <name>
docker volume rm <name>
docker volume prune
```

### Running with mounts

```bash
docker run -v myvol:/data myapp            # named volume
docker run -v "$PWD":/app myapp            # bind mount (current dir)
docker run -v "$PWD"/conf:/etc/app:ro myapp # read-only bind mount
docker run --tmpfs /tmp myapp              # tmpfs mount
docker run --mount type=volume,src=myvol,dst=/data myapp  # explicit syntax
```

### Inspecting your build

```bash
docker image history myapp:1.0   # see the layers you created (from Day 2)
docker image inspect myapp:1.0
docker images                    # compare sizes (esp. before/after multi-stage)
```

---

## 11. Hands-On Labs

> **Do these in order, type don't paste, predict outputs first.** Today's labs take you from a hand-written Dockerfile to a slim multi-stage image with persistent data and a healthcheck.

### Lab 1 — Write and build your first Dockerfile

Create a project folder and three files.

`app.py`:
```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from my Docker image!\n")

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

`requirements.txt` (empty is fine — stdlib only, but we'll keep the line for the caching lesson):
```
# no external deps yet
```

`Dockerfile`:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["python", "app.py"]
```

Build and run:
```bash
docker build -t myapp:1.0 .
docker run -d --name web -p 8000:8000 myapp:1.0
curl http://localhost:8000        # or open in a browser
```

✅ **Expected:** `Hello from my Docker image!` You built an image from scratch.

---

### Lab 2 — Watch layer caching in action

Edit `app.py` (change the message text), then rebuild:

```bash
docker build -t myapp:1.1 .
```

Watch the output: the `FROM`, `COPY requirements.txt`, and `RUN pip install` steps say **`CACHED`** — only the `COPY . .` layer and below rebuild, because only your code changed. Now do the opposite: edit `requirements.txt` and rebuild — notice the `pip install` layer (and everything after) rebuilds.

✅ **What you learned:** instruction order controls what gets cached. Code-last ordering keeps expensive layers cached. (This is the Day 2 layer model working for you.)

---

### Lab 3 — CMD vs ENTRYPOINT experiment

Build a tiny image to feel the difference. `Dockerfile.cmd`:
```dockerfile
FROM ubuntu
CMD ["echo", "default message"]
```
```bash
docker build -f Dockerfile.cmd -t demo:cmd .
docker run --rm demo:cmd                 # → default message
docker run --rm demo:cmd echo overridden # → overridden  (CMD replaced)
```

Now `Dockerfile.entry`:
```dockerfile
FROM ubuntu
ENTRYPOINT ["echo", "always:"]
CMD ["default arg"]
```
```bash
docker build -f Dockerfile.entry -t demo:entry .
docker run --rm demo:entry               # → always: default arg
docker run --rm demo:entry custom arg    # → always: custom arg  (CMD replaced, ENTRYPOINT stays)
```

✅ **What you learned:** `CMD` is fully overridden by run args; `ENTRYPOINT` stays fixed and args append to it.

---

### Lab 4 — Multi-stage build, compare sizes

We'll use a compiled-language style to make the size difference dramatic. Create `hello.c`:
```c
#include <stdio.h>
int main() { printf("Hello from a tiny image!\n"); return 0; }
```

Single-stage `Dockerfile.fat`:
```dockerfile
FROM gcc:13
WORKDIR /src
COPY hello.c .
RUN gcc -o hello hello.c
CMD ["./hello"]
```

Multi-stage `Dockerfile.slim`:
```dockerfile
FROM gcc:13 AS builder
WORKDIR /src
COPY hello.c .
RUN gcc -static -o hello hello.c

FROM alpine:3.20
COPY --from=builder /src/hello /hello
CMD ["/hello"]
```

Build both and compare:
```bash
docker build -f Dockerfile.fat  -t hello:fat  .
docker build -f Dockerfile.slim -t hello:slim .
docker images | grep hello
```

✅ **Expected:** `hello:fat` is well over 1 GB (it carries the whole GCC toolchain); `hello:slim` is a few MB. Run both — they print the same thing. **Same output, a fraction of the size and attack surface.**

---

### Lab 5 — Persistent data with a named volume

Prove data survives container deletion. Run Postgres with a named volume:

```bash
docker volume create pgdata
docker run -d --name db \
    -e POSTGRES_PASSWORD=secret \
    -v pgdata:/var/lib/postgresql/data \
    postgres:16
```

Create some data:
```bash
docker exec -it db psql -U postgres -c "CREATE TABLE notes(msg TEXT);"
docker exec -it db psql -U postgres -c "INSERT INTO notes VALUES ('survives!');"
```

Now **destroy the container** and recreate it pointing at the same volume:
```bash
docker rm -f db
docker run -d --name db \
    -e POSTGRES_PASSWORD=secret \
    -v pgdata:/var/lib/postgresql/data \
    postgres:16
docker exec -it db psql -U postgres -c "SELECT * FROM notes;"
```

✅ **Expected:** the row `survives!` is still there. The container was destroyed; the data wasn't — because it lived in the volume, not the container's writable layer. **This is the answer to Day 1's statelessness.**

---

### Lab 6 — Bind mount for live development

Go back to your Lab 1 app folder and run with a bind mount:

```bash
docker run -d --name dev -p 8001:8000 -v "$PWD":/app myapp:1.0
curl http://localhost:8001
```

Edit `app.py` on your host (change the message), then restart the process inside the container (or re-run). The change is visible without rebuilding the image, because the host folder is mounted live.

✅ **What you learned:** bind mounts map host files into the container — ideal for editing code during development.

---

### Lab 7 — Add a HEALTHCHECK

Add this to your Lab 1 `Dockerfile` (above the `CMD`):

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000')" || exit 1
```

Rebuild and run, then watch the health status appear:
```bash
docker build -t myapp:health .
docker run -d --name hc -p 8002:8000 myapp:health
docker ps        # STATUS column shows "health: starting" → "healthy"
```

✅ **What you learned:** Docker can verify your app actually works, not just that the process is up.

---

### Lab 8 — Clean up

```bash
docker rm -f web dev hc db 2>/dev/null; true
docker volume rm pgdata
docker image prune -f
docker images | grep -E "myapp|hello|demo"   # review what remains
```

---

## 12. Common Mistakes & Gotchas

- **Expecting `EXPOSE` to publish a port.** It only documents. You still need `-p` at run time.
- **Putting `COPY . .` before dependency installation.** Busts the cache on every code change; install deps first.
- **Separate `apt-get update` and `apt-get install` lines.** Causes stale package indexes; chain them with `&&`.
- **Using shell form for CMD/ENTRYPOINT.** Breaks signal handling and clean shutdown; use exec form `["..."]`.
- **Confusing CMD and ENTRYPOINT.** CMD = overridable default; ENTRYPOINT = fixed command, args append.
- **Confusing ENV and ARG.** ARG is build-time only; ENV persists to runtime. Neither is for secrets.
- **No `.dockerignore`.** Ships `.git`, `node_modules`, and possibly `.env` into the build — slow and unsafe.
- **Forgetting that removing a container keeps its named volume.** Good for data safety, but volumes accumulate — prune occasionally. And `docker volume rm` deletes data permanently.
- **Bind-mounting over a folder the image populated.** The host folder *replaces* the container's version at that path — if your host folder is empty, you'll hide the image's files there.
- **Building huge images by not using multi-stage.** Ship only the artifact, not the whole toolchain.

---

## 13. Day 3 Checklist

Tick these before moving to Day 4. Redo the matching lab for anything shaky.

- [ ] I can explain why a **Dockerfile** is better than `docker commit` (reproducible, reviewable, versioned).
- [ ] I understand that each instruction generally creates a **layer**, and I can order instructions to maximize cache hits.
- [ ] I know what `FROM`, `WORKDIR`, `RUN`, `COPY`, `ENV`, `EXPOSE`, `LABEL`, `USER`, `HEALTHCHECK` each do.
- [ ] I can explain **COPY vs ADD** and when (rarely) to use ADD.
- [ ] I can clearly explain **CMD vs ENTRYPOINT**, including using them together.
- [ ] I know **exec form vs shell form** and why exec form is preferred.
- [ ] I can explain **ENV vs ARG** and that neither should hold secrets.
- [ ] I created and used a **.dockerignore**.
- [ ] I built a **multi-stage** image and saw the size drop dramatically vs single-stage.
- [ ] I understand the three mount types: **named volume, bind mount, tmpfs**, and when to use each.
- [ ] I proved data **survives container deletion** using a named volume (the answer to Day 1 statelessness).
- [ ] I used a **bind mount** to edit code live.
- [ ] I added a **HEALTHCHECK** and saw the health status in `docker ps`.
- [ ] I can manage volumes: `create`, `ls`, `inspect`, `rm`, `prune`.

---

### What's next — Day 4 preview

Tomorrow you connect containers together: **Docker networking deep dive.** You'll learn the network drivers (**bridge, host, none, overlay, macvlan**), why you should always create **user-defined bridge networks** (automatic DNS by container name), how port mapping actually works under the hood (iptables/NAT), and then tie the whole week together with **Docker Compose** — defining a multi-container app (web + database + cache) in a single file, where the database volume you learned today and service-to-service networking come together. Today's Postgres-with-a-volume lab is the exact piece Compose will orchestrate.

> **Tip:** In your `docker-notes.md`, write — in your own words — the difference between `CMD` and `ENTRYPOINT`, and between a named volume and a bind mount. Also jot the before/after image sizes from Lab 4; that number is the most persuasive argument for multi-stage builds you'll ever have.