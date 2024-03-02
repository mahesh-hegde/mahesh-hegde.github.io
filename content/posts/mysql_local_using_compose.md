---
title: "Ephemeral MySQL for local testing using Docker Compose"
date: 2023-10-24T15:31:43+05:30
draft: false
type: post
showTableOfContents: true
tags: [Docker, MySQL, "Developer tools"]
---

## Why?

Many times tests are run against a simple database like H2 or SQLite, whereas production uses a heavier DB like MySQL. However, its many times desirable to run testing with the real DB. For example, when the application makes use of DB-specific features or quirks, it's required for proper testing.

Once upon a time it took long time to setup MySQL through distribution package manager, and tests ideally need ephemeral environments. But now with containers, we can run the same DB in local. Plus, with docker-compose and Docker volumes, we can easily create and tear-down data volumes, giving us clean state when we need.

Many months ago I wrote a post about using Docker compose to run a local instance of PostgreSQL for testing and / or experimentation. Similarly, this post shows a configuration for locally running __MySQL__.

## Configuration

This should be in `docker-compose.yml`

```yaml
version: '3.9'

services:
  mysql:
    image: mysql
    environment:
      MYSQL_USER: myusername
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_DATABASE: appdb ## Use the name of Database used by application
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - 127.0.0.1:3306:3306

volumes:
  mysql_data:
```

I have kept the passwords as environment variables. But it shouldn't matter for most local environments if you hardcode them.

Replace the value of MYSQL_DATABASE by the DB name used by the application code.

Usage is fairly straightforward. Just make sure to run these commands in same directory as `docker-compose.yml` (or pass the file explicitly through `-f` flag).

```bash
## Set password as required, (if not hardcoded in YAML)
export MYSQL_PASSWORD=somePassword

## Start the MySQL server
docker compose up -d

## Connect to server using `mycli` (replace username and DB name accordingly)
mycli -D appdb -u myusername -h localhost -p "$MYSQL_PASSWORD"

## Stop the server
docker compose down

## Stop the server and delete the data
docker compose down --volumes
```

## Tips

* If an application's docker image needs to access this server, It can't use `localhost:3306` - since the running container is on different docker network.

  To work around this, attach it to the Docker network created by compose. You can get this network's name by running `docker network ls` and this network will be usually named like `${directory_of_compose_file}_default`. Once the application container is attached to this network, it can access the MySQL through hostname `mysql`.

  For example, an application container which takes DB configuration through environment can be run like this:

  ```bash
  docker run --network="mysql_compose_default" --hostname "myapp" --name "myapp" -dp 127.0.0.1:8000:80 -e MYSQL_USER="myuser" -e MYSQL_PASSWORD="mypassword" -e MYSQL_HOST=mysql "myapp-image:latest"
  ```

* Many settings like password are set when the storage volume is initialized for the first time. So you cannot change these settings without deleting the volumes and starting again.

* Finally, some DB drivers may try to connect to Unix Domain Socket instead of TCP socket if `localhost` is specified as DB host in connection strings. This domain socket will not exist because our MySQL is in docker. To avoid this behavior, try using the localhost IP (127.0.0.1) instead.
