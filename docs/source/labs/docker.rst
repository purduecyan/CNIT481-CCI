******
Docker
******



Docker is a containerization platform that enables developers to package applications and their dependencies into lightweight, portable containers. Containers provide a consistent runtime environment, ensuring that applications run the same way across different systems. Docker simplifies application deployment, scaling, and management, making it easier to build, ship, and run applications in various environments, from local development machines to cloud infrastructure. Several components make up the Docker ecosystem, including the Docker Engine (the runtime), Docker CLI (command-line interface), Docker Hub (a cloud-based registry for sharing images), and Docker Compose (a tool for defining and running multi-container applications). See `https://www.docker.com/ <https://www.docker.com/>`_ for more information.

Alternatively, containerd is an industry-standard core container runtime that provides the essential functionalities needed to run containers. It is designed to be a lightweight, high-performance, and extensible runtime that can be integrated into various container orchestration systems, such as Kubernetes. Containerd handles tasks such as image management, container lifecycle management, and low-level storage and networking. It is often used as the underlying runtime for Docker, but it can also be used independently in other container ecosystems. See `https://containerd.io/ <https://containerd.io/>`_ for more information.


.. contents::
   :local:
   :depth: 2


Environment
===========

.. code-block:: bash
    :linenos:

    docker version
    docker info                     # quick peek of your environment
    docker run --rm hello-world     # confirm basic run works

If any command fails, ensure the Docker daemon is running and your user has permission to access the Docker socket.


Conventions
-----------

* Shell blocks assume *Linux/macOS Bash*.
* Replace ``$USER``, ``$REGISTRY`` or similar placeholders with values for your environment.
* Run commands from a writable working directory (e.g., a separate project folder).

.. note::

   Many examples use small images like ``alpine`` or ``busybox`` to keep builds fast.


Using Containers
================

Objectives
----------

* Pull, run, stop, start, and remove containers
* Inspect logs, processes, and metadata
* Map ports, pass environment variables, and manage restart policies
* Use resource controls


First Runs
----------

.. code-block:: bash
    :linenos:

    # Pull an image explicitly
    docker pull alpine:latest

    # Run a one-time command
    docker run --rm alpine:latest echo "Hello from Alpine"

    # Start an interactive shell
    docker run -it --name demo-alpine alpine:latest sh

    # Inside container, try a few commands
    uname -a
    cat /etc/os-release
    exit


Validation and Cleanup:
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
    :linenos:

    docker ps -a --filter name=demo-alpine
    docker logs demo-alpine
    docker rm demo-alpine



Ports, Environment, and Restart Policies
----------------------------------------

Run a tiny HTTP server in BusyBox.

.. code-block:: bash
    :linenos:

    # Start a background web server publishing container port 8080 to host port 8080
    docker run -d --name web1 \
        -p 8080:8080 \
        -e APP_MESSAGE="Hello from web1" \
        busybox:latest sh -c 'echo "$APP_MESSAGE" > index.html && httpd -f -p 8080'

    # Verify port binding
    curl -s localhost:8080

    # Restart policy: restart unless stopped
    docker update --restart unless-stopped web1
    docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' web1

    # Stop and start again
    docker stop web1 && docker start web1
    curl -s localhost:8080


Validation and Cleanup:
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   docker port web1
   docker rm -f web1


Exec, Copy, Inspect, Resource Limits
------------------------------------

.. code-block:: bash
    :linenos:

    docker run -d --name cpu-mem-demo alpine:latest sh -c "sleep 3600"

    # Exec into running container
    docker exec -it cpu-mem-demo sh -lc "echo inside && ls -la /"

    # Copy a file into the container
    echo "sample file" > local.txt
    docker cp local.txt cpu-mem-demo:/root/in-container.txt
    docker exec -it cpu-mem-demo sh -lc "cat /root/in-container.txt"

    # Inspect metadata
    docker inspect cpu-mem-demo | head -n 30

    # Apply resource limits (recreate container)
    docker rm -f cpu-mem-demo
    docker run -d --name cpu-mem-demo \
        --cpus="0.50" --memory="128m" --pids-limit=128 alpine:latest sleep 3600

