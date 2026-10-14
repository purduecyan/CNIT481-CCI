**************
Docker Compose
**************

.. contents::
   :local:
   :depth: 2

Introduction
============

The Docker Compose tool allows you to define and manage multi-container Docker
applications using simple YAML files. This article guides you through creating a
robust development and production-ready stack using Docker Compose, covering essential
features like service definitions, networking, volumes, healthchecks, scaling, and
profiles. In this tutorial you will:

- Define multi-service stacks
- Wire up networks, volumes, healthchecks, and dependencies
- Use **profiles** for dev/test/observability extras
- Scale services
- Prepare a production-oriented override

Prerequisites
-------------

- Ubuntu 26.04 with Docker Engine and the **Compose plugin** (the ``docker compose``
  CLI) installed. If you followed the Docker lab, you already have both.
- Familiarity with Docker images, containers, volumes, and networks.

Install the Compose plugin if it is missing, then confirm the versions:

.. code-block:: bash
    :linenos:

    # From Docker's APT repository (see the Docker lab's install section)
    sudo apt-get install -y docker-compose-plugin
    # ...or, using Ubuntu's own packages instead:
    # sudo apt-get install -y docker-compose-v2

    docker version --format 'Engine {{.Server.Version}}'
    docker compose version        # this lab requires Compose v2.24 or newer

.. note::

   Always type ``docker compose`` (a space, the v2+ plugin). The legacy Python
   ``docker-compose`` (hyphen, v1) is end-of-life and is not used anywhere in this lab.
   Compose looks for ``compose.yaml`` first and still accepts ``docker-compose.yml``;
   this lab uses ``docker-compose.yml`` throughout. A top-level ``version:`` key is
   obsolete and should be omitted.


Example Project Structures
--------------------------

Minimal (two services)
^^^^^^^^^^^^^^^^^^^^^^

This structure is what you build in the first sections of this tutorial: a simple web
app behind an Nginx reverse proxy. All commands run from ``compose-labs/``.

.. code-block:: text

   compose-labs/
   ├── app/
   │   ├── Dockerfile
   │   ├── requirements.txt
   │   └── src/
   │       └── app.py
   ├── nginx/
   │   └── default.conf
   ├── docker-compose.yml
   └── docker-compose.override.yml


Full (dev + prod, profiles, tests)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

From :ref:`compose_profiles_1` onward, the Compose files move into a ``compose/``
subdirectory and gain profiles, a worker, and environment-specific overrides. From that
point, run ``docker compose`` commands from ``compose-labs/compose/``.

.. code-block:: text

   compose-labs/
   ├── app/
   │   ├── Dockerfile
   │   ├── requirements.txt
   │   └── src/
   │       └── app.py
   ├── worker/
   │   ├── Dockerfile
   │   └── worker.py
   ├── nginx/
   │   └── default.conf
   ├── compose/
   │   ├── docker-compose.yml              # Base stack (web, app, db) + profiled services
   │   ├── docker-compose.override.yml     # Dev overrides (bind mounts, live reload)
   │   ├── docker-compose.prod.yml         # Prod overrides (limits, restart policies)
   │   └── prometheus.yml                  # Used by the "obs" profile
   └── tests/
       └── test_smoke.py


First Compose Stack
===================

**Objective:** Define a single-container service (Nginx) with port mapping.

In this exercise, we will create a simple Docker Compose file that defines a single
service running Nginx, a popular web server. We will map port 8080 on the host to port
80 in the container so that we can access the Nginx welcome page from our browser.

1. Create the folder and base file:

   .. code-block:: bash
       :linenos:

       # Create a directory named 'compose-labs' if it doesn't exist
       mkdir -p compose-labs && cd compose-labs

       # Create docker-compose.yml with the following YAML content
       cat > docker-compose.yml <<'YAML'
       services:                     # Define the services for Docker Compose
         web:                        # Service name: 'web'
           image: nginx:1.27-alpine  # Pin a specific tag rather than 'latest'
           ports:
             - "8080:80"             # Map host port 8080 to container port 80 (HTTP)
       YAML

   .. warning::

      YAML nesting is defined by indentation. ``image`` and ``ports`` must be indented
      *under* ``web``; if they line up with ``web``, Compose rejects the file with
      ``services.image must be a mapping``. Run ``docker compose config`` after every
      edit — it validates the file and prints the fully resolved model.

