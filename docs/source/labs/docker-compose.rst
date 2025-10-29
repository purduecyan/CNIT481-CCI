**************
Docker Compose
**************

.. contents::
   :local:
   :depth: 2

Introduction
============
The Docker Compose tool allows you to define and manage multi-container Docker applications using simple YAML files. This article guides you through creating a robust development and production-ready stack using Docker Compose, covering essential features like service definitions, networking, volumes, healthchecks, scaling, and profiles.

- Define multi-service stacks
- Wire up networks, volumes, healthchecks, and dependencies
- Use **profiles** for dev/test/observability extras
- Scale services
- Prepare a production-oriented override

Prerequisites
-------------

- Docker Engine and Docker Compose (``docker compose`` CLI) installed
- Familiarity with Docker images/containers, volumes, and networks)


Example Project Structures
--------------------------

Minimal (two services)
^^^^^^^^^^^^^^^^^^^^^^

This example structure includes a simple web app and Nginx reverse proxy.

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
   ├── .env
   └── docs/
       └── index.rst


Full (dev + prod, profiles, tests, CI)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This example structure includes additional services, profiles, and testing setup.

.. code-block:: text

   compose-labs/
   ├── app/
   │   ├── Dockerfile
   │   ├── requirements.txt
   │   └── src/
   │       ├── app.py
   │       └── db.py
   ├── worker/
   │   ├── Dockerfile
   │   └── worker.py
   ├── nginx/
   │   └── default.conf
   ├── compose/
   │   ├── docker-compose.yml              # Base stack (web, app, db, cache)
   │   ├── docker-compose.override.yml     # Dev overrides (bind mounts, watch, debug)
   │   ├── docker-compose.prod.yml         # Prod overrides (limits, restart, secrets)
   │   └── .env.example
   ├── migrations/
   ├── tests/
   │   └── test_smoke.py
   ├── .env
   ├── Makefile
   └── docs/
       ├── conf.py
       └── index.rst


First Compose Stack
===================

**Objective:** Define a single-container service (Nginx) with port mapping.

In this exercise, we will create a simple Docker Compose file that defines a single service running Nginx, a popular web server. We will map port 8080 on the host to port 80 in the container so that we can access the Nginx welcome page from our browser.

1. Create the folder and base file:

    .. code-block:: bash
        :linenos:

        # Create a directory named 'compose-labs' if it doesn't exist
        mkdir -p compose-labs && cd compose-labs

        # Create a new file named 'docker-compose.yml' and write the following YAML content into it
        cat > docker-compose.yml <<'YAML'
        services:                   # Define the services for Docker Compose
            web:                    # Service name: 'web'
            image: nginx:latest     # Use the latest official NGINX image from Docker Hub
            ports:
                - "8080:80"         # Map host port 8080 to container port 80 (HTTP)
        YAML

2. Up the stack:

   .. code-block:: bash

      docker compose up -d
      docker compose ps


**Expected Outcome:** Open http://localhost:8080 and see the Nginx welcome page.


Add an App Service
==================

**Objective:** Add a Python API and a reverse proxy.

Next, we will add a simple Python Flask application that serves a health check endpoint. We will also configure Nginx to act as a reverse proxy, forwarding requests from port 80 to the Flask app running on port 5000.

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
        CMD ["python", "src/app.py"]
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

        # Run the app on all network interfaces at port 5000
        if __name__ == "__main__":
            # Enable external access when running in Docker
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
            # Use the official NGINX image (latest tag)
            image: nginx:latest
            
            # Mount configuration files into the container and map local nginx/default.conf to container path (read-only)
            volumes:
              - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro

            # Publish and expose container port 80 on host port 8080
            ports:
              - "8080:80"

            # Specify startup order for services. Ensure the 'app' service is started before 'web'
            depends_on:
              - app

          # Backend application service
          app:
            # Build the image from the Dockerfile in ./app
            build: ./app

            # Make port available to other services on the Compose network. Internal port used by the app (not published to host)
            expose:
              - "5000"

4. Up:

   .. code-block:: bash

      docker compose up -d --build
      curl -s http://localhost:8080/api/health

**Expected Outcome:** ``{"status":"ok"}`` is returned via Nginx → app.


Network, Environment, and Volumes
=================================

**Objective:** Add Postgres with a named volume; wire up app via environment variables and a user-defined network.