Validation and Cleanup:
^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash
    :linenos:

    # Validation
    docker stats --no-stream cpu-mem-demo
    docker inspect -f '{{.HostConfig.NanoCpus}} {{.HostConfig.Memory}} {{.HostConfig.PidsLimit}}' cpu-mem-demo

    # Cleanup resources
    rm -f local.txt
    docker rm -f cpu-mem-demo


Create, Build, and Maintain Images
==================================

Objectives
----------

* Write Dockerfiles (``CMD`` vs ``ENTRYPOINT``, layers, caching, .dockerignore)
* Build single- and multi-stage images with BuildKit
* Tag, run, and test images; manage versions and labels
* Operate a local registry to push/pull


Working with Dockerfile
-----------------------

A typical Docker project structure may look like this:

.. code-block:: text

   app/
   ├─ Dockerfile
   ├─ .dockerignore
   └─ server.py

Create files:

.. code-block:: bash

   mkdir -p app && cd app

.. code-block:: python
    :linenos:
    :caption: server.py

    from http.server import BaseHTTPRequestHandler, HTTPServer
    import os

    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            msg = os.getenv("APP_MESSAGE", "Hello, Docker!")
            self.send_response(200)
            self.end_headers()
            self.wfile.write(msg.encode("utf-8"))

    if __name__ == "__main__":
        port = int(os.getenv("PORT", "8000"))
        server = HTTPServer(("0.0.0.0", port), Handler)
        print(f"Serving on :{port}")
        server.serve_forever()


.. code-block:: dockerfile
    :linenos:
    :caption: Dockerfile

    FROM python:3.13-slim
    LABEL org.opencontainers.image.title="simple-http" \
            org.opencontainers.image.source="https://example.invalid/demo" \
            org.opencontainers.image.description="A tiny HTTP server demo"
    WORKDIR /app
    COPY server.py .
    EXPOSE 8000
    ENV PORT=8000
    # Use the exec form to preserve signals and avoid shell quirks
    CMD ["python", "server.py"]

.. code-block:: text
    :linenos:
    :caption: .dockerignore

    __pycache__/
    *.pyc
    .git
    .env
    .DS_Store


Build & run:

.. code-block:: bash
    :linenos:

    # Enable BuildKit (Optional. Enabled by default on new Docker installs.)
    export DOCKER_BUILDKIT=1

    docker build -t simple-http:1.0.0 .
    docker run -d --name simple-http -p 8000:8000 -e APP_MESSAGE="Hello from 1.0.0" simple-http:1.0.0
    curl -s localhost:8000


Validation:

.. code-block:: bash

    docker image ls simple-http
    docker inspect -f '{{.Config.Env}}' simple-http:1.0.0

Cleanup:

.. code-block:: bash

    docker rm -f simple-http

Multi‑Stage Build and Image Slimming
------------------------------------

Refactor to separate build and runtime:

.. code-block:: dockerfile
    :linenos:
    :caption: Dockerfile (multi-stage)

    # syntax=docker/dockerfile:1.19.0
    FROM python:3.13-slim AS base
    WORKDIR /app
    COPY server.py .

    FROM gcr.io/distroless/python3-debian12 AS runtime
    WORKDIR /app
    COPY --from=base /app/server.py /app/server.py
    ENV PORT=8000
    EXPOSE 8000
    ENTRYPOINT ["/usr/bin/python3", "/app/server.py"]


Build & compare sizes:

.. code-block:: bash

    docker build -t simple-http:2.0.0 -f Dockerfile .
    docker image ls simple-http

Run and validate:

.. code-block:: bash

    docker run -d --name simple-http -p 8000:8000 -e APP_MESSAGE="Hello, Multi-stage" simple-http:2.0.0
    curl -s localhost:8000
    docker rm -f simple-http


Tagging, Labels, Digests, Healthcheck
-------------------------------------

.. code-block:: dockerfile
    :linenos:
    :caption: Dockerfile (add healthcheck)

    FROM python:3.13-slim
    WORKDIR /app
    COPY server.py .
    ENV PORT=8000
    EXPOSE 8000
    HEALTHCHECK --interval=10s --timeout=2s --retries=3 \
        CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000', timeout=1).read() or exit(1)"
    CMD ["python", "server.py"]


.. code-block:: bash
    :linenos:

    docker build -t simple-http:2.1.0 .
    docker tag simple-http:2.1.0 simple-http:latest
    docker run -d --name simple-http-healthcheck -p 8000:8000 simple-http:latest
    sleep 2
    docker inspect -f '{{.State.Health.Status}}' simple-http-healthcheck
    docker inspect --format='{{index .RepoDigests 0}}' simple-http:2.1.0
    docker rm -f simple-http-healthcheck



