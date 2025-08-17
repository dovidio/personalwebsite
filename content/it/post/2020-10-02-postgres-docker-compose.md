---
title: "Configurazione Postgres con Docker Compose"
date: 2020-10-02T16:00:00+02:00
draft: false
tags: ["docker-compose", "postgres"]
---

Vuoi usare Postgres all'interno di un Docker container? Leggi qua
<!--more-->
Probabilmente vorrai lanciare il database insieme ad altre applicazioni che vi accedono.
Il miglior strumento che puoi utilizzare per lo sviluppo locale e' senza dubbio Docker Compose.

Ecco una configurazione tipo per Postgres
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
Niente di troppo arcano, ma ci sono un paio di cose da tenere a mente

1. Puoi utilizzare i docker volumes per evitare di perdere i dati ogni volta che riavvi il container. In questo esempio la cartella db della mia macchina si collega alla cartella postgresql/data del container. Inoltre **init.sql** della mia macchina viene utilizzato da postgres all'avvio del container. Dentro a questo script si possono effettuare le primissime operazioni di configurazione,
come la creazione di utenti e database.
```
create user if not exists docker;
create database if not exists app;
grant all privileges on database app to docker;
```
2. La password viene collegata ad una variabile di ambiente. Quando si lancia docker-compose, si puo' specificare un file di variabili d'ambiente, come segue:
```
docker-compose --env-file .env.dev --project-name app up -d
```
In tal modo si possono avere piu' configurazioni per ambienti diversi.

E questo e' tutto!