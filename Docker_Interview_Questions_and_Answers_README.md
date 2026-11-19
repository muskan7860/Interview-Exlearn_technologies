# Docker Interview Questions and Answers

## DevOps Engineer -- 4 Years Experience \| Technical Panel Preparation

**Source:** Uploaded *Docker Interview Questions.pdf* (3 pages).\
**Coverage:** All 54 questions from the questionnaire, in the original
topic order.\
**Answer level:** Practical DevOps engineering, production
troubleshooting, and solution-architect considerations.

> **How to use this guide:** The "Interview answer" sections are phrased
> so you can speak them in an interview. Treat implementation examples
> as patterns you understand and can explain; do not claim production
> ownership unless it reflects your actual work.

------------------------------------------------------------------------

# 1. Docker Basics

## Q1. What is Docker?

### Interview answer

Docker is a containerization platform used to package an application and
its runtime dependencies into a portable unit called a container image.
We build an image from a Dockerfile and run it as a container using the
Docker Engine or a compatible container runtime. This helps maintain
consistency across development, testing, and production environments.

A container shares the host operating-system kernel but runs in an
isolated process environment. Docker uses Linux kernel capabilities such
as namespaces for isolation and cgroups for resource control. In a
DevOps workflow, Docker images are commonly built in CI, scanned, pushed
to a registry such as Amazon ECR, and deployed to environments such as
Kubernetes.

### Technical explanation

-   **Dockerfile:** Instructions used to build an image.
-   **Image:** Immutable, layered template containing application files,
    runtime and dependencies.
-   **Container:** A running or stopped instance created from an image,
    with a writable container layer.
-   **Docker Engine:** Includes the Docker daemon (`dockerd`), API and
    CLI interaction.
-   **Registry:** Stores and distributes images.
-   **Container runtime:** The component responsible for creating and
    running containers. Modern Kubernetes commonly uses containerd
    through CRI; Docker Engine itself is not the Kubernetes runtime in
    current standard configurations.

Linux isolation uses namespaces (PID, network, mount, IPC, UTS, and user
namespaces where configured). cgroups account for and limit resources
such as CPU and memory. Images are commonly stored using a layered
filesystem such as OverlayFS.

### Enterprise workflow

Developer commits code → CI checks out source → Docker builds an image →
tests and vulnerability scans run → image is tagged with an immutable
version or commit SHA → image is pushed to a registry → deployment
platform pulls and runs that image. Promote the same tested image
between environments rather than rebuilding different artifacts.

### Architect perspective

Use small, trusted base images, non-root execution, image scanning,
resource limits, immutable tags/digests, registry access controls,
logging, and a patch/rebuild process. Containers improve packaging
consistency; they do not automatically make an application highly
available or secure.

------------------------------------------------------------------------

## Q2. What is containerization?

### Interview answer

Containerization is the method of packaging an application with the
libraries, runtime and configuration it needs, then running it in an
isolated process environment. Containers share the host kernel, so they
are generally lighter and start faster than full virtual machines. They
make application delivery more consistent across environments.

### Container vs virtual machine

  --------------------------------------------------------------------------
  Area                    Container               Virtual machine
  ----------------------- ----------------------- --------------------------
  Isolation               Process-level isolation Hardware virtualization
                          using kernel features   with a guest OS

  Kernel                  Shares host kernel      Has its own guest kernel

  Footprint               Usually smaller         Usually larger

  Startup                 Usually seconds or less Commonly longer

  Isolation boundary      Depends on              Generally stronger
                          kernel/runtime          boundary, depending on
                          configuration           hypervisor/configuration

  Best fit                Microservices, CI jobs, Different OS kernels,
                          scalable app workloads  legacy workloads, stronger
                                                  OS-level separation
  --------------------------------------------------------------------------

### Important point

A container is not simply a lightweight VM. Because containers share the
host kernel, kernel vulnerabilities and unsafe privileges can affect the
host. Use least privilege, user namespaces where suitable, seccomp,
AppArmor/SELinux, and patched hosts.

------------------------------------------------------------------------

## Q3. What is the difference between an image and a container?

### Interview answer

A Docker image is a read-only, layered template. A container is an
instance created from that image. When Docker starts a container, it
adds a writable layer on top of the image layers. Changes made inside
the container normally stay in that writable layer and are lost when the
container is removed, unless the data is stored in a volume or bind
mount.

  -----------------------------------------------------------------------
  Image                               Container
  ----------------------------------- -----------------------------------
  Build artifact/template             Runtime instance

  Read-only layers                    Image layers plus writable
                                      container layer

  Stored locally or in a registry     Created and managed by the runtime

  Identified by tags and content      Identified by container ID/name
  digest                              

  Created using `docker build`        Created/run using `docker create` /
                                      `docker run`
  -----------------------------------------------------------------------

### Commands

``` bash
docker build -t myapp:1.0 .
docker image ls
docker create --name myapp myapp:1.0
docker start myapp
docker ps -a
docker rm myapp
```

**Interview point:** Removing a container does not remove the image.
Removing a container also does not automatically remove a separately
managed named volume.

------------------------------------------------------------------------

## Q4. How do you build a Docker image? Explain the steps.

### Interview answer

First, I prepare a Dockerfile and the application source. I review the
build context and create a `.dockerignore` file so unnecessary files
such as `.git`, local dependencies, logs and secrets are not sent to the
builder. Then I run `docker build`, tag the image, test it locally, scan
it, and push it to the registry.

### Example

``` bash
# Build from the directory containing the Dockerfile
docker build -t myapp:1.0 .

# Build with an explicit Dockerfile and build argument
docker build -f Dockerfile --build-arg APP_ENV=production -t myapp:1.0 .

# Inspect and test
docker image inspect myapp:1.0
docker run --rm -p 8080:8080 myapp:1.0

# Tag and push
docker tag myapp:1.0 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:1.0
```

### Build sequence

1.  Docker reads the Dockerfile and build context.
2.  The builder processes instructions in order.
3.  Each instruction may create or reuse a cached layer.
4.  The resulting image is tagged.
5.  The image is tested and scanned before publication.

**Security:** Never pass passwords or API keys through Dockerfile `ARG`
or `ENV`. Use BuildKit secret mounts for build-time secrets.

------------------------------------------------------------------------

## Q5. How do you run a Docker container?

### Interview answer

I use `docker run`, which creates a container from an image and starts
it. I specify options such as detached mode, port mapping, environment
variables, volume mounts, restart policy and resource limits depending
on the application requirement.

``` bash
docker run -d \
  --name web \
  --publish 8080:80 \
  --restart unless-stopped \
  nginx:stable
```

-   `-d`: Run in background.
-   `--name`: Assign a readable container name.
-   `-p 8080:80`: Map host port 8080 to container port 80.
-   `--restart unless-stopped`: Restart after daemon/host restart unless
    explicitly stopped.

Useful checks:

``` bash
docker ps
docker logs --tail 100 web
docker inspect web
docker stats web
```

Do not expose a port publicly unless required. For local-only access,
use `-p 127.0.0.1:8080:80`.

------------------------------------------------------------------------

## Q6. How do you create a Docker container?

### Interview answer

`docker create` creates a container but does not start it.
`docker start` starts that already-created container. `docker run` is
effectively a convenient create-and-start operation.

``` bash
docker create --name test-nginx -p 8080:80 nginx:stable
docker ps -a
docker start test-nginx
docker ps
```

Use `create` when I want to prepare the container configuration first
and start it later. Use `run` for the common one-step workflow.

------------------------------------------------------------------------

## Q7. What are dangling objects in Docker?

### Interview answer

Dangling images are images that have no tag and are not referenced by a
named image tag, often left behind after rebuilding an image. Docker can
also accumulate stopped containers, unused networks, build cache and
unused volumes. I inspect what is consuming space before pruning
anything.

``` bash
# Dangling images
docker images --filter dangling=true

# Remove dangling images only
docker image prune

# Review disk usage
docker system df
docker system df -v
```

`docker system prune` removes unused containers, networks, dangling
images and build cache. Adding `-a` removes all unused images, not only
dangling ones. Adding `--volumes` can remove unused anonymous volumes
and may cause data loss. Always review impact and protect persistent
data before cleanup.