2. Bring up the stack:

   .. code-block:: bash

      docker compose up -d
      docker compose ps

**Expected Outcome:** Open http://localhost:8080 and see the Nginx welcome page.


Add an App Service
==================

**Objective:** Add a Python API and a reverse proxy.

Next, we will add a simple Python Flask application that serves a health check
endpoint. We will also configure Nginx to act as a reverse proxy, forwarding requests
from port 80 to the Flask app running on port 5000.

1. Create the ``app``:

   .. code-block:: bash
       :linenos:

       # Create the directory structure: 'app/src'
       mkdir -p app/src

       # Create a Dockerfile inside 'app' directory
       cat > app/Dockerfile <<'DOCKER'
       # Use a lightweight Python 3.13 base image
       FROM python:3.13-slim
       WORKDIR /app
       COPY requirements.txt .
       RUN pip install --no-cache-dir -r requirements.txt
       COPY src/ src/
       EXPOSE 5000
       # Serve with gunicorn (a production WSGI server) rather than Flask's dev server
       CMD ["gunicorn", "--chdir", "src", "--bind", "0.0.0.0:5000", "app:app"]
       DOCKER

       # Create a requirements.txt file with Python dependencies
       cat > app/requirements.txt <<'REQ'
       flask==3.0.3
       gunicorn==22.0.0
       REQ

       # Create the Flask application in app/src/app.py
       cat > app/src/app.py <<'PY'
       from flask import Flask, jsonify
       # Initialize the Flask application
       app = Flask(__name__)

       # Define a GET endpoint for health check
       @app.get("/api/health")
       def health():
           # Return a JSON response with status "ok"
           return jsonify(status="ok")

       # Allow `python src/app.py` for quick local runs outside Docker
       if __name__ == "__main__":
           app.run(host="0.0.0.0", port=5000)
       PY

2. Add Nginx config:

   .. code-block:: bash
       :linenos:

       mkdir -p nginx
       cat > nginx/default.conf <<'NGINX'
       server {
           listen 80;
           location / {
               proxy_pass http://app:5000;
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
           }
       }
       NGINX

3. Update ``docker-compose.yml``:

   .. code-block:: yaml
       :linenos:

       # Root section defining all services managed by Docker Compose
       services:
         # Reverse proxy / static web server
         web:
           # Pin the official NGINX image to a specific tag
           image: nginx:1.27-alpine

           # Bind-mount local nginx/default.conf into the container (read-only)
           volumes:
             - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro

           # Publish container port 80 on host port 8080
           ports:
             - "8080:80"

           # Start order only: 'app' is *started* before 'web' (not "ready")
           depends_on:
             - app

         # Backend application service
         app:
           # Build the image from the Dockerfile in ./app
           build: ./app

           # Documents the internal port; services on the same network can reach it
           # without this, and it is NOT published to the host
           expose:
             - "5000"

4. Up:

   .. code-block:: bash

      docker compose up -d --build
      curl -s http://localhost:8080/api/health

**Expected Outcome:** ``{"status":"ok"}`` is returned via Nginx → app.


Network, Environment, and Volumes
=================================

**Objective:** Add Postgres with a named volume; wire up app via environment variables
and a user-defined network.

At this point, we have three services: ``web``, ``app``, and ``db`` (Postgres). We will
create a private network for backend communication, persist Postgres data with a named
volume, and configure the app to connect to the database using environment variables.
Additionally, we will set up service dependencies to ensure proper startup order. The
services will be able to communicate with each other using their service names as
hostnames.

1. Extend Compose:

   .. code-block:: yaml
       :linenos:

       # Define custom networks for inter-service communication
       networks:
         # Private backend network shared by web, app, and db
         backend:

       # Volume to persist PostgreSQL data across container restarts
       volumes:
         db_data:

       services:
         web:
           # ... as before ...
           networks: [backend]
         app:
           # ... as before ...
           # Inject the database connection string into the app
           environment:
             DATABASE_URL: "postgresql://appuser:apppass@db:5432/appdb"
           # Attach the app service to the private backend network
           networks: [backend]
           # Start app after db has *started* (readiness comes in the next section)
           depends_on:
             - db
         db:
           image: postgres:16-alpine
           # Configure database credentials and default database
           environment:
             POSTGRES_USER: appuser
             POSTGRES_PASSWORD: apppass
             POSTGRES_DB: appdb
           # Persist database files outside the container
           volumes:
             - db_data:/var/lib/postgresql/data
           # Attach the database to the private backend network
           networks: [backend]

