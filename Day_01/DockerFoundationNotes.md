# Docker — Day 1: Foundations & Installation

> **Goal for today:** Understand what Docker actually is, why it exists, install it cleanly, and run your first containers. By the end you should be able to pull an image, run a container, get a shell inside it, map ports, and clean up — all from memory.
>
> **Time budget:** ~30 min theory → ~90 min hands-on → ~30 min review and notes.

---

## Table of Contents

1. [The Problem Docker Solves](#1-the-problem-docker-solves)
2. [Virtual Machines vs Containers](#2-virtual-machines-vs-containers)
3. [How Containers Actually Work (Linux internals)](#3-how-containers-actually-work-linux-internals)
4. [Docker Architecture](#4-docker-architecture)
5. [Key Vocabulary](#5-key-vocabulary)
6. [Installation](#6-installation)
7. [Verifying Your Install](#7-verifying-your-install)
8. [The Container Lifecycle](#8-the-container-lifecycle)
9. [Essential Commands Reference](#9-essential-commands-reference)
10. [Hands-On Labs](#10-hands-on-labs)
11. [Common Mistakes & Gotchas](#11-common-mistakes--gotchas)
12. [Day 1 Checklist](#12-day-1-checklist)

---

## 1. The Problem Docker Solves

You've probably heard the phrase **"but it works on my machine."** That sentence is the entire reason Docker exists.

Software doesn't run in a vacuum. Any application depends on:

- A specific language runtime (Python 3.12, Node 20, Java 17…)
- System libraries (a certain version of `glibc`, `openssl`, `libpq`…)
- Environment variables, config files, file paths
- Other services (a database, a cache, a message queue)

When you build something on your laptop and hand it to a teammate — or deploy it to a server — any one of those dependencies being slightly different can break everything. Traditionally teams wrote long setup documents, used config-management tools, or kept fragile "golden" server images.

**Docker's answer:** package the application *together with its entire environment* into a single, portable unit called an **image**. That image runs identically on your laptop, your teammate's laptop, a test server, and production — because it carries its dependencies with it. The running instance of that image is called a **container**.

Think of it like shipping containers in global trade. Before standardized steel containers, loading a ship was chaos — every item had a different shape. Once everything went into identical containers, any crane, truck, or ship could handle any container without caring what was inside. Docker did the same thing for software.

---

## 2. Virtual Machines vs Containers

This is the single most important concept for Day 1. People constantly assume a container is "a lightweight VM." It is **not**. The difference is fundamental.

### Virtual Machines

A VM virtualizes **hardware**. A piece of software called a **hypervisor** (VMware, VirtualBox, Hyper-V, KVM) creates fake virtual hardware, and on top of that you install a **complete guest operating system** — its own kernel, its own drivers, the whole thing.

```
┌─────────────────────────────────────────────┐
│  App A   │   App B   │   App C                │
├──────────┼───────────┼────────────────────────┤
│ Bins/Lib │ Bins/Lib  │ Bins/Lib               │
├──────────┼───────────┼────────────────────────┤
│ Guest OS │ Guest OS  │ Guest OS  ← full kernel │
│ (Linux)  │ (Linux)   │ (Windows) each!         │
├──────────┴───────────┴────────────────────────┤
│              Hypervisor                        │
├────────────────────────────────────────────────┤
│              Host Operating System             │
├────────────────────────────────────────────────┤
│              Physical Hardware                 │
└────────────────────────────────────────────────┘
```

Each VM is heavy — often gigabytes — because it carries a full OS. Booting one takes minutes.

### Containers

A container virtualizes the **operating system**, not the hardware. All containers on a host **share the host's kernel**. There is no guest OS inside a container — just your app and its libraries. The Docker engine uses built-in Linux kernel features to give each container its own isolated view of the system.

```
┌─────────────────────────────────────────────┐
│  App A   │   App B   │   App C                │
├──────────┼───────────┼────────────────────────┤
│ Bins/Lib │ Bins/Lib  │ Bins/Lib  ← no OS!     │
├──────────┴───────────┴────────────────────────┤
│         Docker Engine (daemon)                 │
├────────────────────────────────────────────────┤
│   Host Operating System (shared kernel)        │
├────────────────────────────────────────────────┤
│              Physical Hardware                 │
└────────────────────────────────────────────────┘
```

### Side-by-side

| Aspect | Virtual Machine | Container |
|---|---|---|
| Virtualizes | Hardware | Operating system |
| Contains a full OS? | Yes (guest OS + kernel) | No (shares host kernel) |
| Typical size | Gigabytes | Megabytes |
| Startup time | Seconds to minutes | Milliseconds to seconds |
| Isolation | Very strong (hardware-level) | Strong (process-level) |
| Density (how many per host) | Tens | Hundreds to thousands |
| Overhead | High | Very low |

### The key takeaway

> A container is essentially **a process (or group of processes) running on the host, given an isolated and restricted view of the system.** It is *not* a separate machine. When you run a container, you are running a normal Linux process — it just can't see the rest of the system.

This is why containers start almost instantly: there's no OS to boot. You're just starting a process.

**One important consequence:** because containers share the host kernel, a Linux container needs a Linux kernel to run. On macOS and Windows, Docker Desktop quietly runs a tiny Linux VM in the background, and your containers run inside *that*. This is normal and you rarely have to think about it.

---

## 3. How Containers Actually Work (Linux internals)

You don't need to master these on Day 1, but knowing the names demystifies the "magic." Containers are built from three ordinary Linux kernel features:

### Namespaces — *"what a process can see"*

Namespaces partition kernel resources so that one set of processes sees one set of resources, and another set sees a different set. Docker uses several:

- **PID namespace** — the container has its own process tree. Inside, your app might be PID 1, and it can't see host processes.
- **NET namespace** — the container gets its own network interfaces, IP address, and routing table.
- **MNT namespace** — its own filesystem mount points (its own view of `/`).
- **UTS namespace** — its own hostname.
- **IPC namespace** — its own inter-process communication resources.
- **USER namespace** — can map container users to different host users.

Namespaces are what give a container the *illusion* of being its own machine.

### Control Groups (cgroups) — *"how much a process can use"*

cgroups limit and meter resource usage — how much CPU, memory, disk I/O, and network bandwidth a container is allowed to consume. This is how you can say "this container may use at most 512 MB of RAM and half a CPU core."

### Union Filesystems (OverlayFS) — *"how images are layered"*

Docker images are built from stacked, read-only **layers**. OverlayFS merges these layers into a single coherent filesystem view, and adds a thin **writable layer** on top when a container runs. This is why images are efficient to store and fast to ship — common layers are shared and never duplicated. (You'll go deep on this on Day 2.)

> **Mental model:** Namespaces = isolation (what you see). cgroups = limits (what you can use). Union FS = efficient layered storage (how the image is assembled).

---

## 4. Docker Architecture

Docker uses a **client–server architecture**. There are four pieces you should know.

```
   ┌──────────────┐         REST API          ┌───────────────────────────┐
   │ Docker Client│ ───────────────────────▶  │   Docker Daemon (dockerd) │
   │  (docker CLI)│ ◀───────────────────────  │                           │
   └──────────────┘                           │  • builds images          │
                                              │  • runs containers        │
                                              │  • manages volumes,       │
   ┌──────────────┐                           │    networks               │
   │   Registry   │ ◀──── pull / push ──────▶ │                           │
   │ (Docker Hub) │                           └───────────────────────────┘
   └──────────────┘
```

### 1. Docker Client (`docker`)

The command-line tool you type into. When you run `docker run nginx`, the client doesn't do the work itself — it sends your command as an API request to the daemon.

### 2. Docker Daemon (`dockerd`)

The background service that does all the real work: building images, starting and stopping containers, managing volumes and networks. It listens for API requests from the client. The client and daemon are usually on the same machine, but they don't have to be — the client can talk to a remote daemon.

### 3. Registry

A storage and distribution system for images. **Docker Hub** is the default public registry (like GitHub, but for container images). When you `docker pull nginx`, the daemon fetches the image from a registry. Companies also run private registries (AWS ECR, Google Artifact Registry, GitHub Container Registry, Harbor).

### 4. Docker Objects

The things the daemon manages:

- **Images** — read-only templates (the blueprint).
- **Containers** — running (or stopped) instances of images.
- **Volumes** — persistent data storage (Day 3).
- **Networks** — how containers communicate (Day 4).

### What happens when you run `docker run nginx`

1. The **client** sends a "run nginx" request to the **daemon**.
2. The daemon checks if the `nginx` **image** is available locally.
3. If not, it **pulls** it from the **registry** (Docker Hub).
4. The daemon **creates a container** from the image.
5. It allocates a filesystem (writable layer), a network interface, and an IP.
6. It **starts** the container, which runs nginx.

You typed one command; all of that happened underneath.

---

## 5. Key Vocabulary

Lock these terms in now — every later day assumes them.

| Term | Meaning |
|---|---|
| **Image** | A read-only template containing your app + dependencies. The blueprint. |
| **Container** | A running (or stopped) instance of an image. The actual thing executing. |
| **Dockerfile** | A text file with instructions to build an image (Day 3). |
| **Registry** | A server that stores and distributes images (e.g. Docker Hub). |
| **Repository** | A named collection of related images, usually different versions (e.g. `nginx`). |
| **Tag** | A label for a specific image version (e.g. `nginx:1.27`, `nginx:latest`). |
| **Docker Hub** | The default public registry. |
| **Daemon (dockerd)** | The background engine that does the work. |
| **Docker Desktop** | The GUI + bundled engine for macOS/Windows (and optionally Linux). |

> **Image vs Container — the analogy that sticks:** An image is like a *class* in programming; a container is an *object* (an instance) created from it. Or: an image is a *recipe*; a container is the *cooked dish*. You can make many dishes (containers) from one recipe (image).

---

## 6. Installation

Pick your operating system below. The goal is a working `docker` command and a running daemon.

### macOS

1. Download **Docker Desktop for Mac** from <https://www.docker.com/products/docker-desktop/>.
2. Choose the right chip: **Apple Silicon** (M1/M2/M3/M4) or **Intel**.
3. Open the `.dmg` and drag Docker to Applications.
4. Launch Docker Desktop. Wait for the whale icon in the menu bar to stop animating — that means the daemon is ready.
5. Verify in a terminal:
   ```bash
   docker version
   ```

### Windows

1. Ensure **WSL 2** (Windows Subsystem for Linux) is enabled — Docker Desktop uses it to run the Linux engine. The installer will prompt you if it's missing.
2. Download **Docker Desktop for Windows** from the same link above.
3. Run the installer, keep the "Use WSL 2" option checked, and restart if asked.
4. Launch Docker Desktop and wait for it to report "running."
5. Verify in PowerShell or a WSL terminal:
   ```bash
   docker version
   ```

### Linux (Ubuntu/Debian) — Docker Engine

On Linux you typically install **Docker Engine** directly (no Desktop needed, though Desktop for Linux exists). Use the official repository so you get current versions:

```bash
# 1. Remove any old/conflicting packages
sudo apt-get remove docker docker-engine docker.io containerd runc

# 2. Set up Docker's official APT repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Install the engine + CLI + Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

> ⚠️ **Always check the official docs** at <https://docs.docker.com/engine/install/> for the latest commands — package names and steps change over time. Treat the block above as a guide, not gospel.

### Critical Linux post-install step: run Docker without `sudo`

By default the Docker daemon socket is owned by `root`, so every command needs `sudo`. Add yourself to the `docker` group to avoid that:

```bash
sudo groupadd docker          # usually already exists
sudo usermod -aG docker $USER # add yourself to the group
newgrp docker                 # activate the group in this shell
# then LOG OUT and back IN for it to apply everywhere
```

> 🔐 **Security note for later (Day 5):** being in the `docker` group is effectively equivalent to root access on the host, because you can mount the host filesystem into a container. That's fine on your personal dev machine — just know it's not something to hand out casually on shared servers.

---

## 7. Verifying Your Install

Run these three checks. All three should succeed before you go further.

```bash
# 1. Is the client installed and can it reach the daemon?
docker version
```
You should see two sections — **Client** and **Server**. If you only see Client (and an error about the daemon), the daemon isn't running. Start Docker Desktop, or on Linux: `sudo systemctl start docker`.

```bash
# 2. System-wide info: how many containers/images, storage driver, etc.
docker info
```

```bash
# 3. The classic smoke test
docker run hello-world
```

`hello-world` is a tiny image whose only job is to print a success message. When you run it, Docker pulls the image (if needed), creates a container, runs it, prints text explaining what just happened, and exits. If you see *"Hello from Docker!"*, your installation works end to end.

---

## 8. The Container Lifecycle

A container moves through a set of states. Understanding this prevents a lot of early confusion (especially "why is my container not in `docker ps`?").

```
   docker create          docker start
        │                      │
        ▼                      ▼
   ┌─────────┐   start    ┌─────────┐   stop/exit   ┌─────────┐   rm    ┌──────────┐
   │ Created │ ─────────▶ │ Running │ ────────────▶ │ Stopped │ ──────▶ │ Removed  │
   └─────────┘            └─────────┘               │(Exited) │         │ (gone)   │
                              ▲   │                 └─────────┘         └──────────┘
                              │   │ pause              │
                              │   ▼                    │ start
                          ┌─────────┐                  │
                          │ Paused  │ ◀────────────────┘
                          └─────────┘
```

- **`docker run`** is the everyday command — it's actually **`create` + `start` combined** in one step, plus it pulls the image first if needed.
- **`docker create`** makes a container but doesn't start it (rarely used alone).
- **`docker start`** starts an existing stopped/created container.
- **`docker stop`** gracefully stops a running container (sends SIGTERM, then SIGKILL after a grace period).
- **`docker kill`** stops it immediately (SIGKILL).
- **`docker rm`** deletes a stopped container permanently.

### Two facts that trip up beginners

1. **A stopped container still exists.** `docker ps` shows only *running* containers. To see stopped ones too, use `docker ps -a`. Stopped containers take up disk and keep their name until you `docker rm` them.

2. **A container lives only as long as its main process.** A container isn't a "box that stays open" — it runs one main process, and when that process exits, the container stops. That's why `docker run ubuntu` seems to "do nothing and immediately stop": Ubuntu's default command finishes instantly, so the container exits. To keep it alive interactively, you give it something to do, like a shell: `docker run -it ubuntu bash`.

---

## 9. Essential Commands Reference

Your Day 1 toolkit. Type each one at least once during the labs — reading isn't enough.

### Getting info

```bash
docker version           # client + server versions
docker info              # system-wide details
docker --help            # list all commands
docker run --help        # help for a specific command
```

### Images

```bash
docker pull nginx        # download an image from Docker Hub
docker pull nginx:1.27   # download a specific tag/version
docker images            # list local images (also: docker image ls)
docker rmi nginx         # remove a local image
```

### Running containers

```bash
docker run hello-world                 # run a container (pull if needed)
docker run -it ubuntu bash             # interactive shell inside ubuntu
docker run -d nginx                    # detached (runs in background)
docker run -d -p 8080:80 nginx         # map host port 8080 → container port 80
docker run --name web -d nginx         # give the container a friendly name
docker run --rm hello-world            # auto-delete container when it exits
```

Flag cheat sheet:
- `-i` = interactive (keep STDIN open)
- `-t` = allocate a pseudo-TTY (gives you a proper terminal)
- `-it` = the two combined, for interactive shells
- `-d` = detached (background)
- `-p HOST:CONTAINER` = publish/map a port
- `--name` = name the container
- `--rm` = remove container automatically on exit

### Inspecting & interacting

```bash
docker ps                # list RUNNING containers
docker ps -a             # list ALL containers (incl. stopped)
docker logs <name|id>    # view a container's output
docker logs -f <name|id> # follow logs live (like tail -f)
docker exec -it web bash # open a shell in an ALREADY-RUNNING container
docker inspect web       # full JSON details (IP, mounts, config…)
docker stats             # live resource usage (CPU/mem) per container
docker top web           # processes running inside the container
```

> **`run` vs `exec` — don't confuse them:** `docker run` creates a *new* container from an image. `docker exec` runs a command inside an *existing, already-running* container. Use `run` to start something; use `exec` to "step into" something already started.

### Stopping & cleaning up

```bash
docker stop web          # graceful stop
docker kill web          # force stop
docker start web         # start a stopped container again
docker restart web       # stop then start
docker rm web            # delete a stopped container
docker rm -f web         # force-delete even if running
docker container prune   # delete ALL stopped containers
docker image prune       # delete dangling (unused) images
docker system prune       # delete all unused data (careful!)
```

### Referring to containers

You can reference a container by its **name** (e.g. `web`) or by its **ID**. You only need enough of the ID to be unique — usually the first 3–4 characters, e.g. `docker stop a1b2`.

---

## 10. Hands-On Labs

> **Do these in order.** Don't copy-paste blindly — type the commands, read the output, and predict what each one will do before you hit Enter. That's how it sticks.

### Lab 0 — Smoke test

```bash
docker run hello-world
```
✅ **Expected:** A "Hello from Docker!" message explaining the steps Docker just took. If you see it, your install is healthy.

---

### Lab 1 — Run a real web server and visit it

Run nginx in the background, mapping container port 80 to host port 8080:

```bash
docker run -d --name myweb -p 8080:80 nginx
```

Now open your browser to **<http://localhost:8080>** — you should see the "Welcome to nginx!" page. 🎉

Check it's running and read its logs:

```bash
docker ps
docker logs myweb
```

Refresh the browser a couple of times, then re-check `docker logs myweb` — you'll see your requests appear in the access log.

**What you just learned:** detached mode (`-d`), port mapping (`-p`), naming (`--name`), and that a container can serve real traffic.

---

### Lab 2 — Get a shell inside a container & prove statelessness

Start an interactive Ubuntu container:

```bash
docker run -it --name playground ubuntu bash
```

Your prompt changes — you're now *inside* the container as root. Try:

```bash
whoami           # root
cat /etc/os-release   # Ubuntu, even if your host is something else
ls /
apt-get update && apt-get install -y curl   # install a tool
curl --version   # it works now
exit             # leave the container — this stops it
```

Now start a **brand-new** Ubuntu container and check for curl:

```bash
docker run -it ubuntu bash
curl --version   # NOT found!
exit
```

**What you just learned:** Each container starts fresh from the image. The `curl` you installed lived only in the writable layer of that *one* container — a new container from the same image doesn't have it. This is the **statelessness** of containers, and it's exactly why we'll need **volumes** (Day 3) to persist data.

---

### Lab 3 — Container lifecycle in action

```bash
docker ps -a               # see your stopped 'playground' and the nginx
docker start playground    # start the stopped ubuntu container...
docker ps                  # ...but it's not here — why?
```

It immediately stopped again, because its main process (`bash`) had no interactive terminal attached this time, so it exited. This reinforces: **a container lives only as long as its main process.**

Now manage the nginx container's lifecycle:

```bash
docker stop myweb          # stop it (browser at :8080 now fails)
docker ps                  # myweb is gone from the running list
docker ps -a               # but still exists, status "Exited"
docker start myweb         # bring it back
docker restart myweb       # stop + start in one go
```

---

### Lab 4 — Exec into a running container

With `myweb` (nginx) running:

```bash
docker exec -it myweb bash       # step inside the live container
ls /usr/share/nginx/html         # nginx's web root
cat /usr/share/nginx/html/index.html   # the welcome page source
echo "Hello from inside the container" > /usr/share/nginx/html/index.html
exit
```

Refresh **<http://localhost:8080>** — the page now shows your custom text. You edited a live container from the inside.

**What you just learned:** the difference between `run` (new container) and `exec` (into an existing one), and where nginx serves files from.

---

### Lab 5 — Inspect and observe

```bash
docker inspect myweb         # huge JSON — scroll for "IPAddress"
docker inspect -f '{{ .NetworkSettings.IPAddress }}' myweb  # just the IP
docker stats                 # live CPU/memory (press Ctrl+C to exit)
docker top myweb             # processes running inside
```

`docker inspect` is your go-to whenever you need to know exactly how a container is configured — its IP, mounts, env vars, ports, and more.

---

### Lab 6 — Clean up

Good hygiene. Remove what you made:

```bash
docker stop myweb
docker rm myweb                 # remove the nginx container
docker rm playground            # remove the ubuntu container
docker ps -a                    # confirm they're gone
docker container prune          # nuke any remaining stopped containers
docker images                   # see images still cached locally
```

Note the **images** are still there even after removing containers — images and containers are separate. You can remove an image with `docker rmi nginx` (only works once no container is using it).

---

## 11. Common Mistakes & Gotchas

- **"My container exited immediately!"** — Its main process finished. For a shell you need `-it` and a command like `bash`. For a server it stays up because the server process keeps running.
- **"Port already allocated."** — Something else is using that host port. Pick another, e.g. `-p 8081:80`.
- **Forgetting `-a` on `docker ps`** — Stopped containers are invisible without `-a`. They're not gone, just not running.
- **Confusing the two ports in `-p 8080:80`** — Format is **`HOST:CONTAINER`**. The left number is what you type in the browser; the right is the port the app listens on inside.
- **Mixing up `run` and `exec`** — `run` = new container. `exec` = into an existing one.
- **Needing `sudo` for every command on Linux** — You skipped the post-install `docker` group step (Section 6). Add yourself to the group and re-login.
- **Editing a container and expecting it to persist** — Changes inside a container vanish when it's removed. Persistence requires volumes (Day 3) or baking changes into an image (Day 2/3).

---

## 12. Day 1 Checklist

Tick these off before moving to Day 2. If any feels shaky, redo the relevant lab.

- [ ] I can explain, in one sentence, how a container differs from a VM.
- [ ] I know that all containers share the host kernel and a container is essentially an isolated process.
- [ ] I can name the four parts of Docker's architecture (client, daemon, registry, objects).
- [ ] I can clearly state the difference between an **image** and a **container**.
- [ ] Docker is installed and `docker run hello-world` works.
- [ ] (Linux) I can run `docker` without `sudo`.
- [ ] I pulled an image and ran a container that I visited in a browser.
- [ ] I opened an interactive shell with `docker run -it` and with `docker exec -it`.
- [ ] I understand why a tool I installed inside one container wasn't in a new container (statelessness).
- [ ] I can list running vs all containers (`docker ps` vs `docker ps -a`).
- [ ] I stopped, started, and removed containers, and cleaned up afterwards.
- [ ] I used `docker logs` and `docker inspect` at least once.

---

### What's next — Day 2 preview

Tomorrow you go under the hood of **images and containers**: the layered filesystem, how caching makes builds fast, image tags and digests, the registry workflow (push your own image to Docker Hub), and the read-only-image / writable-container-layer model. Today's statelessness lab is the perfect setup for it.

> **Tip:** Keep a personal `docker-notes.md` as you go. Every time a command surprises you, write down what you expected vs what happened. That file will be worth more than any tutorial by Day 5.