Local Registry for Push/Pull
----------------------------

Start a local registry:

.. code-block:: bash

   docker run -d --name registry -p 5000:5000 registry:2

Tag & push:

.. code-block:: bash

   docker tag simple-http:2.1.0 localhost:5000/simple-http:2.1.0
   docker push localhost:5000/simple-http:2.1.0

Test pull:

.. code-block:: bash

   docker rmi simple-http:2.1.0
   docker pull localhost:5000/simple-http:2.1.0

Cleanup:

.. code-block:: bash

   docker rm -f registry

.. note::

   Pushing to Docker Hub or another remote registry follows the same pattern after authenticating with ``docker login`` and tagging with the registry hostname.


Advanced Storage, Networking, and Security
==========================================

Objectives
----------

* Persist data with volumes, use bind mounts, tmpfs, and backups
* Build user‑defined bridge networks, DNS-based service discovery, and port publishing
* Apply security hardening: non‑root users, capabilities, read‑only rootfs, seccomp/AppArmor/SELinux basics, and ``no-new-privileges``



Volumes, Bind Mounts, Backups
-----------------------------

Create and use a named volume:

.. code-block:: bash
    :linenos:

    docker volume create appdata
    docker run -d --name vol-demo -v appdata:/data alpine:latest sh -c 'echo "persist me" > /data/file.txt; sleep 3600'
    docker exec vol-demo cat /data/file.txt
    docker rm -f vol-demo

    # Re-attach the same volume to verify persistence
    docker run --rm -v appdata:/data alpine:latest cat /data/file.txt


Bind mounts (host path):

.. code-block:: bash
    :linenos:

    mkdir -p "$(pwd)/hostdir"
    echo "hello from host" > "$(pwd)/hostdir/host.txt"
    docker run --rm \
        --mount type=bind,src="$(pwd)/hostdir",dst=/mnt,readonly \
        alpine:latest sh -lc 'echo "Listing:" && ls -la /mnt && echo "Try write:" && (echo hi > /mnt/x || echo "write blocked (as expected)")'

Tmpfs (in‑memory):

.. code-block:: bash

    docker run --rm --tmpfs /tmp:rw,size=64m alpine:latest sh -lc 'mount | grep /tmp && echo data > /tmp/inmem && ls -la /tmp'

Backup/restore a volume (using a throwaway helper):

.. code-block:: bash
    :linenos:

    # Backup appdata to a tar stream into host file
    docker run --rm -v appdata:/data -v "$(pwd)":/backup alpine:latest \
        sh -lc 'tar -C /data -czf /backup/appdata.tgz .'

    # Restore into a fresh volume
    docker volume create appdata2
    docker run --rm -v appdata2:/data -v "$(pwd)":/backup alpine:latest \
        sh -lc 'tar -C /data -xzf /backup/appdata.tgz && ls -la /data'

Cleanup:

.. code-block:: bash

    docker volume rm appdata appdata2
    rm -rf hostdir appdata.tgz



User‑Defined Networks and DNS
-----------------------------

Create a user-defined bridge network and connect services by name:

.. code-block:: bash
    :linenos:

    docker network create appnet

    # Service A: a tiny key-value service using BusyBox HTTPD on the appnet
    docker run -d --name svc-a --network appnet busybox:1.36 sh -c 'echo "svc-a" > index.html && httpd -f -p 8080'

    # Service B: curl Service A by its container name (DNS from user-defined network)
    docker run --rm --network appnet curlimages/curl:8.9.1 curl -s http://svc-a:8080

    # Publish a separate service to the host on :8081 (initially on default bridge)
    docker run -d --name svc-a-pub -p 8081:8080 busybox:1.36 sh -c 'echo "svc-a-public" > index.html && httpd -f -p 8080'
    curl -s http://127.0.0.1:8081

Explore networking metadata:

.. code-block:: bash

    docker network inspect appnet
    docker port svc-a-pub

Connect / disconnect ``svc-a-pub`` to/from ``appnet``:

.. code-block:: bash

    docker network connect appnet svc-a-pub
    docker network disconnect appnet svc-a-pub

Cleanup:

.. code-block:: bash

    docker rm -f svc-a svc-a-pub
    docker network rm appnet

.. note::

   ``host`` and ``none`` networks have special behavior. User-defined bridge networks provide built-in DNS-based service discovery without extra tooling.

Security Hardening Essentials
-----------------------------

Run as non‑root (build-time) and drop capabilities:

.. code-block:: dockerfile
    :linenos:
    :caption: Dockerfile (non-root user)

    FROM alpine:latest
    RUN adduser -D appuser
    USER appuser
    WORKDIR /home/appuser
    COPY --chown=appuser:appuser . /home/appuser
    CMD ["sh", "-lc", "id && sleep 3600"]

.. code-block:: bash
    :linenos:

    docker build -t security-demo:nonroot .
    docker run -d --name nonroot security-demo:nonroot
    docker exec nonroot id

Drop Linux capabilities by default and selectively add:

.. code-block:: bash
    :linenos:

    # Try to run ping (needs CAP_NET_RAW)
    docker run --rm --cap-drop ALL alpine:latest sh -lc 'apk add --no-cache iputils >/dev/null && ping -c1 1.1.1.1 || echo "No CAP_NET_RAW -> ping fails (expected)"'

    # Add CAP_NET_RAW back
    docker run --rm --cap-drop ALL --cap-add NET_RAW alpine:latest sh -lc 'apk add --no-cache iputils >/dev/null && ping -c1 -W1 1.1.1.1 && echo "Ping works with NET_RAW"'

Read‑only root filesystem and writable mounts:

.. code-block:: bash

    docker run --rm --read-only --tmpfs /tmp alpine:latest sh -lc 'echo ok > /tmp/x && echo "wrote to tmpfs ok"; (echo fail > /etc/x || echo "rootfs is read-only (expected)")'

No new privileges:

.. code-block:: bash

    docker run --rm --security-opt=no-new-privileges alpine:latest sh -lc 'echo "no-new-privileges active"'

Optional: Custom seccomp / AppArmor / SELinux
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* Default seccomp profile already blocks many risky syscalls. You can pass ``--security-opt seccomp=/path/profile.json`` for custom profiles.
* AppArmor (Debian/Ubuntu) and SELinux (Fedora/RHEL) apply host MAC policies. You can select profiles via ``--security-opt apparmor=profile`` or ``--security-opt label=...`` (SELinux).  
  Consult your OS documentation for appropriate profiles and labels.

Cleanup:

.. code-block:: bash

    docker rm -f nonroot


Multi‑Architecture Builds
=========================

Objectives
----------

* Use ``docker buildx`` for cross‑platform builds
* Emulate architectures via ``binfmt_misc`` (QEMU)
* Build, push, and verify multi‑arch images


Enable Buildx and Binfmt
------------------------

.. code-block:: bash
    :linenos:

    # Ensure Buildx is available
    docker buildx version

    # Ensure a builder exists (create if needed)
    docker buildx ls
    docker buildx create --name multi --use || docker buildx use multi
    docker buildx inspect --bootstrap

    # (Optional) Install QEMU emulation for cross-building
    docker run --privileged --rm tonistiigi/binfmt --install all
    docker buildx inspect --bootstrap

Build for linux/amd64 and linux/arm64
-------------------------------------

Use the ``app`` from Module 2 or create a fresh minimal sample.

.. code-block:: bash
    :linenos:

    cd app  # if not already there

    # Ensure a local registry is running for pushing the multi-arch manifest
    docker run -d --name registry -p 5000:5000 registry:2

    # Build a multi-arch image and push to the local registry
    docker buildx build \
        --platform linux/amd64,linux/arm64 \
        -t localhost:5000/simple-http:multi \
        --push .

Inspect the manifest:

.. code-block:: bash

    docker buildx imagetools inspect localhost:5000/simple-http:multi

Optionally, pull a specific platform (on a multi‑arch host):

.. code-block:: bash

    docker pull --platform linux/arm64 localhost:5000/simple-http:multi
    docker image rm localhost:5000/simple-http:multi

Cleanup:

.. code-block:: bash

    docker rm -f registry
    # keep the builder 'multi' for future use or remove with:
    # docker buildx rm multi


Maintenance and Housekeeping
============================

Objectives
----------

* Keep images current, manage tags, and rebuild with cache awareness
* Export, import, and inspect images; prune unused resources
* Diagnose disk usage and layer sharing