2. Recreate:

   .. code-block:: bash

      docker compose up -d

**Expected Outcome:**

- ``db_data`` volume persists Postgres data.
- Services can reach each other via service names on ``backend``.

.. note::

   The sample Flask app does not open a database connection yet; ``DATABASE_URL`` is
   there to show how configuration is injected. Credentials in plain YAML are fine for a
   local lab only — see the production checklist below.

.. caution::

   Keep the ``postgres:16-alpine`` pin for this lab. Starting with ``postgres:18`` the
   image expects its volume at ``/var/lib/postgresql`` (not ``.../data``), so blindly
   bumping the tag with this volume line will fail at startup.


Healthchecks and Conditional Start
==================================

**Objective:** Ensure ``app`` waits until Postgres is ready.

Add healthcheck to ``db`` and modify ``app``'s ``depends_on``.

.. code-block:: yaml
    :linenos:

    services:
      db:
        image: postgres:16-alpine
        # ... as before ...
        healthcheck:
          # Simple command to check if Postgres is ready to accept connections
          test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
          # Healthcheck settings
          interval: 5s
          timeout: 3s
          retries: 10
          start_period: 10s   # grace period: failures don't count while the DB initialises

      app:
        build: ./app
        # ... as before ...
        depends_on:
          db:
            # Condition to wait for healthy status
            condition: service_healthy

Check:

.. code-block:: bash

   docker compose ps
   docker compose logs db --tail=50

**Expected Outcome:** ``app`` starts *after* ``db`` reports healthy.

.. note::

   ``depends_on`` supports three conditions:

   - ``service_started`` (default) — the dependency's container has started.
   - ``service_healthy`` — the dependency's **healthcheck** passes. The dependency must
     be a *long-running* container: a healthcheck only runs while the container runs,
     so a container that exits makes its dependents fail with
     ``dependency failed to start: container ... exited (0)``.
   - ``service_completed_successfully`` — the dependency ran to completion with exit
     code 0. Use this for one-shot jobs such as migrations or static-site builds.


Scaling the App
===============

**Objective:** Run multiple replicas of ``app``.

Next, we will scale the ``app`` service to run multiple instances. This is useful for
load balancing and improving the availability of our application. We will use Docker
Compose's scaling feature to achieve this.

.. code-block:: bash

   docker compose up -d --scale app=3
   docker compose restart web        # make Nginx re-resolve 'app' (see note)
   docker compose ps app

.. note::

   Compose's DNS returns one address per ``app`` replica. Nginx resolves ``app`` once,
   when it starts or reloads, and then round-robins across the addresses it found. It
   does not notice replicas added later and does no health-based balancing. That is why
   ``web`` is restarted after scaling. A service can only be scaled if it does not
   publish a fixed host port or set ``container_name``. ``app`` qualifies because it
   only uses ``expose``.


Dev Overrides and Live Code
===========================

**Objective:** Use ``docker-compose.override.yml`` to mount source and enable an
auto-reloader.

In development, it's common to want to see code changes reflected immediately without
rebuilding images. We will create a Docker Compose override file that mounts the
application source code into the container and enables Flask's debug mode for live
reloading.

Example Workflow
----------------

Create ``docker-compose.override.yml`` next to ``docker-compose.yml``:

.. code-block:: yaml
    :linenos:

    services:
      app:
        # Mount local source code into the container for live editing
        volumes:
          - ./app/src:/app/src
        # Replace gunicorn with Flask's auto-reloading dev server
        command: python -m flask --app src/app.py run --debug --host=0.0.0.0 --port=5000

Restart:

.. code-block:: bash

   docker compose up -d --build

Edit files under ``app/src`` and refresh your browser.

