# Installation

To use foxops for managing your Git repositories, you need to host the foxops API server (which also hosts the web UI).

## Local Setup (Development Only)

If you only want to test out foxops on your local machine to see how it works, you can use the following commands to get started in less than a minute.

First, let's prepare a local SQLite Database to store the foxops data. The following will create a folder `foxops_db` at your current location, containing an initialized foxops database:

```bash
docker run --rm -v "$(pwd)"/foxops_db:/database \
    -e FOXOPS_DATABASE_URL=sqlite+aiosqlite:////database/foxops.sqlite \
    ghcr.io/roche/foxops:latest \
    alembic upgrade head
```

With the database in place, we can now start the foxops API server. Two options exist for running it locally:

### Run FoxOps Connected to an Existing Gitlab Instance

If you already have a running Gitlab instance where you want to host your templates and incarnations, you can run foxops connected to that instance:

```bash
export FOXOPS_HOSTER_GITLAB_ADDRESS=https://gitlab.com
export FOXOPS_HOSTER_GITLAB_TOKEN=glpat-abcdefgh123456

docker run --rm -p 8000:8000 -v "$(pwd)"/foxops_db:/database \
    -e FOXOPS_DATABASE_URL=sqlite+aiosqlite:////database/foxops.sqlite \
    -e FOXOPS_STATIC_TOKEN=dummy-token \
    -e FOXOPS_HOSTER_TYPE=gitlab \
    -e FOXOPS_HOSTER_GITLAB_ADDRESS=$FOXOPS_HOSTER_GITLAB_ADDRESS \
    -e FOXOPS_HOSTER_GITLAB_TOKEN=$FOXOPS_HOSTER_GITLAB_TOKEN \
    ghcr.io/roche/foxops:latest
```

### Run FoxOps Without Gitlab

For the very first steps (development only), foxops can also be run without a Gitlab instance connected. This is not only useful for getting started quickly, but also when running tests in other systems that require a running foxops instance.

Be aware that Gitlab in this is the replaced with a very basic implementation that creates all repositories/merge requests in a local folder. There is no UI for viewing or interacting with these repositories, except for your file browser.

```bash
# put a folder here where foxops should create the repositories
export FOXOPS_HOSTER_LOCAL_DIRECTORY=/tmp/foxops_hoster

docker run --rm -p 8000:8000 -v "$(pwd)"/foxops_db:/database \
    -e FOXOPS_DATABASE_URL=sqlite+aiosqlite:////database/foxops.sqlite \
    -e FOXOPS_STATIC_TOKEN=dummy-token \
    -e FOXOPS_HOSTER_TYPE=local \
    -e FOXOPS_HOSTER_LOCAL_DIRECTORY=$FOXOPS_HOSTER_LOCAL_DIRECTORY \
    ghcr.io/roche/foxops:latest
```

```{note}
In this setup, your template repositories must also be placed in the `/tmp/foxops_hoster` folder. Place the git repositories of your templates in a folder called `/tmp/foxops_hoster/<template_name>/git` to make them usable by foxops.
```

### Accessing FoxOps

Then, you can open your browser and go to the following URLs:
* <http://localhost:8000/> - the foxops web UI (login with `dummy-token` when prompted for the password)
* <http://localhost:8000/docs> - the foxops API documentation

## Production Deployment

```{warning}
Be aware that foxops is still in an early stage of development.
```

To run foxops in production, you need a place where you can run Docker containers and expose the contained services. For example a Kubernetes cluster.

### Prerequisites

Foxops requires a PostgreSQL database to store its data. Prepare the connection string for that database as a [SQLAlchemy asyncpg URL](https://docs.sqlalchemy.org/en/14/core/engines.html#postgresql) that looks like this:

```text
postgresql+asyncpg://user:password@host:port/database
```

### Deployment

#### Prepare / Upgrade Database

Before starting the foxops API server, the database must be prepared (initially) or upgraded (after foxops update) to the correct database schema.

To do so, run the `ghcr.io/roche/foxops` container (with the version you want to deploy):
* Set the `FOXOPS_DATABASE_URL` environment variable to the SQLAlchemy database URL prepared above
* override the container command with `alembic upgrade head`.

You can either do this manually, or (recommended) run it as an init container when running foxops in Kubernetes.

#### Run foxops

The following is needed to run foxops:

* Run the `ghcr.io/roche/foxops` Docker image (consider picking a specific version instead of `latest`)
* Expose port `8000` of that container to the outside world (e.g. using a Kubernetes Ingress)
* Set the following environment variables:
  * `FOXOPS_STATIC_TOKEN` - Set to a (long) random and **secret** string. This secret is used to authenticate all users of the foxops UI & API
  * `FOXOPS_DATABASE_URL` - Set to the database URL
  * `FOXOPS_HOSTER_TYPE` - Set to `gitlab` for a production deployment
  * `FOXOPS_HOSTER_GITLAB_ADDRESS` - Set to the address of your GitLab instance (e.g. `https://gitlab.com`) (if hoster type is set to `gitlab`)
  * `FOXOPS_HOSTER_GITLAB_TOKEN` - Set to a GitLab access token that has access to all repositories (incarnations & templates) that foxops should manage (if hoster type is set to `gitlab`)

#### Kubernetes Example

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: foxops-api
  labels:
    app: foxops
spec:
  replicas: 2
  selector:
    matchLabels:
      app: foxops
  template:
    metadata:
      labels:
        app: foxops
        version: v2.0.0
    spec:
      initContainers:
        - name: database-migrator
          image: ghcr.io/roche/foxops:v2.0.0
          imagePullPolicy: Always
          command: ["alembic", "upgrade", "head"]
          env:
          - name: FOXOPS_DATABASE_URL
            value: postgresql+asyncpg://user:password@database_host:5432/foxops
          securityContext:
            runAsUser: 2102
            runAsGroup: 2102
            fsGroup: 2102
            allowPrivilegeEscalation: false
      containers:
        - name: foxops
          image: ghcr.io/roche/foxops:v2.0.0
          env:
            - name: FOXOPS_HOSTER_TYPE
              value: gitlab
            - name: FOXOPS_HOSTER_GITLAB_ADDRESS
              value: https://gitlab.com
            - name: FOXOPS_HOSTER_GITLAB_TOKEN
              value: <dummy>
            - name: FOXOPS_STATIC_TOKEN
              value: <dummy>
            - name: FOXOPS_DATABASE_URL
              value: postgresql+asyncpg://user:password@database_host:5432/foxops
            - name: FOXOPS_LOG_LEVEL
              value: INFO
          ports:
          - containerPort: 8000

---
apiVersion: v1
kind: Service
metadata:
  name: foxops-api
spec:
  ports:
  - name: http
    targetPort: 8000
    port: 80
  selector:
    app: foxops

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: foxops-api
spec:
  rules:
  - host: foxops.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foxops-api
            port:
              number: 80
```