------------------------------------------------------------------------

# 2. Docker Images & Optimization

## Q8. Explain how multi-stage Docker builds help with image optimization.

### Interview answer

A multi-stage build uses multiple `FROM` statements in one Dockerfile. I
use an earlier stage to install build tools, compile code or create
production assets, and a final stage containing only the runtime and
required artifacts. This reduces the final image size and avoids
shipping compilers, package managers and temporary build files
unnecessarily.

### Example

``` dockerfile
# Build stage
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /out/server ./cmd/server

# Runtime stage
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Benefits

-   Smaller attack surface and fewer packages to patch.
-   Reduced image transfer and pull time.
-   Build dependencies are excluded from runtime.
-   Build and runtime environments can be independently selected.

Check that the final stage contains required CA certificates, timezone
data, shared libraries and runtime assets. A smaller image is not
automatically secure; scan and maintain it.

------------------------------------------------------------------------

## Q9. How do you reduce the size of a Docker image?

### Interview answer

I start by measuring image size and identifying which layers contribute
most. Then I choose a suitable minimal trusted base image, use
multi-stage builds, install only runtime dependencies, remove
package-manager caches in the same build layer, use `.dockerignore`, and
avoid copying unnecessary source files or local artifacts.

### Practical checklist

1.  Use a runtime-specific base image, not a full build image.
2.  Use multi-stage builds.
3.  Pin base image versions or digests.
4.  Copy only required files.
5.  Exclude `.git`, `node_modules` from host, test output, logs and
    local secrets through `.dockerignore`.
6.  Combine package installation and cache cleanup in the same `RUN`
    instruction.
7.  Avoid unnecessary tools and duplicate dependencies.
8.  Use BuildKit cache mounts for faster builds without baking caches
    into final layers.

Example:

``` dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ ./app/
CMD ["python", "-m", "app"]
```

Do not remove files in a later layer expecting to reclaim earlier layer
bytes. If a large file was added in a previous layer, rebuilding with a
corrected instruction is usually necessary.

------------------------------------------------------------------------

## Q10. Your image size is way too large. What steps will you take to optimize it?

### Interview answer

I would not immediately change the base image. First I would establish a
baseline, inspect the image layers and build context, and identify
whether the size comes from build tools, dependencies, copied source,
caches or generated artifacts. Then I would optimize one cause at a time
and compare image size, build time and application behavior.

### Investigation

``` bash
docker image ls
docker image inspect myapp:latest
docker history --no-trunc myapp:latest
docker system df -v
```

For deeper layer analysis, use a trusted tool such as `dive`.

### Remediation sequence

-   Review `.dockerignore` and build context.
-   Find large layers with image history/layer analysis.
-   Separate build and runtime stages.
-   Install only production dependencies.
-   Remove package caches in the same `RUN` layer.
-   Avoid copying local dependency directories.
-   Use a slim/distroless base only after compatibility testing.
-   Rebuild, test, scan and compare.

### Architect perspective

Balance image size against compatibility, patchability, observability
and debugging requirements. An extremely minimal image may make
shell-based debugging impossible; use external ephemeral debugging tools
or a separate debug image rather than adding unnecessary utilities to
production.

------------------------------------------------------------------------

## Q11. How do multistage Docker builds improve security and reduce image size?

### Interview answer

They separate the build environment from the runtime environment. Build
tools such as compilers, package managers and source-control utilities
remain in the builder stage, while the final stage receives only the
compiled binary or production artifacts. That reduces the number of
packages and binaries available to an attacker and reduces the amount of
software that needs patching.

### Important considerations

-   Use trusted, maintained base images.
-   Copy only explicit artifacts from the builder.
-   Run as a non-root user.
-   Do not copy secrets into any stage or layer.
-   Scan the final image, not only the builder.
-   Pin dependencies and base image digests where reproducibility is
    required.

Multi-stage builds reduce attack surface but do not replace
vulnerability scanning, runtime hardening or access controls.

------------------------------------------------------------------------

## Q12. Are Docker image layers read-only or read-write, and how does that work?

### Interview answer

The layers that make up an image are read-only. When a container is
created, the container runtime adds a separate writable layer on top of
those image layers. The container sees a unified filesystem. If a
process modifies a file inherited from an image layer, a copy-on-write
mechanism may copy that file into the writable layer before modifying
it.

### Key points

-   Image layers are immutable/read-only.
-   Each container gets its own writable layer.
-   Containers created from the same image can share the underlying
    read-only layers.
-   Writable-layer data disappears when the container is removed.
-   Persistent application data belongs in volumes or bind mounts.

The exact filesystem implementation depends on the storage driver,
commonly OverlayFS on Linux.

------------------------------------------------------------------------

## Q13. Explain image layering in Docker.

### Interview answer

Docker images are composed of ordered filesystem layers. Dockerfile
instructions such as `RUN`, `COPY` and `ADD` can create layers. These
layers are content-addressed and reusable, so multiple images can share
common base layers. This reduces storage and network transfer and
enables build caching.

### Example

``` dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "server.js"]
```

A change to application source generally allows Docker to reuse earlier
dependency layers if the package files have not changed. This is why
dependency manifests are copied and installed before frequently changing
source code.

**Important:** Layer caching is affected by instruction contents, files
copied and build arguments. Keep stable, expensive steps early and
frequently changing files later.

------------------------------------------------------------------------

# 3. Dockerfile

## Q14. What is a Dockerfile?

### Interview answer

A Dockerfile is a text file containing declarative instructions used by
the Docker builder to create an image. It defines the base image,
working directory, dependencies, files to copy, runtime user, exposed
port metadata and startup command. It makes image creation repeatable
and suitable for version control and CI/CD.

Common instructions: - `FROM`: Selects the base image. - `WORKDIR`: Sets
the working directory. - `COPY`: Copies files from build context or a
stage. - `RUN`: Executes commands during build. - `ENV`: Sets runtime
environment defaults. - `ARG`: Defines build-time variables. - `EXPOSE`:
Documents the intended listening port. - `USER`: Selects the runtime
user. - `ENTRYPOINT` / `CMD`: Define container startup behavior.

------------------------------------------------------------------------

## Q15. Could you please write a Dockerfile?

### Interview answer

The Dockerfile depends on the application stack. For a simple Python
API, I would use a maintained slim base image, install pinned
dependencies, copy only the application code, create a non-root user and
use exec-form startup syntax.

``` dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=10001:10001 app/ ./app/

USER 10001:10001

EXPOSE 8000

CMD ["python", "-m", "app"]
```

Build and run:

``` bash
docker build -t sample-api:1.0 .
docker run --rm -p 8000:8000 sample-api:1.0
```

For a production application, I would also pin dependencies, add health
checks where appropriate, scan the image, use a read-only root
filesystem if compatible, and configure resource limits in the
deployment platform.

------------------------------------------------------------------------

## Q16. Best practices for production-ready Dockerfiles?

### Interview answer

I focus on reproducibility, security, maintainability and efficient
builds. I use a trusted and maintained base image, pin dependencies, use
multi-stage builds where appropriate, keep the build context small, run
as non-root, avoid secrets in image layers, and use exec-form
`ENTRYPOINT` or `CMD` so the application receives signals correctly.

### Checklist

-   Use a trusted, maintained base image; pin version and consider
    digest pinning.
-   Use `.dockerignore`.
-   Use multi-stage builds.
-   Install only required packages.
-   Run as a non-root user.
-   Do not embed credentials or private keys.
-   Prefer `COPY` over `ADD` for ordinary file copying.
-   Use exec-form startup commands.
-   Keep one primary service/process per container in most application
    designs.
-   Handle SIGTERM and shut down gracefully.
-   Add a `HEALTHCHECK` only when it provides meaningful health
    information and is supported by the runtime/orchestrator.
-   Scan images and generate an SBOM in the build pipeline.
-   Rebuild regularly to incorporate security updates.

Example `.dockerignore`:

``` text
.git
.env
*.log
__pycache__/
.pytest_cache/
node_modules/
coverage/
dist/
```

------------------------------------------------------------------------

## Q17. How do you run a Dockerfile? Please give the commands.

### Interview answer

A Dockerfile is not executed directly. I build an image from it, then
create and start a container from that image.

``` bash
# Navigate to the directory containing Dockerfile
cd /path/to/application