At this point, we have three services: ``web``, ``app``, and ``db`` (Postgres). We will create a private network for backend communication, persist Postgres data with a named volume, and configure the app to connect to the database using environment variables. Additionally, we will set up service dependencies to ensure proper startup order. The services will be able to communicate with each other using their service names as hostnames.

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
            # Start app only after db is ready
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


Scaling the App
===============

**Objective:** Run multiple replicas of ``app``.

Next, we will scale the ``app`` service to run multiple instances. This is useful for load balancing and improving the availability of our application. We will use Docker Compose's scaling feature to achieve this.

.. code-block:: bash

   docker compose up -d --scale app=3

.. note::

   Nginx is a simple reverse proxy here and does not do health-based load balancing; in this setup it will round-robin across IPs known on the network.


Dev Overrides and Live Code
===========================

**Objective:** Use ``docker-compose.override.yml`` to mount source and enable an auto-reloader.

In development, it's common to want to see code changes reflected immediately without rebuilding images. We will create a Docker Compose override file that mounts the application source code into the container and enables Flask's debug mode for live reloading.

Example Workflow
----------------

Create ``docker-compose.override.yml`` (auto-applied):

.. code-block:: yaml
    :linenos:

    services:
      app:
        # Mount local source code into the container for live editing
        volumes:
          - ./app/src:/app/src
        # Enable Flask debug mode and auto-reload
        command: python -m flask --app src/app.py run --debug --host=0.0.0.0 --port=5000
        environment:
          FLASK_DEBUG: "1"

Restart:

.. code-block:: bash

   docker compose up -d --build

Edit files under ``app/src`` and refresh your browser.


Profiles - I
============

**Objective:** Use **profiles** to conditionally include services and developer tools.

Profiles allow you to define optional services that can be activated based on the environment or use case. As an example, we will add a Redis cache, a background worker, and developer tools like Mailhog and Adminer, each behind specific profiles. Mailhog is a web-based email testing tool, and Adminer is a lightweight database management tool. The profiles will help us keep the base stack clean while enabling additional functionality as needed.

- **Production Profile:** Includes Redis cache and worker for background tasks.
- **Development Profile:** Adds Mailhog for email testing and Adminer for database management.
- **QA Profile:** Includes Adminer for database management.


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
      cache [shape=box, style="filled,dotted", fillcolor=lightgrey, label="cache\ndev, prod"];

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



Base file: ``compose/docker-compose.yml``

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

      # Optional cache for performance (enabled in dev and prod)
      cache:
        image: redis:7-alpine
        volumes:
          - redis_data:/data
        command: ["redis-server", "--appendonly", "yes"]
        networks: [backend]
        profiles: ["dev", "prod"]

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

Below, we define profile-specific overrides for development and production environments. In the **development override**, we mount the application source code into the container and enable Flask's debug mode for live reloading. In the **production override**, we set resource limits and restart policies for the services.

Development Profile
-------------------

The development profile includes live-reload capabilities and developer tools like Mailhog and Adminer.

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
        environment:
          FLASK_DEBUG: "1"
        command: python -m flask --app src/app.py run --debug --host=0.0.0.0 --port=5000


Production Profile
------------------
The production profile includes resource limits and restart policies for critical services.

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

The QA profile includes Adminer for database management and the worker service, but omits developer live-reload features.

QA override: N/A (uses base + adminer + worker profiles)

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
      worker -> db [label="DATABASE_URL"];
      db -> db_data [label="volume"];

      subgraph cluster_backend {
        label="Backend Network";
        style=dashed;
        app [shape=oval, style=filled, fillcolor=yellow];
        db [shape=oval, style=filled, fillcolor=yellow];
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

    }


Run with profiles:

.. code-block:: bash

   # Dev profile (includes cache, worker, tools)
   cd compose
   docker compose --profile dev up -d

   # QA profile (worker + adminer, but not developer live-reload if you omit override)
   docker compose --profile qa up -d

   # Prod base + prod overrides
   docker compose --profile prod -f docker-compose.yml -f docker-compose.prod.yml up -d

   # Alternatively via env var:
   COMPOSE_PROFILES=dev,observability docker compose up -d

.. note::

   - Services **without** ``profiles`` run by default.
   - A service **with** a ``profiles`` list runs only when **any** listed profile is active.
   - Starting a specific service may pull in its dependencies even if those dependencies carry different profiles.


Testing and One-Off Commands
============================