.. important::

   ``docker-compose.override.yml`` is applied **automatically** whenever you run
   ``docker compose`` **without** ``-f``. To run *without* it, name the base file
   explicitly: ``docker compose -f docker-compose.yml up -d``. Use
   ``docker compose config`` to see the merged result.

   When files are merged, single values (``image``, ``command``) are replaced, but
   **lists such as** ``ports`` **are appended**. An override that sets
   ``ports: ["8081:80"]`` therefore publishes *both* 8080 and 8081. To replace a list,
   tag it with ``!override`` (Compose 2.24+): ``ports: !override ["8081:80"]``. Use
   ``!reset`` to remove a key entirely.


.. _compose_profiles_1:

Profiles - I
============

**Objective:** Use **profiles** to conditionally include services and developer tools.

Profiles allow you to define optional services that can be activated based on the
environment or use case. As an example, we will add a Redis cache, a background worker,
and developer tools like Mailhog and Adminer, each behind specific profiles. Mailhog is
a web-based email testing tool, and Adminer is a lightweight database management tool.
The profiles will help us keep the base stack clean while enabling additional
functionality as needed.

- **Production Profile:** Includes Redis cache and worker for background tasks.
- **Development Profile:** Adds cache, worker, Mailhog for email testing, and Adminer
  for database management.
- **QA Profile:** Includes cache, worker, and Adminer.

Before you start, stop the stack from the previous sections (it also publishes port
8080) and create the new layout:

.. code-block:: bash
    :linenos:

    docker compose down          # run in compose-labs/
    mkdir -p compose worker

Create a minimal background worker for the ``worker`` service:

.. code-block:: bash
    :linenos:

    cat > worker/Dockerfile <<'DOCKER'
    FROM python:3.13-slim
    WORKDIR /worker
    COPY worker.py .
    CMD ["python", "-u", "worker.py"]
    DOCKER

    cat > worker/worker.py <<'PY'
    import os, time
    # Placeholder job loop: a real worker would pull jobs from Redis
    while True:
        print(f"worker alive; queue={os.getenv('REDIS_URL')}", flush=True)
        time.sleep(10)
    PY


Base Architecture
-----------------

The overall architecture with profiles looks like this:

.. graphviz::
    :caption: Overall Architecture with Profiles
    :align: center

    digraph compose_architecture {
      rankdir=LR;
      fontsize=12;
      fontname="Arial";

      // Networks
      subgraph cluster_frontend {
        label="Frontend Network";
        style=dashed;
        web [shape=oval, style=filled, fillcolor=yellow];
      }

      subgraph cluster_backend {
        label="Backend Network";
        style=dashed;
        app [shape=oval, style=filled, fillcolor=yellow];
        db [shape=oval, style=filled, fillcolor=yellow];
        cache;
        worker;
        mailhog;
        adminer;
      }

      // Volumes
      db_data [shape=cylinder, label="db_data", fillcolor=lightblue, style=filled];
      redis_data [shape=cylinder, label="redis_data", fillcolor=lightblue, style=filled];

      // Services
      web -> app [label="depends_on"];
      app -> db [label="DATABASE_URL"];
      app -> cache [label="REDIS_URL"];
      worker -> db [label="DATABASE_URL"];
      worker -> cache [label="REDIS_URL"];
      db -> db_data [label="volume"];
      cache -> redis_data [label="volume"];

      // Profiles
      mailhog [shape=box, style="filled,dotted", fillcolor=lightgrey, label="mailhog\ndev only"];
      adminer [shape=box, style="filled,dotted", fillcolor=lightgrey, label="adminer\ndev, qa"];
      worker [shape=box, style="filled,dotted", fillcolor=lightgrey, label="worker\ndev, qa, prod"];
      cache [shape=box, style="filled,dotted", fillcolor=lightgrey, label="cache\ndev, qa, prod"];

      // Overrides
      app_override [shape=note, label="app override\n(dev: live reload)"];
      prod_limits [shape=note, label="prod limits\nCPU & memory"];

      app -> app_override [style=dotted];
      app -> prod_limits [style=dotted];
      worker -> prod_limits [style=dotted];
      web -> prod_limits [style=dotted];
    }


The base architecture without profiles looks like this:

