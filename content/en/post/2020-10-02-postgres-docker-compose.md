---
title: "Postgres configuration with Docker Compose"
date: 2020-10-02T16:00:00+02:00
draft: false
tags: ["docker-compose", "postgres"]
---

You want to run Postgres inside a Docker container for development? Read here
<!--more-->
Chances are that you will run it alongside other applications that will access it.
The best tool you can use for local development is docker-compose
Here's my configuration
```
version: "3.8"
services:
  db:
    image: postgres:12
    restart: always 
    ports:
      - 5432:5432
    volumes:
      - /c/db:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```
Nothing too fancy, but there are a couple of things that are good to keep in mind

1. You can use volumes to avoid loosing data. Here I'm binding the db directory of my drive to the postgresql/data container directory. Moreover I'm binding an **init.sql** file that resides in the
same directory of the docker-compose file to *docker-entrypoint-initdb.d/init.sql*. Inside this script
I do the initial user and db creation. The file is called by postgres when starting up.
```
create user if not exists docker;
create database if not exists app;
grant all privileges on database app to docker;
```
2. The password is provided as an environment variable. When launching the docker-compose script, you
can specify to use an env file to create environment variables for the container. E.g.
```
docker-compose --env-file .env.dev --project-name quarkus-boilerplate up -d
```
In this way you can have multiple environment files for different environments.

That's it!