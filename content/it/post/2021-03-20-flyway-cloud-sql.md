---
title: "Configurare Flyway e Google Cloud SQL"
date: 2021-03-20T10:42:16+01:00
draft: false
tags: ["flyway", "gcp", "cloudsql"]
---

Flyway e' un tool che permette di evolvere lo schema del database in maniera robusta. In questo post vedremo come si puo' connettere con Cloud SQL.

<!--more-->
Nel mio caso specifico, utilizzo Postgres e il flyway-maven-plugin per effettuare le mie migrazioni.
Ho deciso di connettere flyway utilizando il [Cloud SQL Proxy](https://cloud.google.com/sql/docs/postgres/connect-admin-proxy).
La connessione viene gestita dalla [cloud sql jdbc socket factory](https://github.com/GoogleCloudPlatform/cloud-sql-jdbc-socket-factory). Dato che uso Postgres,
ho dovuto aggiungere la seguente dipendenza al mio pom.xml
```xml
<dependency>
    <groupId>com.google.cloud.sql</groupId>
    <artifactId>postgres-socket-factory</artifactId>
    <version>1.2.1</version>
</dependency>
```
L'url per la connessione diventa il seguente:

```
jdbc:postgresql:///<IL_TUO_DATABASE>?cloudSqlInstance=<NOME_DELLA_CONNESSIONE>&socketFactory=com.google.cloud.sql.postgres.SocketFactory
```
Il nome della connessione si trova nella pagina della tua istanza cloud sql nel campo "connection name".
Nota che per utilizzare il Cloud SQL Proxy, ho utilizzato un IP pubblico.
Per far cio' sono andato nella pagina dell'istanza Cloud SQL, poi su Connessioni, ed infine ho settato Public IP nella sezione Networking.

Dopo aver fatto cio', quando ho provato ad eseguire la migrazione con il seguente comando
```bash
mvn flyway:migrate
```
Ho ricevuto il seguente errore:
```
The Application Default Credentials are not available. They are available if running in Google Compute Engine. Otherwise, the environment variable GOOGLE_APPLICATION_CREDENTIALS must be defined pointing to a file defining the credentials. See https://developers.google.com/accounts/docs/application-def
ault-credentials for more information.
```
Per risolverlo, ho dovuto creare un nuovo service account nella pagina Service Accounts. Fatto cio', l'ho scaricato e ho settato la variabile d'ambiente GOOGLE_APPLICATION_CREDENTIALS con il percorso del file.
Dopo aver riprovato ad eseguire la migrazione, ho ricevuto il seguente errore:
```
Cloud SQL Admin API has not been used in project <my project id> before or it is disabled. Enable it by visiting ...
```
Per risolverlo, sono andato nella pagina Cloud SQL Admin API, e ho abilato la funzione.

Ora dovresti essere in grado di connetterti con successo a Cloud SQL (assumendo che tu stia utilizzando il nome del database, utente e password corretti).