# Build
docker build -t myapp:1.0 .

# Run
docker run -d --name myapp -p 8080:8080 myapp:1.0

# Verify
docker ps
docker logs myapp
docker inspect myapp

# Stop and remove
docker stop myapp
docker rm myapp
```

The final `.` is the build context. It tells Docker which directory's
files are available to `COPY` and `ADD`.

------------------------------------------------------------------------

## Q18. Difference between ENTRYPOINT and CMD.

### Interview answer

Both define what runs when a container starts. `ENTRYPOINT` defines the
main executable, while `CMD` provides default arguments or a default
command. If both are in exec form, Docker combines them as `ENTRYPOINT`
followed by `CMD`. A command supplied after the image name to
`docker run` replaces the default `CMD` arguments; `--entrypoint` can
override the entrypoint.

  -----------------------------------------------------------------------
  ENTRYPOINT                          CMD
  ----------------------------------- -----------------------------------
  Defines the main executable         Defines default command or
                                      arguments

  Usually remains fixed               Commonly overridden at runtime

  Useful for application/container    Useful for default parameters
  behavior                            
  -----------------------------------------------------------------------

Example:

``` dockerfile
ENTRYPOINT ["python", "-m", "http.server"]
CMD ["8000"]
```

Run with default port:

``` bash
docker run --rm myserver
```

Override CMD argument:

``` bash
docker run --rm myserver 9000
```

Prefer exec form (`["executable", "arg"]`) over shell form for
application processes. Exec form allows the process to receive signals
directly as PID 1. If a shell wrapper is required, use `exec` in the
script so it replaces the shell process.

------------------------------------------------------------------------

## Q19. What is WORKDIR in a Dockerfile? What does it do?

### Interview answer

`WORKDIR` sets the working directory for subsequent Dockerfile
instructions such as `RUN`, `COPY`, `ADD`, `CMD` and `ENTRYPOINT` where
relevant. If the directory does not exist, Docker creates it. It avoids
repeatedly writing absolute paths.

``` dockerfile
WORKDIR /app
COPY . .
RUN npm ci
```

This is clearer and more maintainable than using repeated
`RUN cd /app && ...` commands. Each `RUN` instruction runs in its own
build process, so `RUN cd /app` does not persist to the next
instruction.

------------------------------------------------------------------------

## Q20. How do you define a startup command in Docker?

### Interview answer

I define the startup behavior using `CMD`, `ENTRYPOINT`, or both. For a
service container, I normally use exec-form syntax so the application
becomes the main process and handles termination signals correctly.

``` dockerfile
ENTRYPOINT ["./start.sh"]
CMD ["--port", "8080"]
```

In `start.sh`, use:

``` sh
#!/bin/sh
set -eu
exec ./application "$@"
```

`EXPOSE` does not start a service or publish a port; it is image
metadata. Port publishing is configured at runtime, for example with
`docker run -p 8080:8080`.

------------------------------------------------------------------------

## Q21. In a Dockerfile, what's the practical difference between COPY and ADD? When would you prefer one over the other?

### Interview answer

`COPY` copies files and directories from the build context, or from
another build stage, into the image. `ADD` has additional behavior: it
can automatically extract supported local tar archives and can retrieve
remote URLs in supported builder contexts. For normal file copying, I
prefer `COPY` because its behavior is explicit and predictable.

``` dockerfile
COPY requirements.txt /app/
COPY --from=builder /out/app /usr/local/bin/app
```

Use `ADD` only when its additional semantics are deliberately needed,
such as local archive extraction. For downloading remote artifacts,
prefer an explicit download command with checksum/signature verification
rather than an opaque remote `ADD`.

------------------------------------------------------------------------

## Q22. Write a Dockerfile for a Node.js application with multi-stage builds.

### Example

``` dockerfile
# Dependencies stage
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Build stage
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production dependencies stage
FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Runtime stage
FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app

COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./package.json

RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Explanation

-   `deps` installs dependencies using the lock file.
-   `build` compiles/transforms application code.
-   `prod-deps` installs production-only dependencies.
-   `runtime` contains only production dependencies and compiled output.
-   The final process runs as a non-root user.

Adjust paths and build commands to the actual project. If native Node
modules are used, ensure builder and runtime environments have
compatible libc and shared libraries. Alpine uses musl, which can create
compatibility issues for packages expecting glibc.

------------------------------------------------------------------------

## Q23. Write a Dockerfile for a secure Python/Node.js API with multistage builds and best practices.

### Interview answer

I would first clarify whether the API is Python or Node.js; these are
different runtimes and should generally have separate Dockerfiles. The
common design is a builder/dependency stage followed by a minimal
runtime stage, with pinned dependencies, non-root execution, no embedded
secrets and a controlled startup command.

### Python API example

``` dockerfile
# Builder stage
FROM python:3.12-slim AS builder
ENV VIRTUAL_ENV=/opt/venv
RUN python -m venv "$VIRTUAL_ENV"
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
WORKDIR /build

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.12-slim AS runtime
ENV VIRTUAL_ENV=/opt/venv \
    PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY --chown=10001:10001 app/ ./app/

USER 10001:10001
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Node.js API alternative

``` dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Security notes

-   Lock and review dependency versions.
-   Use maintained base images and scan the final image.
-   Keep secrets outside the image; inject them at runtime from a secret
    manager.
-   Run non-root and drop unnecessary Linux capabilities.
-   Use read-only root filesystem where compatible.
-   Set CPU/memory limits in the orchestrator.
-   Do not assume `EXPOSE` publishes a port.
-   Add health/readiness checks at the orchestration layer where
    appropriate.

------------------------------------------------------------------------

# 4. Containers -- Troubleshooting & Scenarios

## Q24. A container keeps restarting --- how do you troubleshoot?

### Interview answer

I start by checking the container status, exit code, restart count and
logs. Then I inspect the container configuration, entrypoint,
environment variables, mounts, health check and resource limits. I
correlate the failure time with host or application events. I do not
immediately disable the restart policy because it may be protecting
service availability.

### Investigation

``` bash
docker ps -a
docker inspect --format='{{.State.Status}} {{.State.ExitCode}} {{.State.OOMKilled}} {{.RestartCount}}' app
docker logs --timestamps --tail 200 app
docker inspect app
docker events --since 30m
docker stats --no-stream app
```

### Common causes

-   Application exits after startup or has an invalid command.
-   Missing environment variable or configuration.
-   Database/dependency unavailable.
-   Permission problem on mounted files.
-   Failed health check (depending on orchestrator behavior).
-   Out-of-memory kill.
-   Incorrect restart policy or external automation repeatedly
    restarting it.

### Resolution

Identify the actual exit reason, correct configuration or application
behavior, validate dependency readiness, tune resource limits based on
measurements, and confirm recovery. A Docker health check by itself does
not normally restart an unhealthy container; restart policies respond to
process exit. Orchestrators may take health status into account
differently.

------------------------------------------------------------------------

## Q25. A container has crashed or exited suddenly --- how would you figure out what went wrong?

### Interview answer

I preserve evidence first. I inspect the exit code and state, read logs
around the failure, inspect OOM status and recent Docker events, and
check host-level memory, disk and kernel logs. If the container has
already exited, I avoid removing it until I have collected the required
evidence.

``` bash
docker ps -a
docker inspect app
docker logs --timestamps app
docker inspect --format='Exit={{.State.ExitCode}} OOM={{.State.OOMKilled}} Error={{.State.Error}}' app
docker events --since 1h
free -h
df -h
sudo journalctl -u docker --since "1 hour ago"
sudo dmesg -T | tail -100
```

### Exit code clues

-   `0`: Process exited successfully; may be unexpected for a
    long-running service.
-   `1`: Generic application error.
-   `137`: Commonly SIGKILL (`128 + 9`); investigate OOM and
    manual/external kill.
-   `143`: Commonly SIGTERM (`128 + 15`); investigate graceful
    stop/deployment.
