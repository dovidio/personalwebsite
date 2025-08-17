---
title: "Setup a Github Worflow for migrating your Cloud SQL instance with Flyway"
date: 2021-03-21T10:42:16+01:00
draft: false
tags: ["flyway", "gcp", "cloudsql", "githubactions"]
---

I've recently wrote a blog post on how to setup the connection between Flyway and Google Cloud SQL. In this post I will 
explain how to create a Github workflow that will run the flyway migration once triggered.
<!--more-->
You will need of course a yaml file in your github repo under the directory `.github/workflows`.
Here's the code
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

I've used `on: [workflow_dispatch]` so that this action can be triggered only manually.
Then I've set some environment variables that specify the credentials and the url used for the jdbc connection.
Then I'm checking out the project, and exporting the default credentials. The `GCP_PROJECT_ID` should contain the id of your project,
while the `GCP_SA_KEY` will contain your service account in json format. The export default credentials will then populate the
`GOOGLE_APPLICATION_CREDENTIALS` which is needed for the Cloud SQL connection. I'm then setting up java with the `setup-java` action (which includes setting up maven),
and finally I'm printing some info from flyway before executing the migration.
That's it, you can now run your migrations with a click from github :)

![Github Action](/images/github-action.png)