.. graphviz::
    :caption: Base Architecture
    :align: center

    digraph base {
      rankdir=LR;
      fontsize=12;
      fontname="Arial";
      node [shape=oval, style=filled, fillcolor=yellow];

      web -> app [label="depends_on"];
      app -> db [label="DATABASE_URL"];
      db -> db_data [label="volume"];

      subgraph cluster_backend {
        style=dashed;
        label="Backend Network";
        db; app;
      }

      subgraph cluster_frontend {
        style=dashed;
        label="Frontend Network";
        web;
      }

      // Volumes
      db_data [shape=cylinder, label="db_data", fillcolor=lightblue, style=filled];
    }


Base file: ``compose/docker-compose.yml`` (paths are relative to the ``compose/``
directory, hence the ``../`` prefixes)

.. code-block:: yaml
    :linenos:

    name: compose-labs

    networks:
      backend:
      frontend:

    volumes:
      db_data:
      redis_data:

    services:
      web:
        image: nginx:1.27-alpine
        volumes:
          - ../nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
        ports:
          - "8080:80"
        depends_on: [app]
        networks: [frontend, backend]

      app:
        build: ../app
        environment:
          DATABASE_URL: postgresql://appuser:apppass@db:5432/appdb
          REDIS_URL: redis://cache:6379/0
        expose: ["5000"]
        depends_on:
          db:
            condition: service_healthy
        networks: [backend]

      db:
        image: postgres:16-alpine
        environment:
          POSTGRES_USER: appuser
          POSTGRES_PASSWORD: apppass
          POSTGRES_DB: appdb
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
          interval: 5s
          timeout: 3s
          retries: 10
        volumes:
          - db_data:/var/lib/postgresql/data
        networks: [backend]

      # Optional cache. Every profile that runs 'worker' must also run 'cache',
      # because worker depends_on cache.
      cache:
        image: redis:7-alpine
        volumes:
          - redis_data:/data
        command: ["redis-server", "--appendonly", "yes"]
        networks: [backend]
        profiles: ["dev", "qa", "prod"]

      # Background worker (optional), consumes jobs from Redis
      worker:
        build: ../worker
        environment:
          DATABASE_URL: postgresql://appuser:apppass@db:5432/appdb
          REDIS_URL: redis://cache:6379/0
        depends_on:
          db:
            condition: service_healthy
          cache:
            condition: service_started
        networks: [backend]
        profiles: ["dev", "qa", "prod"]

      # Dev-only helpers
      mailhog:
        image: mailhog/mailhog:v1.0.1
        ports:
          - "8025:8025"
        networks: [backend]
        profiles: ["dev"]

      adminer:
        image: adminer:4
        ports:
          - "8081:8080"
        networks: [backend]
        profiles: ["dev", "qa"]

.. warning::

   If a service in an active profile ``depends_on`` a service that is **not** active,
   Compose refuses to start the project at all:
   ``service "worker" depends on undefined service "cache": invalid compose project``.
   That is why ``cache`` lists every profile that ``worker`` does.

.. tip::

   MailHog has not had a release since 2020. `Mailpit <https://mailpit.axllent.org/>`_
   (image ``axllent/mailpit``) is a maintained replacement with the same web UI port
   (8025) and SMTP port (1025).

Below, we define profile-specific overrides for development and production environments.
In the **development override**, we mount the application source code into the container
and enable Flask's debug mode for live reloading. In the **production override**, we set
resource limits and restart policies for the services.

Development Profile
-------------------

The development profile includes live-reload capabilities and developer tools like
Mailhog and Adminer.

.. graphviz::
    :caption: Development Profile Architecture
    :align: center

    digraph dev {
      rankdir=LR;
      fontsize=12;
      fontname="Arial";
      node [shape=box];

      web -> app [label="depends_on"];
      app -> db [label="DATABASE_URL"];
      app -> cache [label="REDIS_URL"];
      worker -> db [label="DATABASE_URL"];
      worker -> cache [label="REDIS_URL"];
      db -> db_data [label="volume"];
      cache -> redis_data [label="volume"];

      subgraph cluster_backend {
        label="Backend Network";
        style=dashed;
        app [shape=oval, style=filled, fillcolor=yellow];
        db [shape=oval, style=filled, fillcolor=yellow];
        cache [shape=box, style="filled,dotted", fillcolor=lightgrey, label="cache"];
        worker [shape=box, style="filled,dotted", fillcolor=lightgrey, label="worker"];

        mailhog [shape=box, style="filled,dotted", fillcolor=lightgrey, label="mailhog"];
        adminer [shape=box, style="filled,dotted", fillcolor=lightgrey, label="adminer"];

      }

      subgraph cluster_frontend {
        label="Frontend Network";
        style=dashed;
        web [shape=oval, style=filled, fillcolor=yellow];
      }

      // Volumes
      db_data [shape=cylinder, label="db_data", fillcolor=lightblue, style=filled];
      redis_data [shape=cylinder, label="redis_data", fillcolor=lightblue, style=filled];

    }

