---
title: "Running Postgres Locally for Back-end Development Using Docker"
type: "post"
date: 2023-05-13T11:36:39+05:30
tags: [docker, postgres. "developer tools"]
draft: false
---

When doing back-end development or just fiddling with SQL, installing postgres the traditional way may not be very convenient for many reasons. This post describes how to run postgres for local development using docker and docker compose.

One important advantage for me is getting the latest version of postgres, since I tend to use a stable linux distribution.

There's already some amount of documentation for postgresql official docker image. But few things are less than obvious if you're a beginner like me.

## Running postgres as standalone docker container

First pull the docker container using

```bash
docker pull postgres
```

Only required configuration for running this image is POSTGRES_PASSWORD. We want to assign a name to this for ease of referring to it later. So this image can be run as

```bash
docker run --name postgres_db -e POSTGRES_PASSWORD=mypassword -d postgres
```

This is the command in official documentation. But this doesn't expose the postgres port. So you can only access it through psql in same container. (Note the default user is `postgres`, and default db name is also `postgres`. It can be overridden with environment variables `POSTGRES_USER` and `POSTGRES_DB` respectively).

This command gives a postgresql prompt, where you can run commands.

```bash
docker exec -it postgres_db psql -U postgres
```

But to actually use this database from application code (or other clients), we need to expose the port 5432. Remove the old `postgres_db` container using:

```bash
docker stop postgres_db && docker rm postgres_db
```

Now run the same command with port forwarding.
```bash
docker run --name postgres_db -e POSTGRES_PASSWORD=mypassword -dp 127.0.0.1:5432:5432 postgres
```

This allows connecting to this database with external clients (For example [pgcli](https://github.com/dbcli/pgcli), which has nice features like autocompletion, syntax highlighting and various output formats).

More importantly, you can access this database from application code such as JDBC or Golang's `pq` driver.

Now you can connect to it using pgcli, and can use the same credentials in application code as well.

```
pgcli -h localhost -u postgres -d postgres
```

Again, the default database name is `postgres` and it can be customized with `POSTGRES_DB` variable, eg: `-e POSTGRES_DB=books`.

## Using volumes for persistent storage

Above technique does not save data when container is removed using `docker rm`. For that, we have to use volumes.

First create a volume using

```bash
docker volume create postgres-data
```

Pass this volume when running docker command

```bash
docker run --name postgres_db -e POSTGRES_PASSWORD=mypassword -dp 127.0.0.1:5432:5432\
	-v postgres-data:/var/lib/postgresql/data postgres
```

## Using docker compose
All above steps look a little manual, because they are.

We can use docker compose instead of specifying long commands every time. It also makes it easy to use other tools. To use docker compose, create a `docker-compose.yml` file with this configuration:

```yaml
version: '3.9'

services:
  postgres:
    image: postgres
    environment:
      POSTGRES_PASSWORD: mysecretpassword
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - 127.0.0.1:5432:5432

volumes:
  postgres-data:
```

Replace `mysecretpassword` by a password for local database. You can also refer to an environment variable in the configuration, which is arguably better than putting password in YAML.

```yaml
	environment:
		POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
		## run `export POSTGRES_PASSWORD=mypassword` in shell
```

After creating config, postgres server can be created by running the following command in the same directory as `docker-compose.yaml`.

```bash
docker compose up -d
```

To stop and remove the containers:
```bash
docker compose down
```

To delete storage volumes associated with the container, you should run:

```bash
docker compose down --volumes
```

## Miscellaneous

Golang `pq` driver requires to disable SSL explicitly. Add `sslmode=disable` in connection string when applicable to local setup.

Official docs also show using `adminer` in same compose file. Adminer (formerly `PhpMyAdmin`) is a popular web UI for many databases including postgres. To run adminer, add another service to the compose file. It will be available on `localhost:8080`.

```yaml
version: '3.9'

services:
  ## ..postgres service from above config

  adminer:
    image: adminer
    restart: always
    environment:
      ADMINER_DEFAULT_SERVER: postgres
    ports:
      - 127.0.0.1:8080:8080

## .. volumes from above config
```

Many things can be configured using environment variables, refer to [official documentation](https://hub.docker.com/_/postgres) for more details.
