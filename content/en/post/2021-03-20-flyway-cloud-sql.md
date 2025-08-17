---
title: "Setup Flyway with Google Cloud SQL"
date: 2021-03-20T10:42:16+01:00
draft: false
tags: ["flyway", "gcp", "cloudsql"]
---

Flyway is a tool that allow robust database schema evolution. In this post we will explore how it can be connected with Google Cloud SQL.
<!--more-->
In my specific case, I am using Maven with the flyway-maven-plugin. I'm running Postgres with on Cloud SQL. The way I decided to connect to the
database is by using the [Cloud SQL Proxy](https://cloud.google.com/sql/docs/postgres/connect-admin-proxy). The connection will be managed by
the [cloud sql jdbc socket factory](https://github.com/GoogleCloudPlatform/cloud-sql-jdbc-socket-factory). In my case, since I'm using Postgres,
I had to add the following dependency to my pom.xml file.
```xml
<dependency>
    <groupId>com.google.cloud.sql</groupId>
    <artifactId>postgres-socket-factory</artifactId>
    <version>1.2.1</version>
</dependency>
```
Next, the connection url will be something like the following:

```
jdbc:postgresql:///<YOUR_DATABASE>?cloudSqlInstance=<YOUR-INSTANCE_CONNECTION_NAME>&socketFactory=com.google.cloud.sql.postgres.SocketFactory
```
The instance connection name can be found in the cloud sql instance in the connection name field. 
Note that I set networking to Public IP for my instance.
To do so, go to your Cloud SQL instance page,
click on Connections, and verify that Public IP is selected in the Networking section.

After doing this, when running
```bash
mvn flyway:migrate
```
I got the following error:
```
The Application Default Credentials are not available. They are available if running in Google Compute Engine. Otherwise, the environment variable GOOGLE_APPLICATION_CREDENTIALS must be defined pointing to a file defining the credentials. See https://developers.google.com/accounts/docs/application-def
ault-credentials for more information.
```
To fix this, all I had to do was to generate a new service account in the Service Accounts page, download it and create the GOOGLE_APPLICATION_CREDENTIALS environment variable with the path to that file as a value.
Next, when I tried to run the migration again, I got the following error:
```
Cloud SQL Admin API has not been used in project <my project id> before or it is disabled. Enable it by visiting ...
```
To fix this, simply go to the Cloud SQL Admin API page, and enable it for your project.

After that, assuming that you are using the correct database, user, and password, you should be able to connect successfully to Cloud SQL and run your migrations.