**Objective:** Run commands inside services.

Next, we will demonstrate how to run one-off commands inside our services using Docker Compose. This is useful for tasks such as running tests, database migrations, or executing administrative commands.


.. code-block:: bash
    :linenos:

    # Run tests inside the app container
    docker compose exec app pytest -q

    # Check Postgres tables
    docker compose exec db psql -U appuser -d appdb -c "\dt"

    # Run a one-off command in the worker container
    docker compose run --rm worker python -c "print('hello from worker')"


Observability (Optional Profile)
================================

**Objective:** Add a lightweight metrics stack under ``observability`` profile.

A lightweight observability stack can help monitor the health and performance of your services. We will add Prometheus for metrics collection and Grafana for visualization, both behind an ``observability`` profile.

Snippet (to append in ``docker-compose.yml``):

.. code-block:: yaml
    :linenos:

    services:
      # Collect metrics with Prometheus
      prometheus:
        image: prom/prometheus:v2.55.0
        volumes:
          - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
        ports:
          - "9090:9090"
        networks: [backend]
        profiles: ["observability"]

      # Visualize metrics with Grafana
      grafana:
        image: grafana/grafana:11.2.0
        ports:
          - "3000:3000"
        networks: [backend]
        profiles: ["observability"]

Run:

.. code-block:: bash

   docker compose --profile observability up -d


Ship a Production-Like Stack
============================

**Objective:** Use resource limits, restart policies, externalized configuration, and minimized images.

To prepare your Docker Compose stack for production, consider the following best practices:

Checklist
---------

- Pin images to **specific tags** (e.g., ``nginx:1.27-alpine``).
- Use ``--build-arg`` and multi-stage builds.
- Mount read-only volumes where possible.
- Configure ``restart: unless-stopped`` for long-running services.
- Store secrets in environment files mounted as files, or Docker secrets if Swarm/Kubernetes.


Profiles - II
=============

What are profiles?
------------------

**Profiles** let you conditionally include services in your application model. They enable you to keep a **single Compose file** (with optional overrides) and activate subsets of services for different scenarios, such as:

- ``dev``: developer conveniences (hot reload, admin tools, Mailhog)
- ``qa``: worker + DB admin, but no dev auto-reload
- ``prod``: caches, workers, hard resource limits
- ``observability``: metrics and dashboards

Defining profiles
-----------------

- Add ``profiles: ["name1", "name2"]`` **under the service** that should be optional.
- Services **without** a ``profiles`` key are always active.

Activating profiles
-------------------

- CLI: ``docker compose --profile <name> up -d`` (repeat flag for multiple)
- Env var: ``COMPOSE_PROFILES=name1,name2 docker compose up -d``

Selection Rules (practical tips)
--------------------------------
- Activating a profile enables *all* services listing that profile.
- If you explicitly start a service, Compose starts its dependencies *even if the dependencies belong to other profiles*.
- Keep "core" dependencies (e.g., ``db``) **profile-less** to avoid surprises.
- Put tooling (e.g., Adminer, Mailhog, Grafana) **behind a profile**.
- Use profile-specific overrides for ports/mounts to avoid conflicts.


Troubleshooting
===============

- **Port conflicts**: Move dev helpers behind profiles and only activate when needed.
- **Healthchecks**: Prefer ``CMD-SHELL`` with clear timeouts/retries; don’t make them too strict.
- **Rebuilds**: Use ``--build`` when you change Dockerfiles or dependencies.
- **Caching**: Bind mounts bypass image layers; if behavior differs between dev/prod, test with bind mounts off.
- **Images**: Pin to specific tags; avoid ``latest`` in production.
- **Profiles**: Keep core infrastructure (``db``) profile-less; optional tooling behind profiles.

References
==========

- `Docker Compose documentation <https://docs.docker.com/compose/>`_
- `Compose file reference <https://docs.docker.com/compose/compose-file/>`_
- `Dockerfile reference <https://docs.docker.com/reference/dockerfile>`_
- `Flask documentation <https://flask.palletsprojects.com/en/latest/>`_
- `PostgreSQL documentation <https://www.postgresql.org/docs/>`_
- `Redis documentation <https://redis.io/docs/latest/>`_
- `Prometheus documentation <https://prometheus.io/docs/introduction/overview/>`_
- `Grafana documentation <https://grafana.com/docs/grafana/latest/>`_