Dev override: ``compose/docker-compose.override.yml``

.. code-block:: yaml
    :linenos:

    services:
      app:
        # Mount local source code into the container for live editing
        volumes:
          - ../app/src:/app/src
        command: python -m flask --app src/app.py run --debug --host=0.0.0.0 --port=5000


Production Profile
------------------

The production profile includes resource limits and restart policies for critical
services.

.. graphviz::
    :caption: Production Profile Architecture
    :align: center

    digraph prod {
      rankdir=LR;
      fontsize=12;
      fontname="Arial";
      node [shape=box];

      web -> app [label="depends_on"];
      app -> db [label="DATABASE_URL"];
      app -> cache [label="REDIS_URL"];
      worker -> db [label="DATABASE_URL"];
      worker -> cache [label="REDIS_URL"];
      db -> db_data [label="volume"];
      cache -> redis_data [label="volume"];

      subgraph cluster_backend {
        label="Backend Network";
        style=dashed;
        db [shape=oval, style=filled, fillcolor=yellow];
        app [shape=oval, style=filled, fillcolor=yellow];
        cache [shape=box, style="filled,dotted", fillcolor=lightgray];
        worker [shape=box, style="filled,dotted", fillcolor=lightgray];
      }

      subgraph cluster_frontend {
        label="Frontend Network";
        style=dashed;
        web [shape=oval, style=filled, fillcolor=yellow];
      }

      // Volumes
      db_data [shape=cylinder, label="db_data", fillcolor=lightblue, style=filled];
      redis_data [shape=cylinder, label="redis_data", fillcolor=lightblue, style=filled];
    }

Prod override: ``compose/docker-compose.prod.yml``

.. code-block:: yaml
    :linenos:

    services:
      app:
        # Set resource limits and restart policy for production.
        # Note that live-reload and bind mounts are omitted here.
        deploy:
          resources:
            limits:
              cpus: "1.0"
              memory: "512M"
        restart: unless-stopped
      worker:
        deploy:
          resources:
            limits:
              cpus: "0.5"
              memory: "256M"
        restart: unless-stopped
      web:
        restart: unless-stopped


QA Profile
----------

The QA profile includes the worker (and the cache it depends on) plus Adminer for
database management, but omits developer live-reload features.

QA override: none. Run the **base file only** (``-f docker-compose.yml``) so the dev
override is not applied automatically.

.. graphviz::
    :caption: QA Profile Architecture
    :align: center

    digraph qa {
      rankdir=LR;
      fontsize=12;
      fontname="Arial";
      node [shape=box];

      web -> app [label="depends_on"];
      app -> db [label="DATABASE_URL"];
      app -> cache [label="REDIS_URL"];
      worker -> db [label="DATABASE_URL"];
      worker -> cache [label="REDIS_URL"];
      db -> db_data [label="volume"];
      cache -> redis_data [label="volume"];

      subgraph cluster_backend {
        label="Backend Network";
        style=dashed;
        app [shape=oval, style=filled, fillcolor=yellow];
        db [shape=oval, style=filled, fillcolor=yellow];
        cache [shape=box, style="filled,dotted", fillcolor=lightgrey, label="cache"];
        worker[shape=box, style="filled,dotted", fillcolor=lightgrey, label="worker"];
        adminer [shape=box, style="filled,dotted", fillcolor=lightgrey, label="adminer"];
      }

      subgraph cluster_frontend {
        label="Frontend Network";
        style=dashed;
        web [shape=oval, style=filled, fillcolor=yellow];
      }

      // Volumes
      db_data [shape=cylinder, label="db_data", fillcolor=lightblue, style=filled];
      redis_data [shape=cylinder, label="redis_data", fillcolor=lightblue, style=filled];

    }


