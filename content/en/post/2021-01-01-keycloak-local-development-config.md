---
title: "Keycloak Local Development Config"
date: 2021-01-01T18:42:16+01:00
draft: false
---

Recently I found myself setting up a local development environment where Keycloak is used.
I've created a [repo](https://github.com/dovidio/keycloak-postgres-local-development) that can be used as a reference when needed in the future.
<!--more-->
Keycloak offers a docker [container](https://hub.docker.com/r/jboss/keycloak/) with several configuration options. I decided to run Keycloak with Postgres. Here's my docker-compose file

```yaml
version: "3.8"
services:
  db:
    image: postgres:12
    restart: always 
    ports:
      - 5432:5432
    volumes:
      - /c/db:/var/lib/postgresql/data
      # This will bind the files inside the pgscripts to docker-entrypoint-initdb.d
      # The scripts will be run on startup
      - $PWD/postgres:/docker-entrypoint-initdb.d
    env_file:
      - .env.dev
  wait-for-db:
    image: dadarek/wait-for-dependencies
    depends_on:
      - db 
    command: db:5432
  keycloak:
    image: jboss/keycloak
    ports:
      - 8080:8080
    env_file:
      - .env.dev
```

To make things more interesting, I've added a boostrap script for Postgres that creates a separate database dedicated to Keycloak, which allows for a nice separation in case later on we want to reuse the same Postgres instance for some other application.
```bash
#!bin/sh
psql << EOF 
CREATE USER $DB_USER WITH PASSWORD '$DB_PASSWORD';
CREATE DATABASE $DB_DATABASE OWNER $DB_USER;
EOF
```
Note that the bootstrap script is using the [here document](https://en.wikipedia.org/wiki/Here_document) in order to access environment variables with sql statements. A neat trick!

Another interesting thing is that all the environment variables configuration is done in a separate env file, keeping our docker-compose file cleaner.
To run the app, the docker-compose file I've created a two liner script
```bash
docker-compose run --rm  wait-for-db
docker-compose up -d keycloak 
```
This first run wait-for-db. Once that command exit, we are sure that Postgres is ready to accept connections, so we can then launch Keycloak. If we wouldn't do this, Keycloak would fail on startup since it cannot connect to Posgres.