-   `139`: Commonly segmentation fault (`128 + 11`).

Exit codes are clues, not definitive root-cause proof. Correlate with
logs, state and host evidence.

------------------------------------------------------------------------

## Q26. What steps would you take if a container crashes immediately after starting?

### Interview answer

I would check whether the process starts and exits because of an invalid
command, missing configuration, permissions, missing runtime
dependencies or a failed external dependency. I would inspect logs and
the exact startup configuration before changing the image.

``` bash
docker ps -a
docker logs --tail 200 app
docker inspect app
docker image inspect myapp:1.0
```

For a diagnostic shell, if the image contains a shell:

``` bash
docker run --rm -it --entrypoint sh myapp:1.0
```

For a minimal/distroless image without a shell, use application logs,
debug builds or approved ephemeral debugging methods. Verify: 1.
Entrypoint and CMD. 2. Required environment variables and config files.
3. File permissions and user ID. 4. Port binding and runtime
dependencies. 5. Mounted paths. 6. Database/API connectivity. 7. Memory
limit and OOM state.

Do not keep changing multiple settings simultaneously; isolate one
hypothesis at a time.

------------------------------------------------------------------------

## Q27. App works locally but not inside a container --- what will you verify first?

### Interview answer

I compare the local and container environments systematically. I verify
the application binds to the correct interface, dependencies are
installed, configuration is present, file paths and permissions are
correct, and the required ports are published. A common issue is an
application listening only on `127.0.0.1` inside the container instead
of `0.0.0.0`.

### Checklist and commands

``` bash
docker logs app
docker inspect app
docker exec -it app sh
docker port app
docker inspect --format='{{json .NetworkSettings.Networks}}' app
```

Verify: - Bind address: use `0.0.0.0` for a service that must be reached
through the container network. - Port: application listening port vs
host published port. - Dependencies: OS libraries, language packages and
correct versions. - Environment: variables, config files and secrets. -
Filesystem: working directory, case-sensitive paths and permissions. -
Networking: DNS, route, firewall and network membership. - Architecture:
image CPU architecture matches the host/runtime. - Startup: process
stays in foreground and handles signals.

------------------------------------------------------------------------

## Q28. Container can't access internet -- how do you debug?

### Interview answer

I check the problem layer by layer: container network attachment, IP
address and route, DNS resolution, host forwarding/firewall, proxy
configuration and destination restrictions. I compare access by IP with
access by hostname to distinguish DNS from general connectivity.

``` bash
docker network ls
docker inspect app
docker exec app cat /etc/resolv.conf
docker exec app ip route
docker exec app sh -c 'getent hosts example.com'
docker exec app sh -c 'wget -S -O- https://example.com'
```

Some diagnostic commands may not exist in minimal images. Use a
temporary diagnostic container on the same network if needed.

### Root causes

-   Container attached to an unexpected network.
-   DNS resolver issue.
-   Host IP forwarding or firewall rules.
-   Corporate proxy not configured.
-   Egress blocked by security policy.
-   MTU or routing issue.
-   Destination endpoint unavailable.

### Resolution

Correct the network, DNS, proxy or host firewall configuration according
to the identified cause. Avoid hardcoding public DNS servers without
understanding corporate DNS and security requirements. Do not use
`--network host` as a default workaround.

------------------------------------------------------------------------

## Q29. Docker build fails only on CI with "no space left on device" --- why?

### Interview answer

CI runners often have limited ephemeral disk. Docker builds can consume
space through image layers, build cache, downloaded dependencies,
intermediate stages, containers, logs and the build context. I check
both filesystem capacity and inode usage because either can cause this
error.

``` bash
df -h
df -i
docker system df -v
docker image ls
docker ps -a
docker builder du
```

### Resolution approach

1.  Identify which filesystem is full (`/`, Docker data root, workspace
    or temporary filesystem).
2.  Identify large images, build cache and workspaces.
3.  Remove only resources confirmed safe to delete.
4.  Add CI cleanup between isolated jobs where appropriate.
5.  Use BuildKit cache with bounded retention.
6.  Increase runner disk if the workload legitimately requires it.
7.  Keep build context small with `.dockerignore`.

**Destructive warning:** `docker system prune -a --volumes` can delete
unused images, cache and volumes. Do not run it blindly on shared or
persistent runners. Prefer targeted cleanup and ephemeral runners.

------------------------------------------------------------------------

## Q30. How do you troubleshoot high CPU usage in a container?

### Interview answer

First I identify which container is consuming CPU and whether the usage
is sustained or a short spike. Then I inspect process-level activity,
application metrics and logs, traffic volume, recent releases and CPU
limits/throttling. I distinguish high application demand from a CPU
limit that is too restrictive.

``` bash
docker stats
docker stats --no-stream app
docker top app
docker inspect app
```

Inside the container, if tools are available:

``` bash
docker exec app ps aux
```

On Linux host:

``` bash
top
pidstat -p ALL 1
```

### Investigation

-   Is CPU high on one container or all containers?
-   Did traffic, job volume or error/retry rate increase?
-   Is there an infinite loop, expensive query, thread contention or
    excessive logging?
-   Is the container CPU quota causing throttling?
-   Did a new release change performance?

### Resolution

Fix the application bottleneck, tune concurrency, optimize queries,
scale horizontally if workload is parallelizable, or adjust CPU limits
based on measured demand. Monitor CPU utilization and throttling
together; simply increasing limits can hide an application defect.

------------------------------------------------------------------------

# 5. Docker Volumes & Storage

## Q31. How do you persist data across container restarts?

### Interview answer

I use Docker volumes or bind mounts rather than relying on the container
writable layer. A named volume is managed by Docker and is a good
default for persistent container data. A bind mount maps a specific host
path into the container and is useful when the host path itself must be
controlled, such as local development.

### Named volume example

``` bash
docker volume create mysql-data

docker run -d \
  --name mysql \
  -e MYSQL_DATABASE=appdb \
  -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql-root-password \
  --mount source=mysql-data,target=/var/lib/mysql \
  mysql:8.4
```

This example assumes the secret file is provided through an appropriate
secret mechanism; do not place a real password directly in shell history
or source control.

### Verify

``` bash
docker volume ls
docker volume inspect mysql-data
```

A volume survives container removal unless explicitly removed. Backups,
restore testing, encryption and storage monitoring are still required.

------------------------------------------------------------------------

## Q32. What's the difference between a Docker volume and bind mount?

  -----------------------------------------------------------------------
  Feature                 Docker volume           Bind mount
  ----------------------- ----------------------- -----------------------
  Source                  Managed by Docker       Explicit host
                                                  filesystem path

  Location                Docker-managed storage  Chosen host path
                          area                    

  Portability             Easier to declare and   Depends on host
                          manage                  directory structure

  Typical use             Persistent              Development source
                          application/database    code, host-managed
                          data                    files

  Lifecycle               Managed through volume  Host directory
                          commands                lifecycle is external

  Permissions             Still require correct   Host/container UID/GID
                          ownership/permissions   mapping is important
  -----------------------------------------------------------------------

### Examples

``` bash
# Named volume
docker run -d --mount source=dbdata,target=/var/lib/mysql mysql:8.4

# Bind mount
docker run --rm -it \
  --mount type=bind,src="$PWD",dst=/workspace \
  node:22-alpine sh
```

Use `--mount` for explicit syntax and clearer validation. Bind mounts
can expose sensitive host files if configured carelessly. Avoid mounting
the Docker socket into untrusted containers because it can effectively
grant control over the host's Docker daemon.

------------------------------------------------------------------------

# 6. Environment Variables & Secrets

## Q33. How do you pass environment variables during docker run?

### Interview answer

I use `-e` for individual variables or `--env-file` for a file
containing environment variables. These are useful for configuration
such as environment name, feature flags and service endpoints. Sensitive
values should be handled through a secret manager or runtime secret
mechanism rather than plain command-line arguments.

``` bash
docker run -d \
  --name api \
  -e APP_ENV=production \
  -e LOG_LEVEL=info \
  -p 8080:8080 \
  myapi:1.0
```

Using an env file:

``` bash
docker run --env-file ./app.env -d --name api myapi:1.0
```

