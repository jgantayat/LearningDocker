# Docker — Day 2: Images, Containers & the Registry

> **Goal for today:** Understand what an image *really* is — the layered, copy-on-write filesystem underneath it — how images are named, tagged, and distributed through registries, and the precise relationship between a read-only image and the thin writable layer of a running container. By the end you'll be able to inspect layers, save and load images as files, commit a container into a new image, and push your own image to Docker Hub.
>
> **Time budget:** ~30 min theory → ~90 min hands-on → ~30 min review and notes.
>
> **Prerequisite:** Day 1 complete — Docker installed, `docker run hello-world` working, comfortable with the basic lifecycle commands.

---

## Table of Contents

1. [Recap: Image vs Container](#1-recap-image-vs-container)
2. [The Layer Model](#2-the-layer-model)
3. [Copy-on-Write & the Writable Container Layer](#3-copy-on-write--the-writable-container-layer)
4. [Image Naming: Registry, Repository, Tag, Digest](#4-image-naming-registry-repository-tag-digest)
5. [Registries & the Push/Pull Workflow](#5-registries--the-pushpull-workflow)
6. [Inspecting Images: history, inspect, manifest](#6-inspecting-images-history-inspect-manifest)
7. [Moving Images Around: save, load, export, import](#7-moving-images-around-save-load-export-import)
8. [Creating Images from Containers: commit](#8-creating-images-from-containers-commit)
9. [Essential Commands Reference](#9-essential-commands-reference)
10. [Hands-On Labs](#10-hands-on-labs)
11. [Common Mistakes & Gotchas](#11-common-mistakes--gotchas)
12. [Day 2 Checklist](#12-day-2-checklist)

---

## 1. Recap: Image vs Container

From Day 1, the one-line version: **an image is the blueprint; a container is a running instance of it.** Today we make that precise.

- An **image** is a read-only, immutable bundle of filesystem content + metadata (what command to run, which ports it expects, environment variables, etc.). It never changes once built.
- A **container** is an image plus a thin **writable layer** on top, plus runtime state (a running process, an IP address, etc.).

> The programming analogy still holds: image = *class*, container = *object*. But today's key insight is *how* that works physically — through **layers** and **copy-on-write**. Everything else on Day 2 builds on those two ideas.

---

## 2. The Layer Model

An image is not a single monolithic blob. It is a **stack of read-only layers**, each one representing a set of filesystem changes. When an image is built (Day 3), each instruction in the build recipe produces a new layer on top of the previous ones.

```
       IMAGE = stack of read-only layers

   ┌───────────────────────────────────┐
   │  Layer 4: COPY app code            │  ← top
   ├───────────────────────────────────┤
   │  Layer 3: RUN pip install deps     │
   ├───────────────────────────────────┤
   │  Layer 2: RUN apt-get install ...  │
   ├───────────────────────────────────┤
   │  Layer 1: base OS files (python)   │  ← bottom (FROM)
   └───────────────────────────────────┘
```

Each layer stores **only the differences** it introduces — the files it added, changed, or deleted relative to the layer below. OverlayFS (the union filesystem from Day 1) merges the whole stack into a single coherent filesystem that the container sees as a normal `/`.

### Why layers matter — three big payoffs

**1. Sharing / deduplication.** Layers are content-addressed (identified by a hash of their contents). If two images are both built `FROM python:3.12-slim`, they *share* those base layers on disk — Docker stores them once. Pull a second python-based image and you'll see `Already exists` for the shared layers; only the new layers download.

**2. Caching.** During a build, if a layer's inputs haven't changed, Docker reuses the cached layer instead of rebuilding it. This is why a second build of the same Dockerfile is near-instant, and why **layer ordering matters** (you'll exploit this heavily on Day 3 — put rarely-changing steps first so the cache survives).

**3. Efficient transfer.** When you `pull` or `push`, only layers the other side doesn't already have move across the network.

### A concrete mental picture

Think of layers like **transparent sheets stacked on an overhead projector.** Each sheet adds or obscures some marks. Looking down through the whole stack, you see one combined picture — but each sheet is stored and shipped separately, and identical sheets can be reused across many stacks.

---

## 3. Copy-on-Write & the Writable Container Layer

Here's the part that makes containers cheap to start and explains the statelessness you saw in Day 1's Lab 2.

When you run a container, Docker takes the image's read-only layer stack and adds **one thin writable layer on top**, just for that container:

```
   CONTAINER

   ┌───────────────────────────────────┐
   │  Writable container layer  ✏️       │  ← per-container, read/write
   ├═══════════════════════════════════┤
   │  Layer 4: COPY app code  (RO)      │  ┐
   ├───────────────────────────────────┤  │
   │  Layer 3: pip install    (RO)      │  │  shared image layers,
   ├───────────────────────────────────┤  │  read-only, reused by
   │  Layer 2: apt install    (RO)      │  │  every container from
   ├───────────────────────────────────┤  │  this image
   │  Layer 1: base python    (RO)      │  ┘
   └───────────────────────────────────┘
```

### How copy-on-write (CoW) works

- **Reading a file:** the container reads it straight from whichever read-only layer contains it. No copying. Fast, and shared across all containers from that image.
- **Modifying a file:** the file is first **copied up** from its read-only layer into the writable layer, and the change is made to that copy. The original in the read-only layer is untouched.
- **Creating a new file:** it's written directly into the writable layer.
- **Deleting a file:** a "whiteout" marker is written in the writable layer that hides the file from view — the underlying read-only copy still exists but is no longer visible.

This is **copy-on-write**: nothing is copied until something is written.

### Why this explains Day 1's Lab 2

When you `apt-get install curl` inside a container, `curl` lands in **that container's writable layer** — not the image. The image is read-only and never changes. When you removed that container, its writable layer (and your `curl`) was destroyed. A brand-new container from the same image gets a *fresh, empty* writable layer — so `curl` isn't there.

### Two crucial consequences

1. **Containers are ephemeral by default.** Anything written inside a running container lives only in that container's writable layer and dies with it. To persist data you need **volumes** (Day 3).

2. **Starting a container is cheap.** Docker doesn't copy the whole image — it just slaps a new empty writable layer on top of the shared read-only stack. That's why containers start in milliseconds and why you can run hundreds from one image without using hundreds of times the disk.

> **`docker diff` lets you *see* the writable layer.** It lists exactly which files a container has Added, Changed, or Deleted relative to its image. You'll use it in the labs.

---

## 4. Image Naming: Registry, Repository, Tag, Digest

Every image has a full name. Understanding each part removes a lot of confusion.

```
        registry      repository        tag
   ┌───────────────┐ ┌──────────┐    ┌──────┐
   registry.example.com/team/webapp  :  1.4.2
   └──────────────────────────────────────────┘
                full image reference
```

| Part | Example | Meaning |
|---|---|---|
| **Registry** | `registry.example.com` | The host that stores the image. Omitted → defaults to Docker Hub. |
| **Repository** | `team/webapp` | A named collection of related images (different versions of the same app). |
| **Tag** | `1.4.2` | A human-friendly label for one specific version. Omitted → defaults to `latest`. |

### Defaults you should internalize

When you type `docker pull nginx`, Docker expands it to:

```
docker.io/library/nginx:latest
```

- `docker.io` → Docker Hub (the default registry).
- `library/` → the namespace for official images.
- `nginx` → the repository.
- `latest` → the default tag.

So `nginx`, `nginx:latest`, and `docker.io/library/nginx:latest` all mean the same thing.

### Tags are mutable — digests are not

A **tag** is just a movable label. The `nginx:1.27` tag can be re-pointed by the maintainer to a rebuilt image tomorrow. That means a tag is **not** a guarantee of getting the exact same bytes every time.

A **digest** is a cryptographic hash of the image's content — it's immutable and unique:

```
nginx@sha256:a8f2c3...e91b
```

Pulling by digest guarantees you get *exactly* that image, byte for byte, forever. This matters enormously for security and reproducibility (you'll pin digests on Day 5 for hardening).

> ⚠️ **`latest` is a trap.** It is *not* magic — it doesn't mean "the newest version." It's literally just the tag applied when none is specified. An image tagged `latest` can be months old. **Never rely on `latest` in anything serious** — always pin an explicit version, and in production, a digest.

### The `--platform` flag

Images can be multi-architecture (e.g. `linux/amd64` and `linux/arm64`). Docker normally pulls the variant matching your machine. On an Apple Silicon Mac pulling an image that only ships `amd64`, or when you need a specific arch, use:

```bash
docker pull --platform linux/amd64 nginx
```

---

## 5. Registries & the Push/Pull Workflow

A **registry** stores and distributes images. The mental model: **a registry is to images what GitHub is to code repositories.**

```
   ┌──────────────┐    docker push     ┌──────────────────┐
   │  Your machine│ ─────────────────▶ │     Registry      │
   │  (local      │                    │  (Docker Hub,     │
   │   images)    │ ◀───────────────── │   ECR, GHCR …)    │
   └──────────────┘    docker pull     └──────────────────┘
```

Common registries:
- **Docker Hub** (`docker.io`) — the default public registry.
- **GitHub Container Registry** (`ghcr.io`).
- **Amazon ECR**, **Google Artifact Registry**, **Azure ACR** — cloud-provider registries.
- **Harbor** — a popular self-hosted private registry.

### The workflow to publish your own image

1. **Build or have an image locally** (you'll build properly on Day 3; today you'll create one via `commit`).
2. **Tag** it with your registry namespace. On Docker Hub that's your username:
   ```bash
   docker tag myimage:1.0 yourusername/myimage:1.0
   ```
3. **Log in** to the registry:
   ```bash
   docker login
   ```
4. **Push** it:
   ```bash
   docker push yourusername/myimage:1.0
   ```
5. Anyone (if public) can now `docker pull yourusername/myimage:1.0`.

> **`docker tag` doesn't copy anything.** It just creates an additional *name* pointing at the same underlying image. One image can have many tags simultaneously. This is why the namespace prefix (your username) is required before pushing — it tells the registry where the image belongs.

---

## 6. Inspecting Images: history, inspect, manifest

You can't optimize or secure what you can't see. Three commands let you look inside an image.

### `docker image history` — see the layers

```bash
docker image history python:3.12-slim
```

This shows each layer, the instruction that created it, and its size. It's how you spot which layer is making an image fat — invaluable for the optimization work on Days 3 and 5. Add `--no-trunc` to see full commands:

```bash
docker image history --no-trunc python:3.12-slim
```

### `docker image inspect` — full metadata

```bash
docker image inspect nginx
```

A big JSON document: the layer hashes, environment variables, exposed ports, the default command/entrypoint, architecture, OS, creation date, and more. Pull out one field with a format string:

```bash
docker image inspect -f '{{ .Config.Env }}' nginx
docker image inspect -f '{{ .Architecture }}' nginx
```

### The manifest — what gets pulled

A **manifest** is the registry's description of an image: which layers it's composed of and, for multi-arch images, which variant exists for each platform. You'll mostly interact with manifests indirectly (via pull), but it's worth knowing the term: when you pull by digest, you're pulling a specific manifest.

---

## 7. Moving Images Around: save, load, export, import

Sometimes you need an image as a plain file — to move it to an air-gapped machine, archive it, or share it without a registry. Two pairs of commands do this, and the difference between them is a classic interview question.

### `save` / `load` — for **images** (preserves layers & history)

```bash
docker save -o nginx.tar nginx:latest   # image → tar file
docker load -i nginx.tar                 # tar file → image
```

`save` exports a full image including **all its layers and metadata**. `load` re-imports it exactly as it was. This is the right tool for moving images between machines without a registry.

### `export` / `import` — for **containers** (flattens, loses history)

```bash
docker export -o rootfs.tar <container>  # container filesystem → tar
docker import rootfs.tar myflat:1.0      # tar → single-layer image
```

`export` dumps a **container's filesystem** as a flat tarball — it does **not** preserve layers, history, or image config (no CMD/ENV/etc.). `import` turns such a tarball into a new single-layer image.

> **save vs export — the one-liner:** `save` works on **images** and keeps the **layered history**; `export` works on **containers** and **flattens** everything into one layer, discarding metadata. Use `save`/`load` for distributing images; `export`/`import` only when you specifically want a flattened snapshot.

---

## 8. Creating Images from Containers: commit

You can turn a modified container into a new image with `docker commit`. This captures the container's current state — including its writable layer — as a fresh image layer.

```bash
docker commit <container> mynewimage:1.0
```

This is useful for experimentation and for understanding how layers form, but be aware:

> ⚠️ **`commit` is not how real images are built.** Images created by `commit` are opaque — there's no record of *how* they were made, so they're not reproducible, hard to audit, and easy to bloat. In practice, images are built from a **Dockerfile** (Day 3), which is version-controllable, reviewable, and reproducible. Treat `commit` as a learning tool and an occasional "quick snapshot," not a production workflow.

You'll use `commit` in today's labs precisely to *see* the layer model in action — and then on Day 3 you'll learn the proper, repeatable way.

---

## 9. Essential Commands Reference

Your Day 2 toolkit. Type each at least once during the labs.

### Listing & inspecting images

```bash
docker images                       # list local images (alias: docker image ls)
docker image ls                     # same thing
docker image history <image>        # show layers + sizes
docker image history --no-trunc <image>   # full commands, untruncated
docker image inspect <image>        # full JSON metadata
docker image inspect -f '{{ .Architecture }}' <image>   # one field
docker system df                    # disk used by images/containers/volumes
```

### Pulling with options

```bash
docker pull nginx                   # defaults to :latest from Docker Hub
docker pull nginx:1.27              # specific tag
docker pull --platform linux/amd64 nginx   # specific architecture
docker pull nginx@sha256:<digest>   # pull an exact image by digest
```

### Tagging, login, push

```bash
docker tag <image> <user>/<repo>:<tag>   # add a name (no copy)
docker login                             # authenticate to a registry
docker push <user>/<repo>:<tag>          # upload to the registry
docker logout
```

### Inspecting a container's writable layer

```bash
docker diff <container>             # files Added/Changed/Deleted vs image
docker cp <container>:/path ./local # copy a file OUT of a container
docker cp ./local <container>:/path # copy a file INTO a container
```

### Snapshotting & moving

```bash
docker commit <container> <image>:<tag>   # container state → new image
docker save -o file.tar <image>           # image → tar (keeps layers)
docker load -i file.tar                   # tar → image
docker export -o file.tar <container>     # container fs → flat tar
docker import file.tar <image>:<tag>      # flat tar → single-layer image
```

### Cleaning up images

```bash
docker rmi <image>                  # remove an image (no container using it)
docker image prune                  # remove dangling (untagged) images
docker image prune -a               # remove ALL unused images (careful!)
```

---

## 10. Hands-On Labs

> **Do these in order, type don't paste, and predict each output first.** Today's labs are about *seeing* the layer model and *doing* the registry workflow end to end.

### Lab 1 — See the layers

Pull a small but real image and inspect its layer history:

```bash
docker pull python:3.12-slim
docker image history python:3.12-slim
```

Read the output top to bottom. Each row is a layer; note the **SIZE** column. Now pull a second python-based image and watch layer sharing:

```bash
docker pull python:3.12-slim-bookworm
```

Notice some layers say **`Already exists`** — those are shared with the image you already have, so they don't re-download.

✅ **What you learned:** images are stacks of layers, sizes vary wildly per layer, and identical layers are stored/transferred once.

---

### Lab 2 — Watch copy-on-write with `docker diff`

Start a container and modify its filesystem:

```bash
docker run -dit --name lab2 ubuntu bash
docker exec lab2 bash -c "apt-get update && apt-get install -y cowsay"
docker exec lab2 bash -c "echo hello > /root/note.txt"
docker exec lab2 rm /etc/hostname.bak 2>/dev/null; true
```

Now see exactly what changed relative to the image:

```bash
docker diff lab2
```

You'll see lines prefixed with:
- `A` — **A**dded (e.g. `/root/note.txt`, the cowsay files)
- `C` — **C**hanged
- `D` — **D**eleted

Every one of those changes lives in this container's **writable layer** — the `ubuntu` image itself is untouched.

✅ **What you learned:** `docker diff` reveals the writable layer; this *is* the copy-on-write mechanism made visible.

---

### Lab 3 — Commit a container into a new image

Turn that modified container into an image and prove the change is captured:

```bash
docker commit lab2 mycowsay:1.0
docker images | grep mycowsay
docker run --rm mycowsay:1.0 bash -c "cowsay 'Layers are cool'"
```

The new image now contains `cowsay`, which the base `ubuntu` image did not. Look at its history:

```bash
docker image history mycowsay:1.0
```

The top layer is the one your `commit` created.

✅ **What you learned:** how a writable layer becomes a new read-only image layer — and *why* this is opaque (the history doesn't explain how cowsay got there; contrast this with a Dockerfile on Day 3).

---

### Lab 4 — Save an image to a file, delete it, load it back

Prove that `save`/`load` round-trips an image with its layers intact:

```bash
docker save -o mycowsay.tar mycowsay:1.0     # export to tar
ls -lh mycowsay.tar                          # see the file
docker rmi mycowsay:1.0                       # delete the image locally
docker images | grep mycowsay                 # gone
docker load -i mycowsay.tar                   # bring it back from the tar
docker images | grep mycowsay                 # it's back, layers and all
```

✅ **What you learned:** images can travel as plain files — no registry required — and `load` restores them fully.

---

### Lab 5 — Push your own image to Docker Hub

This is the real registry workflow.

1. Create a free account at <https://hub.docker.com> if you don't have one.
2. Log in from the terminal:
   ```bash
   docker login
   ```
3. Tag your image with your username (replace `yourusername`):
   ```bash
   docker tag mycowsay:1.0 yourusername/mycowsay:1.0
   docker images | grep mycowsay        # note: same IMAGE ID, two names
   ```
4. Push it:
   ```bash
   docker push yourusername/mycowsay:1.0
   ```
5. Verify on the Docker Hub website — your repository now appears. To fully prove portability, remove the local copies and pull it back:
   ```bash
   docker rmi yourusername/mycowsay:1.0 mycowsay:1.0
   docker pull yourusername/mycowsay:1.0
   docker run --rm yourusername/mycowsay:1.0 bash -c "cowsay 'Pulled from the cloud'"
   ```

✅ **What you learned:** tag → login → push → pull — the complete distribution cycle, and that `docker tag` just adds a name to the same image (same IMAGE ID).

---

### Lab 6 — Pull by digest (immutability)

Get the digest of an image and pull that exact version:

```bash
docker inspect --format='{{index .RepoDigests 0}}' python:3.12-slim
```

Copy the `python@sha256:...` value it prints, then:

```bash
docker pull python@sha256:<the-digest-you-copied>
```

✅ **What you learned:** a digest pins an exact, immutable image — the foundation of reproducible, secure pulls you'll rely on for Day 5 hardening.

---

### Lab 7 — Clean up & check disk usage

```bash
docker system df                    # where your disk is going
docker stop lab2 && docker rm lab2  # remove the lab container
docker image prune                  # remove dangling images
rm mycowsay.tar                     # remove the tarball
docker images                       # review what remains
```

✅ **What you learned:** `docker system df` is your map of disk usage; images persist independently of containers and need explicit cleanup.

---

## 11. Common Mistakes & Gotchas

- **Treating `latest` as "newest."** It's just the default tag, not a freshness guarantee. Pin explicit versions.
- **Confusing `save`/`load` with `export`/`import`.** `save` = images, keeps layers. `export` = containers, flattens and loses metadata. Mixing them up loses your CMD/ENV/history.
- **Expecting `docker tag` to copy or move an image.** It only adds another name pointing at the same image ID.
- **Pushing without the namespace prefix.** `docker push mycowsay:1.0` fails or targets the wrong place — it must be `yourusername/mycowsay:1.0`.
- **Forgetting `docker login` before push.** You'll get an authentication error.
- **Thinking `commit` is how you build images.** It works, but it's opaque and unreproducible — Dockerfiles (Day 3) are the real answer.
- **Surprised that removing containers didn't reclaim much space.** Images are separate; use `docker image prune` / `docker system prune` and `docker system df` to manage image disk usage.
- **Believing changes inside a container modify its image.** They never do — they live in the container's writable layer and vanish with the container.

---

## 12. Day 2 Checklist

Tick these before moving to Day 3. Redo the matching lab for anything shaky.

- [ ] I can explain that an image is a **stack of read-only layers**, each storing only its differences.
- [ ] I can state three benefits of layering (sharing/dedup, caching, efficient transfer).
- [ ] I can explain **copy-on-write** and why a container gets a thin **writable layer** on top of read-only image layers.
- [ ] I can connect CoW back to Day 1's statelessness (why installed tools vanished in a new container).
- [ ] I can break down a full image reference into **registry / repository / tag**, and state the Docker Hub defaults.
- [ ] I understand the difference between a **tag** (mutable) and a **digest** (immutable), and why `latest` is risky.
- [ ] I used `docker image history` and `docker image inspect` to look inside an image.
- [ ] I used `docker diff` to see a container's writable layer.
- [ ] I created an image from a container with `docker commit` — and I know why that's not the real way to build images.
- [ ] I round-tripped an image with `docker save` / `docker load`.
- [ ] I can explain the difference between `save`/`load` and `export`/`import`.
- [ ] I completed the full registry workflow: `tag` → `login` → `push` → `pull`.
- [ ] I pulled an image by **digest**.
- [ ] I checked disk usage with `docker system df` and cleaned up.

---

### What's next — Day 3 preview

Tomorrow is the heaviest, most rewarding day: **building your own images the right way.** You'll write **Dockerfiles** (the reproducible alternative to today's `commit`), learn every key instruction (`FROM`, `RUN`, `COPY`, `CMD` vs `ENTRYPOINT`, `ENV` vs `ARG`, `HEALTHCHECK`…), exploit the **layer caching** you saw today to make builds fast, slash image size with **multi-stage builds**, and finally solve the statelessness problem with **volumes** for persistent data. Everything you saw about layers today is the foundation for it.

> **Tip:** In your personal `docker-notes.md`, write down — in your own words — the difference between an image layer and a container's writable layer, and the difference between `save` and `export`. Those two distinctions come up constantly.