Upgrades, Retagging, and Rebuilds
---------------------------------

.. code-block:: bash
    :linenos:

    # Inspect base image and digests
    docker history simple-http:2.1.0
    docker inspect --format '{{.RepoDigests}}' simple-http:2.1.0

    # Retag for a new release
    docker tag simple-http:2.1.0 simple-http:2.1.1

    # Rebuild with cache (e.g., after code change)
    touch server.py
    docker build -t simple-http:2.1.1 .
    docker image ls simple-http

Save/Load and Export/Import
---------------------------

.. code-block:: bash
    :linenos:

    # Save image as tarball and load elsewhere
    docker save simple-http:2.1.1 | gzip > simple-http_2.1.1.tar.gz
    docker image rm simple-http:2.1.1
    gunzip -c simple-http_2.1.1.tar.gz | docker load

    # Export a container filesystem (no image metadata) and import
    docker run --name export-demo simple-http:2.1.1 sh -lc 'echo "artifact" > /app/art.txt; sleep 1'
    docker export export-demo | gzip > export-demo.tar.gz
    docker rm -f export-demo
    gunzip -c export-demo.tar.gz | docker import - import:v1
    docker run --rm import:v1 sh -lc 'ls -la /app'

Disk Usage and Pruning
----------------------

.. code-block:: bash
    :linenos:

    docker system df
    docker image prune -f
    docker container prune -f
    docker volume prune -f
    docker builder prune -f

.. warning::

   ``prune`` deletes unused resources. Ensure you don’t need dangling images/volumes before pruning.


Best Practices & Cheat‑Sheet
============================

General
-------

* **Prefer minimal bases** (``alpine``, distroless) and **multi‑stage builds** to reduce image size and attack surface.
* **Pin versions** (and optionally **digests**) for reproducible builds.
* Use **.dockerignore** to keep contexts small. Avoid copying your entire repo when only a subdir is needed:
  
  .. code-block:: dockerfile

     COPY --link src/ /app/          # when using BuildKit
* Order Dockerfile steps to **maximize cache hits** (install dependencies before copying fast‑changing app code).
* Use **exec form** for ``CMD``/``ENTRYPOINT`` to preserve signals and avoid shell interpolation pitfalls.
* Emit logs to **stdout/stderr**; don’t write logs to files inside the container.
* Keep containers **immutable and ephemeral**; store state in **volumes**.

Security
--------

* Do not store passwords or security-sensitive information on containers.
* Run as **non‑root** (``USER``), **drop capabilities** (``--cap-drop ALL``; add back only what you need).
* Use **read‑only rootfs**, **tmpfs** for scratch paths, and **no‑new‑privileges** where possible.
* Keep images **patched** and **rebuilt regularly**; avoid installing extra tools in runtime images.
* Prefer **distroless** or slim runtimes and fetch debug shells only in a **debug build stage**.

Networking & Storage
--------------------

* Use **user-defined bridge networks** for isolation and **DNS-based discovery**.
* Map ports deliberately; avoid exposing unnecessary ports.
* Persist data to **named volumes**; use **bind mounts** for local dev only.
* Use **tmpfs** for sensitive ephemeral data to keep it in memory.

Multi‑Arch & Build
------------------

* Use **BuildKit** and **buildx** for parallel, multi‑platform builds and advanced features (``--platform``, inline cache, provenance/SBOM as supported by your version).
* Test images on target platforms and **verify manifests** with ``docker buildx imagetools inspect``.

Operational Tips
----------------

* Label images using OCI labels (``org.opencontainers.image.*``) for traceability.
* Use **healthchecks** for better container lifecycle signals.
* Monitor **resource limits** (CPU/mem/PIDs) and adapt to workload needs.
* Regularly **prune** unused artifacts and audit **disk usage** with ``docker system df``.


Troubleshooting
===============

* **Port already in use**: choose another host port (e.g., ``-p 8082:8080``) or stop conflicting process.
* **Permission denied on bind mount**: check path exists and permissions; on SELinux systems, consider adding context options (e.g., ``:Z``).
* **DNS resolution between containers** works only on **user-defined** networks (not the default ``bridge`` with container names).
* **QEMU/binfmt not installed**: run ``tonistiigi/binfmt`` helper and re-bootstrap the builder.