Keep `.env` out of Git and the Docker build context. Environment
variables can be visible to users with sufficient host/container
inspection permissions and may appear in diagnostic output.

------------------------------------------------------------------------

## Q34. How do you handle environment variables securely?

### Interview answer

I separate non-sensitive configuration from secrets. Non-sensitive
settings can be passed as environment variables. Credentials, tokens and
private keys should come from a secrets manager or orchestrator secret
mechanism with restricted access, rotation and audit logging.

### Recommended approach

-   Store secrets in AWS Secrets Manager or Systems Manager Parameter
    Store (with appropriate encryption/access controls), or an approved
    enterprise vault.
-   Grant workloads least-privilege IAM permissions.
-   Retrieve secrets at runtime or inject them through a controlled
    secret mechanism.
-   Rotate credentials and avoid long-lived static credentials.
-   Prevent secrets from appearing in logs, command history, CI output
    and crash dumps.
-   Restrict who can inspect containers and deployment definitions.

Environment variables are convenient but are not a secret-management
system by themselves. A mounted secret file or workload identity may be
preferable depending on the platform.

------------------------------------------------------------------------

## Q35. Best practices for handling secrets in Docker.

### Interview answer

I never bake secrets into a Docker image or commit them to source
control. I use a secrets manager and inject secrets at runtime with
narrowly scoped access. For build-time secrets, I use BuildKit secret
mounts so the secret is not persisted in an image layer.

### Build-time secret example

``` dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.21
RUN --mount=type=secret,id=repo_token \
    TOKEN="$(cat /run/secrets/repo_token)" && \
    ./download-private-dependency.sh "$TOKEN"
```

Build:

``` bash
docker build --secret id=repo_token,src=./repo_token.txt -t app:1.0 .
```

The script should avoid printing the token. Delete the local secret file
securely according to organizational policy after use.

### Avoid

-   `ENV PASSWORD=...`
-   `ARG TOKEN=...` for sensitive values
-   Copying `.env` into the image
-   Printing secrets in pipeline logs
-   Putting secrets in image labels or build arguments

BuildKit secret mounts protect build-time secret persistence; runtime
secrets still need a runtime secret-management design.

------------------------------------------------------------------------

## Q36. How would you handle secrets in a Docker container for a PHP application connecting to MySQL?

### Interview answer

I would keep the MySQL password outside the PHP image and inject it at
runtime from a secrets manager. The PHP application would read the
secret from a protected file or approved environment-variable mechanism.
The database would be on a private Docker network, and only the PHP
service would need database connectivity.

### Compose pattern using a secret file

``` yaml
services:
  php:
    build: .
    environment:
      DB_HOST: db
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    networks:
      - backend
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: appdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
    secrets:
      - db_password
      - mysql_root_password
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root --password=\"$$(cat /run/secrets/mysql_root_password)\""]
      interval: 10s
      timeout: 5s
      retries: 10

secrets:
  db_password:
    file: ./secrets/db_password.txt
  mysql_root_password:
    file: ./secrets/mysql_root_password.txt

volumes:
  mysql_data:

networks:
  backend:
```

### Important caveats

-   This is a local Compose file-secret pattern. Local Compose
    file-backed secrets are mounted as files; they are not equivalent to
    encrypted Swarm secrets or a cloud secrets manager.
-   Protect local secret files with restrictive permissions and exclude
    them from Git.
-   Confirm the selected MySQL image supports the `_FILE` variables
    used.
-   Use a dedicated least-privilege database user for the application.
-   In AWS, prefer retrieving the credential from Secrets Manager with
    workload IAM permissions and a controlled rotation strategy.
-   Do not publish the database port to the host unless there is a
    specific requirement.

------------------------------------------------------------------------

# 7. Docker Compose

## Q37. What is the use of Docker Compose?

### Interview answer

Docker Compose is a tool for defining and running multi-container
applications using a YAML file. I can define services, networks,
volumes, environment configuration, health checks and dependencies in
one place. It is particularly useful for local development, integration
testing and small single-host deployments.

Example application stack: frontend, API and database. Compose makes the
setup repeatable with `docker compose up` and simplifies lifecycle
operations.

Compose is not a replacement for a production orchestrator such as
Kubernetes when requirements include multi-node scheduling, advanced
self-healing, autoscaling and cluster-level operations.

------------------------------------------------------------------------

## Q38. How do you use Docker Compose in a local development setup?

### Interview answer

I define the application and its dependencies in `compose.yaml`, mount
source code for development when appropriate, use named volumes for
database data, configure a private network and load local configuration
from an environment file. Then I start the stack and inspect service
logs.

``` bash
docker compose up -d --build
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose down
```

`docker compose down` removes the Compose containers and networks but
preserves named volumes by default. `docker compose down -v` removes
named volumes and can delete database data, so use it only when that
data is disposable.

------------------------------------------------------------------------

## Q39. Write a Docker Compose file and explain it line by line.

### Example: API + PostgreSQL

``` yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    image: sample-api:1.0
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app_net
    restart: unless-stopped

  db:
    image: postgres:17
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  postgres_data:

networks:
  app_net:
    driver: bridge
```

### Explanation

-   `services`: Declares containers that make up the application.
-   `api.build`: Builds the API image from the current directory and
    specified Dockerfile.
-   `image`: Assigns a local image name/tag.
-   `environment`: Supplies non-secret configuration.
-   `secrets`: Mounts the secret as a file at the path managed by
    Compose.
-   `ports`: Publishes API port only on host loopback, not every host
    interface.
-   `depends_on` with `service_healthy`: Waits for the database health
    check to pass before starting the API. The API should still
    implement connection retries because dependencies can fail later.
-   `networks`: Connects both services to a private bridge network.
-   `volumes`: Persists database data outside the container writable
    layer.
-   `healthcheck`: Checks database readiness.
-   `restart`: Defines restart behavior after process exit.
-   `secrets.file`: Reads a local file; protect it and exclude it from
    source control.
-   `volumes.postgres_data`: Declares a named persistent volume.

Start and validate:

``` bash
docker compose config
docker compose up -d --build
docker compose ps
docker compose logs -f
```

This is an illustrative local Compose setup. Production deployment needs
managed secret delivery, backup/restore, monitoring, patching, network
controls and a suitable orchestrator/managed database decision.

------------------------------------------------------------------------

## Q40. How do you manage networking between multiple Docker containers?

### Interview answer

I create a user-defined bridge network and attach the related containers
to it. Containers on the same user-defined bridge network can
communicate using container names or network aliases through Docker's
embedded DNS. I normally keep internal services, such as databases, off
published host ports and publish only the entry-point service.

``` bash
docker network create app-net

docker run -d --name db --network app-net postgres:17
docker run -d --name api --network app-net myapi:1.0
```

The API can use `db` as its database hostname, assuming the application
and database port/configuration are correct.

For Compose, define a network and attach services to it. Use separate
networks to isolate frontend, application and data tiers where
appropriate.

------------------------------------------------------------------------

# 8. Docker Networking

## Q41. What is a Docker network?

### Interview answer

A Docker network provides connectivity between containers and, depending
on the driver and configuration, between containers, the host and
external networks. It provides network isolation and service-to-service
communication. I use user-defined networks to control which containers
can communicate and to use Docker's built-in name resolution.

Important elements include network drivers, IP address allocation,
routes, DNS, port publishing and host firewall/NAT behavior.

------------------------------------------------------------------------

## Q42. What is the default Docker network?

### Interview answer

When Docker Engine is installed, it normally creates a default bridge
network named `bridge`. If I run a container without specifying a
network, it is attached to this default bridge.

However, for application stacks I prefer a user-defined bridge network
because it provides automatic DNS resolution between containers by name,
better isolation and easier network management.

``` bash
docker network ls
docker network inspect bridge
docker run -d --name web nginx:stable
```

Do not confuse the default `bridge` network with a user-created bridge
network. User-defined bridge networks provide better service discovery.

------------------------------------------------------------------------

## Q43. How do you create a Docker network?

### Interview answer

I use `docker network create`, specifying a driver and, if required, a
subnet or other options.