Run with profiles (from ``compose-labs/compose/``):

.. code-block:: bash
    :linenos:

    cd compose

    # Dev profile: base + dev override (auto-applied) + cache, worker, tools
    docker compose --profile dev up -d

    # QA profile: base file ONLY, so the dev override is not merged in
    docker compose -f docker-compose.yml --profile qa up -d

    # Prod: base + prod overrides (naming files explicitly also skips the dev override)
    docker compose --profile prod -f docker-compose.yml -f docker-compose.prod.yml up -d

    # Alternatively, select profiles via an environment variable:
    COMPOSE_PROFILES=dev,obs docker compose up -d

    # Stop everything, including services from profiles (see note)
    docker compose --profile '*' down

.. note::

   - Services **without** ``profiles`` run by default.
   - A service **with** a ``profiles`` list runs only when **any** listed profile is
     active.
   - Explicitly starting a profiled service (``docker compose up worker``) activates it
     and its dependencies.
   - Commands only act on services in **active** profiles. A plain
     ``docker compose down`` leaves profiled containers (and their network) running;
     use ``docker compose --profile '*' down`` to stop everything.


Testing and One-Off Commands
============================

**Objective:** Run commands inside services.

Next, we will demonstrate how to run one-off commands inside our services using Docker
Compose. This is useful for tasks such as running tests, database migrations, or
executing administrative commands.

Test tooling is not part of the runtime image. For this exercise, create a smoke test
(in ``compose-labs/``), mount it through the dev override, and install ``pytest`` on
demand:

.. code-block:: bash
    :linenos:

    mkdir -p tests
    cat > tests/test_smoke.py <<'PY'
    import json, urllib.request

    def test_health():
        # Runs inside the app container, where Flask listens on port 5000
        with urllib.request.urlopen("http://localhost:5000/api/health") as resp:
            assert json.load(resp)["status"] == "ok"
    PY

.. code-block:: yaml
    :linenos:

    # compose/docker-compose.override.yml (add to the existing app service)
    services:
      app:
        volumes:
          - ../app/src:/app/src
          - ../tests:/app/tests

.. code-block:: bash
    :linenos:

    # Run tests inside the app container (dev override active)
    docker compose exec app sh -c "pip install -q pytest && python -m pytest -q tests"

    # Check Postgres tables
    docker compose exec db psql -U appuser -d appdb -c "\dt"

    # Run a one-off command in a fresh worker container
    docker compose --profile dev run --rm worker python -c "print('hello from worker')"

.. tip::

   For real projects, bake test dependencies into a dedicated ``test`` build stage
   (``build: {context: ../app, target: test}``) instead of installing them at run time.


Observability (Optional Profile)
================================

**Objective:** Add a lightweight metrics stack under the ``obs`` profile.

A lightweight observability stack can help monitor the health and performance of your
services. We will add Prometheus for metrics collection and Grafana for visualization,
both behind an ``obs`` profile.

1. Create the Prometheus configuration in ``compose/`` **before** starting the stack. If
   the file is missing, Docker silently creates a *directory* at that path and Prometheus
   fails to start.

   .. code-block:: bash
       :linenos:

       cat > prometheus.yml <<'YAML'
       global:
         scrape_interval: 15s
       scrape_configs:
         - job_name: prometheus
           static_configs:
             - targets: ["localhost:9090"]
       YAML

2. Add these two services **under the existing** ``services:`` **key** of
   ``compose/docker-compose.yml`` (pasting a second ``services:`` key fails with
   ``mapping key "services" already defined``):

   .. code-block:: yaml
       :linenos:

       # Collect metrics with Prometheus
       prometheus:
         image: prom/prometheus:v2.55.0
         volumes:
           - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
         ports:
           - "9090:9090"
         networks: [backend]
         profiles: ["obs"]

       # Visualize metrics with Grafana
       grafana:
         image: grafana/grafana:11.2.0
         ports:
           - "3000:3000"
         networks: [backend]
         profiles: ["obs"]

3. Run:

   .. code-block:: bash

      docker compose --profile obs up -d

**Expected Outcome:** Prometheus at http://localhost:9090 (Status → Targets shows the
``prometheus`` job as UP) and Grafana at http://localhost:3000.


