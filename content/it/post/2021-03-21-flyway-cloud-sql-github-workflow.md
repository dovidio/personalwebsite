---
title: "Creare un Github Worflow per migrazioni Cloud SQL con Flyway"
date: 2021-03-21T10:42:16+01:00
draft: false
tags: ["flyway", "gcp", "cloudsql", "githubactions"]
---

Ho scritto recentemente un post su come settare la connessione tra Flyway e Google Cloud SQL. In questo post
spieghero' come create un Github Worflow che permetta di eseguire migrazioni di schema con un solo click.
<!--more-->
Ovviamente avremmo bisogno di un file yaml che descriva il workflow.
Questo file deve essere creato nella cartella `.github/workflows` della repository dalla quale creeremo le nostre migrazioni.
Ecco il codice
```yaml
name: Migrate
on: [workflow_dispatch]

jobs:
  migrate:
    runs-on: ubuntu-latest
    env:
      FLYWAY_URL: ${{ secrets.CLOUD_SQL_JDBC_URL }}
      FLYWAY_USER: ${{ secrets.CLOUD_SQL_USER }}
      FLYWAY_PASSWORD: ${{ secrets.CLOUD_SQL_PASSWORD }}

    steps:
      - uses: actions/checkout@v2
      - uses: google-github-actions/setup-gcloud@master
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}
          service_account_key: ${{ secrets.GCP_SA_KEY}}
          export_default_credentials: true
      - name: Set up JDK 1.11
        uses: actions/setup-java@v1
        with:
          java-version: 1.11
      - name: Get Flyway info
        run: mvn flyway:info
      - name: Migrate
        run: mvn flyway:migrate
```

Ho utilizzato `on: [workflow_dispatch]` in modo da poter eseguire il workflow manualmente.
Poi ho settato le variabli d'ambiente che specificano l'url della connessione jdbc e utente e password.
Poi utilizzo l'azione checkout per effettuare il scaricare il codice della repo nel runner che esegue il workflow.
Utiizzzo setup-gcloud per popolare la variabile d'ambiente `GOOGLE_APPLICATION_CREDENTIALS` necessaria per la connessione a Cloud SQL.
Il segreto `GCP_PROJECT_ID` deve contentere il tuo id progetto,
Mentre il segreto `GCP_SA_KEY` conterra' il tuo service account in formato json.
Infine utilizzo `setup-java` per la configurazione Java e Maven.
Fatto cio', posso eseguire i comandy flyway, come info o migrate.

Ecco fatto, ora puoi eseguire le tue migrazioni con un semplice click da github :)

![Github Action](/images/github-action.png)