``` bash
docker network create --driver bridge app-net
docker network ls
docker network inspect app-net
```

Attach a new container:

``` bash
docker run -d --name api --network app-net myapi:1.0
```

Connect an existing container:

``` bash
docker network connect app-net existing-container
```

Disconnect:

``` bash
docker network disconnect app-net existing-container
```

Plan custom subnets carefully to avoid overlap with corporate, VPN,
cloud VPC and Kubernetes CIDRs.

------------------------------------------------------------------------

## Q44. Docker networking types.

  ------------------------------------------------------------------------
  Driver                  Purpose                 Typical use
  ----------------------- ----------------------- ------------------------
  `bridge`                Connects containers on  Default choice for many
                          one Docker host         single-host app stacks

  `host`                  Shares host network     Special
                          namespace on supported  performance/networking
                          Linux configurations    requirements; reduced
                                                  isolation

  `none`                  Disables normal         Isolated batch or
                          container networking    security-sensitive
                                                  workloads

  `overlay`               Connects services       Multi-host Swarm
                          across multiple Docker  networking
                          hosts in Swarm          

  `macvlan`               Gives containers MAC    Legacy/network
                          addresses on a physical appliances requiring L2
                          network                 presence

  `ipvlan`                Provides L2/L3          Specific
                          connectivity with       datacenter/network
                          different MAC handling  integration needs
  ------------------------------------------------------------------------

### Interview explanation

For most single-host development or Compose applications, I use a
user-defined bridge network. For multi-host container orchestration, I
use the networking model provided by the orchestrator. Host networking
removes network namespace isolation and should be used only when
justified. Overlay networking is associated with Docker Swarm;
Kubernetes uses its CNI-based pod networking model.

------------------------------------------------------------------------

## Q45. DB connection --- how do you establish the connection?

### Interview answer

I put the application and database on the same private Docker network.
The application connects to the database using the database container's
service name as the hostname and the database's internal listening port.
I do not use `localhost` from one container to reach another container;
`localhost` refers to the current container itself.

### Example

``` bash
docker network create backend

docker run -d --name db --network backend \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:17

docker run -d --name api --network backend \
  -e DB_HOST=db \
  -e DB_PORT=5432 \
  myapi:1.0
```

This abbreviated example assumes the database secret is mounted
correctly; it is not a complete secret setup.

### Troubleshooting

``` bash
docker network inspect backend
docker inspect api
docker logs db
docker logs api
docker exec api getent hosts db
```

Verify DNS, network membership, DB readiness, credentials, database
grants, TLS requirements, listening address and firewall rules.
`depends_on` controls startup order in Compose but does not replace
application-level retry logic.

------------------------------------------------------------------------

# 9. Registry & Image Management

## Q46. Which container registry do you use for storing Docker images?

### Interview answer

For an AWS-based environment, Amazon Elastic Container Registry (ECR) is
a common choice. It is a managed private registry integrated with IAM,
AWS networking and AWS security services. Other organizations may use
Docker Hub, GitHub Container Registry, GitLab Container Registry, Azure
Container Registry or an enterprise registry such as Harbor.

I would choose based on cloud platform, identity integration,
vulnerability scanning, replication, retention, availability, network
access and cost.

------------------------------------------------------------------------

## Q47. What services do you use for storing Docker images?

### Interview answer

Common options include: - **Amazon ECR:** Managed private/public image
registry in AWS. - **Docker Hub:** Public and private repositories. -
**GitHub Container Registry (GHCR):** Registry integrated with GitHub
workflows. - **GitLab Container Registry:** Registry integrated with
GitLab CI/CD. - **Azure Container Registry (ACR):** Managed registry in
Azure. - **Google Artifact Registry:** Managed artifact and container
registry in Google Cloud. - **Harbor:** Self-hosted enterprise registry
with policy and security features.

For an AWS workload, ECR is often a natural option because IAM policies
control access and workloads can pull images using AWS identity
mechanisms.

------------------------------------------------------------------------

## Q48. How do you manage and push Docker images, especially with multiple services?

### Interview answer

I use a consistent naming and tagging strategy for each microservice.
Each service has its own build context and image, and CI builds only the
services affected by a change where practical. I tag images with an
immutable identifier such as the Git commit SHA and may also publish a
human-readable release tag. After testing and scanning, the pipeline
pushes the image to the registry and deploys the exact image digest.

### Example

``` bash
AWS_REGION=ap-south-1
ACCOUNT_ID=123456789012
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"

docker build -t "$REGISTRY/orders:abc1234" ./services/orders
docker push "$REGISTRY/orders:abc1234"
```

For multiple services:

``` text
orders:<commit-sha>
payments:<commit-sha>
catalog:<commit-sha>
frontend:<commit-sha>
```

### Best practices

-   Use immutable commit-based tags.
-   Deploy by digest for strongest artifact identity.
-   Do not rely on `latest` as a deployment identifier.
-   Use separate repositories or clear naming conventions per service.
-   Apply least-privilege IAM policies.
-   Scan images and control promotion to production.
-   Enable lifecycle policies and cross-region replication if required.
-   Retain release artifacts needed for rollback and compliance.

------------------------------------------------------------------------

## Q49. Do you have an archival or cleanup process for Docker images in AWS ECR?

### Interview answer

I would implement repository lifecycle policies to expire old untagged
images and retain a defined number of recent release images. The policy
must align with rollback, audit and compliance requirements. For images
that must be retained long term, I would use an explicit
release/retention strategy rather than allowing a general cleanup rule
to delete them.

### Example lifecycle policy

``` json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images after 14 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 14
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep the most recent 30 release images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["release-"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Apply and inspect:

``` bash
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region ap-south-1

aws ecr get-lifecycle-policy \
  --repository-name myapp \
  --region ap-south-1
```

### Architect considerations

-   Test lifecycle rules and review preview results before relying on
    them.
-   Ensure release tags match the policy exactly.
-   Keep rollback and compliance retention requirements.
-   Consider replication, backup/export and disaster recovery.
-   ECR lifecycle policies expire images; they are not a general-purpose
    archival storage tier.

------------------------------------------------------------------------

## Q50. How do you clean unused images and containers when disk space is full?

### Interview answer

I first identify whether disk pressure is caused by images, stopped
containers, build cache, volumes, logs or another filesystem consumer. I
inspect Docker's disk usage and the host filesystem before deleting
anything. I then remove only resources that are safe to remove.

``` bash
df -h
df -i
docker system df -v
docker ps -a
docker images
docker volume ls
```

Targeted cleanup:

``` bash
# Remove stopped containers
docker container prune

# Remove dangling images
docker image prune

# Remove unused build cache
docker builder prune
```

Broader cleanup:

``` bash
docker system prune
```

More aggressive:

``` bash
docker system prune -a
```

**High-risk command:**

``` bash
docker system prune -a --volumes
```

This can remove unused images, containers, networks, build cache and
unused volumes. A volume may contain important data even if no current
container is using it. Do not run this blindly in production.

### Prevention

Use disk alerts, image lifecycle policies, bounded build cache, log
rotation, ephemeral CI runners and documented cleanup procedures.
Monitor inode usage as well as bytes.

------------------------------------------------------------------------

# 10. Image Scanning & Security

## Q51. Are you aware of security scanning tools?

### Interview answer

Yes. Common container image and dependency scanning tools include: -
**Trivy:** Scans images, filesystems, repositories, Kubernetes
configurations and related artifacts. - **Grype:** Vulnerability scanner
for container images and filesystems. - **Snyk Container:** Container
and dependency security analysis. - **Anchore Enterprise / Syft +
Grype:** Image analysis, SBOM generation and vulnerability scanning. -
**Docker Scout:** Image analysis, recommendations and vulnerability
information. - **Amazon ECR image scanning:** Registry-level image
vulnerability scanning capabilities.

The exact tool depends on organizational licensing, CI/CD integration,
vulnerability database freshness and policy requirements.

------------------------------------------------------------------------

## Q52. How do you scan Docker images during build and at registry level?

### Interview answer

I use two checkpoints. In CI, I scan the image after it is built and
before it is pushed or promoted. At registry level, I enable the
registry's scanning capability so images are assessed when they are
stored and can be reassessed as vulnerability intelligence changes,
depending on the service and configuration.

### CI scan example with Trivy

``` bash
# Build
docker build -t myapp:"$GIT_COMMIT" .