Ship a Production-Like Stack
============================

**Objective:** Use resource limits, restart policies, externalized configuration, and
minimized images.

To prepare your Docker Compose stack for production, consider the following best
practices:

Checklist
---------

- Pin images to **specific tags** (e.g., ``nginx:1.27-alpine``).
- Use ``--build-arg`` and multi-stage builds.
- Mount read-only volumes where possible.
- Configure ``restart: unless-stopped`` for long-running services.
- Keep secrets out of YAML: use ``secrets:`` (file-based in plain Compose) or your
  orchestrator's secret store.
- Remember that override files *append* to lists; use ``!override`` / ``!reset`` when a
  production override must replace ports, volumes, or commands.


Profiles - II
=============

What are profiles?
------------------

**Profiles** let you conditionally include services in your application model. They
enable you to keep a **single Compose file** (with optional overrides) and activate
subsets of services for different scenarios, such as:

- ``dev``: developer conveniences (hot reload, admin tools, Mailhog)
- ``qa``: worker + DB admin, but no dev auto-reload
- ``prod``: caches, workers, hard resource limits
- ``obs``: metrics, dashboards, and log viewers

Defining profiles
-----------------

- Add ``profiles: ["name1", "name2"]`` **under the service** that should be optional.
- Services **without** a ``profiles`` key are always active.

Activating profiles
-------------------

- CLI: ``docker compose --profile <name> up -d`` (repeat flag for multiple)
- Env var: ``COMPOSE_PROFILES=name1,name2 docker compose up -d``
- All profiles: ``docker compose --profile '*' <command>`` (useful for ``down``)

Selection Rules (practical tips)
--------------------------------

- Activating a profile enables *all* services listing that profile.
- If you explicitly start a service, Compose starts its dependencies *even if the
  dependencies belong to other profiles*.
- If a service in an active profile depends on a service in an *inactive* profile,
  Compose rejects the project; give dependencies every profile their dependents have.
- Keep "core" dependencies (e.g., ``db``) **profile-less** to avoid surprises.
- Put tooling (e.g., Adminer, Mailhog, Grafana) **behind a profile**.
- Use profile-specific overrides for ports/mounts to avoid conflicts.


Troubleshooting
===============

- **Validate first**: ``docker compose config`` catches indentation, duplicate keys, and
  undefined dependencies before anything starts.
- **Port conflicts**: Move dev helpers behind profiles and only activate when needed.
- **Healthchecks**: Prefer ``CMD-SHELL`` with clear timeouts/retries and a
  ``start_period``; don't make them too strict. Healthchecks run *inside* the container,
  so target the container port, not the published host port.
- **Rebuilds**: Use ``--build`` when you change Dockerfiles or dependencies.
- **Caching**: Bind mounts bypass image layers; if behavior differs between dev/prod,
  test with bind mounts off.
- **Images**: Pin to specific tags; avoid ``latest`` in production.
- **Profiles**: Keep core infrastructure (``db``) profile-less; optional tooling behind
  profiles; use ``--profile '*' down`` to stop everything.
- **Overrides**: Lists merge by appending; check ``docker compose config`` and use
  ``!override`` to replace.
- **Variables**: ``${VAR}`` in a Compose file is substituted from your shell and
  ``.env`` *when the file is parsed*, not from the service's ``environment:`` block. To
  pass a literal ``$`` through to the container's shell, write ``$$``.


References
==========

- `Docker Compose documentation <https://docs.docker.com/compose/>`_
- `Compose file reference <https://docs.docker.com/reference/compose-file/>`_
- `Merging Compose files (!override and !reset) <https://docs.docker.com/compose/how-tos/multiple-compose-files/merge/>`_
- `Using profiles with Compose <https://docs.docker.com/compose/how-tos/profiles/>`_
- `Dockerfile reference <https://docs.docker.com/reference/dockerfile>`_
- `Flask documentation <https://flask.palletsprojects.com/en/latest/>`_
- `PostgreSQL documentation <https://www.postgresql.org/docs/>`_
- `Redis documentation <https://redis.io/docs/latest/>`_
- `Prometheus documentation <https://prometheus.io/docs/introduction/overview/>`_
- `Grafana documentation <https://grafana.com/docs/grafana/latest/>`_
