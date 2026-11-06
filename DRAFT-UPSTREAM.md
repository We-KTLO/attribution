# Draft upstream notes: container builds on arm64 (Grace) racks

This file is a working draft that lives only on this fork branch. It is not
meant to be merged. CONTRIBUTING.md routes feature requests and design
proposals to GitHub Discussions; a maintainer opens a tracking issue once a
proposal is accepted. Nothing here has been filed upstream.

---

## 1. Kaniko-compatible uv cache mount: not worth proposing

### What was changed locally

A downstream build replaced the BuildKit cache mount on the final `uv sync` of
`service/service-app/Dockerfile`:

```dockerfile
# upstream
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-editable --no-dev

# downstream
RUN \
    uv sync --frozen --no-editable --no-dev
```

because that build ran in Kaniko, which does not implement BuildKit's
`RUN --mount` flags.

### Assessment

- There is no clean way to keep the cache mount and also build without
  BuildKit. `RUN --mount` is BuildKit Dockerfile syntax; the legacy builder and
  Kaniko do not accept it, and the `# syntax=docker/dockerfile:1.7` line is
  only honoured by BuildKit. A second Dockerfile, a build-arg switch or a
  sed step would all add maintenance for an unsupported builder.
- Dropping the mount upstream would make every BuildKit rebuild slower for
  the project's own documented path (`docker build -f
  service/service-app/Dockerfile`, see `docs/service-operators.md`) in exchange
  for supporting a builder the project does not use. The mount only speeds up
  the second `uv sync` anyway (the first, dependency-only `uv sync` has no
  mount), so its absence costs little downstream.
- Kaniko itself is no longer maintained by its original owner, so asking
  upstream to target it is hard to justify.

**Conclusion:** keep this as a one-line downstream patch (or build with
BuildKit/buildx instead of Kaniko). Do not propose it upstream. No branch with
a code change was created for it.

---

## 2. Draft feature request: publish multi-arch (amd64 + arm64) service images

**Where:** GitHub Discussions (Ideas), per CONTRIBUTING.md.

**Title:** `Publish multi-arch (linux/amd64 + linux/arm64) container images for the attribution service`

### Context

`RELEASE.md` says the project does not currently publish packages, prebuilt
container images or Helm charts, and that prebuilt images (including an NGC
image) and a public Helm chart are planned. Operators build
`service/service-app/Dockerfile` themselves today.

### Request

When images are published, publish them as a multi-arch manifest list covering
both `linux/amd64` and `linux/arm64`, for the service image and, if it is
published, the blueprint web image.

### Why arm64 matters

GB200 and GB300 NVL72 racks are Grace-based: their compute trays are arm64
hosts, and so are many of the clusters running AI-factory services next to
them. An amd64-only image either cannot run there or has to be rebuilt by every
operator, which is the situation the planned image is meant to remove.

### Why it should be cheap

- CI already runs on arm64 runners (`linux-arm64-cpu16` in `tests.yml` and
  `release.yml`), so arm64 is the tested architecture today.
- The Dockerfile has no architecture-specific steps: the base images
  (`python:*-slim-trixie`, `ghcr.io/astral-sh/uv`) are multi-arch, and
  `service/service-app/uv.lock` carries both `manylinux ... x86_64` and
  `manylinux ... aarch64` wheels for its compiled dependencies.
- `docker buildx build --platform linux/amd64,linux/arm64` (BuildKit) builds
  both from one Dockerfile; native runners per architecture avoid emulation.

### What was checked (on an arm64 host, upstream `main` at 51c01f3, no changes)

- `linux/arm64`: every builder step and the runtime stage built, including
  both `uv sync` steps (38 then 2 packages installed). The final OCI export on
  that host was interrupted by a host-side buildx session timeout under memory
  pressure, not by a Dockerfile error.
- `linux/amd64` under QEMU user emulation (qemu 8.2.2) on a 64 KiB-page arm64
  host: the builder stage stopped at `apt-get update` with Debian trixie
  `OpenPGP signature verification failed ... No good signature`. A plain
  `debian:trixie-slim` `apt-get update` fails identically under the same
  emulator while `debian:bookworm-slim` succeeds, so this is the emulator on
  that host, not the Dockerfile. A native amd64 runner (or a current QEMU) is
  needed to confirm the amd64 leg end to end; this was not done.

### Not requested

No change to the Dockerfile; no request to support non-BuildKit builders.