# Scan for vulnerabilities
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  myapp:"$GIT_COMMIT"
```

This example fails the pipeline when findings in the selected severity
range are detected. In a real pipeline, define approved exceptions,
severity thresholds, exploitability context and remediation SLAs rather
than blindly blocking every finding.

### ECR registry scanning

Configure ECR scanning for the repository/registry and inspect findings
using the AWS Console or APIs/CLI supported by the configured scanning
mode. ECR offers basic scanning and enhanced scanning integrations;
capabilities and scan timing depend on region, account configuration and
current AWS service behavior.

### Pipeline flow

Build → test → generate SBOM → scan → policy evaluation → push/promote →
deploy by digest → continuous registry monitoring.

------------------------------------------------------------------------

## Q53. Any extensions or tools you are using for image scanning?

### Interview answer

Tools commonly integrated into a Docker security workflow include Trivy,
Grype, Snyk, Docker Scout and ECR image scanning. I would select a
standard tool based on the organization's security policy and integrate
it into CI/CD. I would also generate an SBOM so the team can understand
which packages and versions are included in an image.

### Example commands

``` bash
# Trivy image scan
trivy image myapp:1.0

# Scan only HIGH and CRITICAL and fail on findings
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:1.0

# Generate an SBOM (CycloneDX format)
trivy image --format cyclonedx --output sbom.json myapp:1.0
```

Tool capabilities and flags can change by version; validate commands
against the installed version. A scan is a point-in-time assessment and
does not guarantee that an image is vulnerability-free.

------------------------------------------------------------------------

## Q54. How would you ensure all Docker images are scanned before deployment?

### Interview answer

I would enforce scanning as a mandatory CI/CD quality gate and restrict
deployment to approved images. The pipeline would build the image, run
tests, generate an SBOM, scan for vulnerabilities and evaluate policy.
Only an image that passes policy would be pushed or promoted to the
production repository. Deployment would reference an immutable digest so
the scanned artifact is exactly the artifact being deployed.

### Example CI logic

``` bash
set -euo pipefail

IMAGE="123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp:${GIT_COMMIT}"

docker build -t "$IMAGE" .

trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  "$IMAGE"

docker push "$IMAGE"
```

### Enterprise controls

1.  Protect pipeline configuration and registry credentials.
2.  Require scan success before deployment approval.
3.  Use immutable tags and deploy by digest.
4.  Enforce admission policy in Kubernetes where applicable.
5.  Re-scan stored images as vulnerability intelligence changes.
6.  Define exception approval, expiry and remediation SLAs.
7.  Track scan coverage and policy violations.
8.  Sign images and verify signatures/attestations where the
    supply-chain security model requires it.

A CI check alone is not enough if users can bypass the pipeline and
deploy arbitrary images. Enforce policy at the deployment boundary as
well.

------------------------------------------------------------------------

# 11. Senior Interviewer Cross-Questions

These questions are not separate items in the uploaded questionnaire;
they are additional practice questions to test the depth of the topics
above.

## C1. Is a Docker container a virtual machine?

No. A container is an isolated process environment that shares the host
kernel. A VM virtualizes hardware and runs a guest OS/kernel. Containers
generally have lower overhead, while VMs provide a different isolation
and OS boundary.

## C2. Does `EXPOSE 8080` publish port 8080?

No. `EXPOSE` is image metadata documenting the intended port. To publish
it on the host, use `docker run -p host_port:container_port` or the
Compose `ports` setting.

## C3. Does a Docker health check restart an unhealthy container?

Not by itself under the normal Docker Engine restart-policy model. A
health check marks health status. Restart policies react to container
process exit. An orchestrator may take health status into account and
replace or restart workloads according to its own behavior.

## C4. Why should the application run as PID 1 and handle SIGTERM?

The main application process needs to receive termination signals and
shut down gracefully, for example by stopping new requests and
completing in-flight work. Exec-form startup helps avoid an intermediate
shell swallowing signals. PID 1 has special signal/reaping behavior in
Linux, so applications or a suitable init process must handle child
processes correctly.

## C5. What is the difference between a Docker volume and a container writable layer?

The writable layer belongs to a specific container and is removed with
that container. A named volume is managed separately and can be reused
by multiple containers. Persistent data should normally be kept outside
the container writable layer.

## C6. Why is `latest` not recommended for production deployment?

It is mutable and does not uniquely identify the artifact. A later push
can change what `latest` points to. Use immutable version/commit tags
and preferably deploy by image digest.

## C7. What is the difference between `docker stop` and `docker kill`?

`docker stop` sends the configured stop signal (SIGTERM by default),
waits for a grace period, then sends SIGKILL if the process has not
exited. `docker kill` sends SIGKILL by default, or another specified
signal, immediately. Prefer graceful stop for normal operations.

## C8. What happens when a container is removed?

The container's writable layer and container metadata are removed. Named
volumes are not removed by default. Data stored in bind mounts remains
in the host path. Image layers remain unless separately pruned.

## C9. How do you make Docker builds reproducible?

Pin base image versions/digests, use lock files, pin dependencies,
control build arguments, use deterministic build steps, minimize
external mutable inputs and record the source commit and resulting image
digest. Reproducibility also depends on the build toolchain and external
artifact availability.

## C10. What is the difference between build-time ARG and runtime ENV?

`ARG` is available during image build and is not automatically a runtime
environment variable. `ENV` is stored in image configuration and is
available to containers by default. Neither should be used to store
secrets because values can be exposed through image metadata or build
history.

## C11. Why should a database usually not be placed in the same container as the application?

Separating them allows independent lifecycle, scaling, resource
allocation, upgrades, backup and failure isolation. A single container
can be suitable for a disposable demo, but production data services need
deliberate persistence, recovery and operational management.

## C12. How do you reduce container privilege?

Run as non-root, drop unnecessary Linux capabilities, avoid privileged
mode, use seccomp and AppArmor/SELinux, restrict mounts, use read-only
filesystems where possible, apply resource limits and keep the
host/runtime patched.

------------------------------------------------------------------------

# 12. Scenario-Based Interview Practice

## Scenario 1: New application release starts returning HTTP 500 errors

**Investigation** 1. Confirm scope: all instances or one container; all
endpoints or one route. 2. Check deployment time and correlate with the
release. 3. Inspect container status, logs and exit/restart counts. 4.
Check environment/configuration and secret versions. 5. Verify database
and downstream service connectivity. 6. Compare the new image
digest/configuration with the previous healthy release.

``` bash
docker ps -a
docker logs --since 15m api
docker inspect api
docker stats --no-stream api
```

**Resolution** If the release is the likely cause, use the approved
rollback procedure to restore the last known-good immutable image.
Preserve logs and evidence, then reproduce and fix the defect in a
controlled environment.

**Prevention** Automated smoke tests, staged rollout, health/readiness
checks, backward-compatible database migrations, release observability
and a tested rollback path.

## Scenario 2: Database data disappeared after a container replacement

**Investigation** 1. Determine whether the database wrote to its
container writable layer or a mounted volume. 2. Inspect the old
container metadata if it still exists. 3. List volumes and inspect mount
configuration. 4. Check backup availability and restore points. 5. Avoid
creating or deleting volumes until the data path is understood.

``` bash
docker volume ls
docker volume inspect mysql-data
docker inspect old-db-container
```

**Resolution** If a persistent volume exists, attach the correct volume
to a compatible database image and validate data integrity. If data was
only in the writable layer and the container was removed, recovery may
be difficult; use backups or storage-level recovery where available.

**Prevention** Named volumes or managed database storage, automated
backups, restore testing, migration procedures and explicit
data-retention policies.

## Scenario 3: Container cannot connect to the database

**Investigation** 1. Verify both containers are running. 2. Verify they
share a network. 3. Resolve the database service name from the
application container. 4. Confirm database readiness and listening port.
5. Check credentials, database grants, TLS and application
configuration.

``` bash
docker ps
docker network inspect app-net
docker exec api getent hosts db
docker logs db
docker logs api
```

**Resolution** Correct network attachment, hostname, port, credentials
or readiness handling. Do not use `localhost` as the database host from
a separate application container.

**Prevention** Compose service discovery, private networks, secret
management, health checks, retry/backoff logic and connection
monitoring.

## Scenario 4: CI image build fails because the runner is out of disk

**Investigation** Check `df -h`, `df -i`, `docker system df -v`, build
cache, workspaces, image layers and logs. Identify the actual full
filesystem.

**Resolution** Remove confirmed disposable resources, prune targeted
build cache, reduce context/image size or increase runner capacity. Do
not delete volumes indiscriminately.

**Prevention** Ephemeral runners, bounded cache retention, image
lifecycle policies, build-context control and disk alerts.

## Scenario 5: Security scan finds a critical vulnerability in the base image

**Investigation** Identify the affected package, installed version,
fixed version, exploitability/context, whether the package is in the
final runtime image, and whether a patched base image is available.

**Resolution** Update the base image or dependency, rebuild, test and
rescan. If no fix exists, document a time-bound exception with
compensating controls and an owner; do not silently suppress the
finding.

**Prevention** Regular base-image rebuilds, dependency update
automation, SBOM generation, vulnerability SLAs and registry rescanning.

------------------------------------------------------------------------

# 13. Important Commands -- Quick Revision

  ---------------------------------------------------------------------------------------------
  Command                                       Purpose                 When to use
  --------------------------------------------- ----------------------- -----------------------
  `docker version`                              Show client/server      Verify Docker
                                                versions                installation and
                                                                        compatibility

  `docker info`                                 Show daemon and runtime Host/runtime
                                                details                 investigation

  `docker build -t app:1.0 .`                   Build image             Create an image from
                                                                        Dockerfile

  `docker images`                               List local images       Inspect available
                                                                        images

  `docker image inspect app:1.0`                Inspect image metadata  Check image
                                                                        configuration

  `docker history app:1.0`                      Show image layer        Investigate image
                                                history                 size/layers

  `docker run -d --name app -p 8080:80 nginx`   Create and start        Run a service
                                                container               

  `docker create --name app nginx`              Create without starting Prepare a container

  `docker start app`                            Start existing          Start a created/stopped
                                                container               container

  `docker ps -a`                                List all containers     Runtime troubleshooting

  `docker logs app`                             View container          Application
                                                stdout/stderr           startup/failure
                                                                        investigation

  `docker logs -f app`                          Follow logs             Live troubleshooting

  `docker exec -it app sh`                      Open a shell in running Inspect runtime
                                                container               environment

  `docker inspect app`                          Inspect detailed        Mount, network, state
                                                configuration/state     and restart
                                                                        investigation

  `docker stats`                                Show resource usage     CPU/memory
                                                                        troubleshooting

  `docker top app`                              Show container          Process-level
                                                processes               investigation

  `docker events`                               Show Docker events      Lifecycle/event
                                                                        investigation

  `docker stop app`                             Gracefully stop         Normal shutdown
                                                container               

  `docker kill app`                             Send signal, SIGKILL by Emergency termination
                                                default                 

  `docker rm app`                               Remove container        Remove stopped
                                                                        container

  `docker image prune`                          Remove dangling images  Targeted image cleanup

  `docker container prune`                      Remove stopped          Reclaim container
                                                containers              resources

  `docker builder prune`                        Remove build cache      Reclaim build-cache
                                                                        space

  `docker system df -v`                         Show Docker disk usage  Disk investigation

  `docker system prune`                         Remove selected unused  Controlled cleanup
                                                resources               

  `docker volume ls`                            List volumes            Persistent storage
                                                                        investigation

  `docker volume inspect NAME`                  Inspect volume          Verify mount/data
                                                                        location

  `docker network ls`                           List networks           Network investigation

  `docker network inspect NAME`                 Inspect network         Connectivity/DNS
                                                                        investigation

  `docker network create NAME`                  Create network          Isolate/connect
                                                                        application services

  `docker network connect NET CONTAINER`        Attach container to     Correct network
                                                network                 membership

  `docker compose up -d --build`                Build/start Compose     Start local
                                                stack                   multi-container app

  `docker compose ps`                           Show Compose services   Verify stack status

  `docker compose logs -f SERVICE`              Follow service logs     Debug a specific
                                                                        service

  `docker compose config`                       Validate/render Compose Validate before
                                                config                  deployment

  `docker compose down`                         Stop/remove Compose     Tear down stack,
                                                containers and networks preserving named
                                                                        volumes

  `docker login`                                Authenticate to         Registry push/pull
                                                registry                

  `docker tag SRC DEST`                         Add image tag           Prepare registry tag

  `docker push IMAGE`                           Push image              Publish artifact

  `docker pull IMAGE`                           Pull image              Retrieve artifact
  ---------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 14. Interview Mistakes to Avoid

1.  **Saying a container is a VM.** Explain that containers share the
    host kernel.
2.  **Saying `EXPOSE` publishes a port.** It is metadata; `-p` or
    Compose `ports` publishes.
3.  **Saying data persists automatically.** Container writable-layer
    data is not durable after container removal.
4.  **Using `localhost` for another container.** Use the
    service/container DNS name on a shared network.
5.  **Putting passwords in Dockerfile `ENV` or `ARG`.** Use runtime
    secret management or BuildKit secret mounts for build-time
    credentials.
6.  **Using `latest` as the only production tag.** Use immutable tags
    and deploy by digest.
7.  **Assuming health check failure always restarts a Docker
    container.** A Docker health status and restart policy are different
    mechanisms.
8.  **Running `docker system prune -a --volumes` without checking
    impact.** It can delete important unused volumes.
9.  **Assuming a smaller image is automatically secure.** Security also
    depends on patching, provenance, runtime privileges, configuration
    and vulnerability management.
10. **Claiming production implementation without evidence.** Clearly
    distinguish hands-on production work, lab practice and conceptual
    knowledge.

### How to make answers stronger

Use this sequence: **requirement → design decision → implementation →
validation → failure handling → security → operational trade-off**.
Explain why you selected an approach and what you would monitor after
deployment.

------------------------------------------------------------------------

# 15. Final Interview Revision

## Five points to remember

1.  Image = immutable layered template; container = runtime instance
    with a writable layer.
2.  Containers share the host kernel; namespaces isolate and cgroups
    control/account for resources.
3.  Persist state with volumes or managed storage, not the container
    writable layer.
4.  Use private user-defined networks and service DNS for
    container-to-container communication.
5.  Build, test, scan, tag immutably, push and deploy the exact approved
    image artifact.

## Five keywords to use naturally

-   **Immutable image**
-   **Multi-stage build**
-   **Least privilege**
-   **Persistent volume**
-   **Image digest**

## Strong concluding statement

"I treat Docker images as versioned, immutable deployment artifacts. My
focus is not only to build and run containers, but also to make the
image reproducible, secure, observable and efficient to operate. In
production, I pay particular attention to non-root execution, secrets
management, persistent storage, network isolation, resource limits,
vulnerability scanning and a reliable rollback strategy."

------------------------------------------------------------------------

# 16. Suggested Practice Plan

  -----------------------------------------------------------------------
  Session                 Focus                   Practice
  ----------------------- ----------------------- -----------------------
  1                       Docker fundamentals and Explain
                          layers                  image/container,
                                                  namespaces, cgroups and
                                                  copy-on-write

  2                       Dockerfile and          Build a multi-stage
                          optimization            Node.js or Python image

  3                       Runtime troubleshooting Reproduce restart
                                                  loops, bad
                                                  configuration and OOM
                                                  scenarios

  4                       Storage, secrets and    Build an API + database
                          networking              Compose stack with
                                                  persistent storage

  5                       Registry and security   Push to ECR, scan with
                                                  Trivy, review lifecycle
                                                  policy

  6                       Mock interview          Answer the 54 original
                                                  questions aloud and
                                                  handle follow-ups
  -----------------------------------------------------------------------

**Final reminder:** Be precise about what you have personally done. A
technically sound answer that clearly separates hands-on experience from
a proposed production design is stronger than an exaggerated claim.
