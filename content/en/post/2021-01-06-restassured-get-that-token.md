---
title: "REST Assured, get that damn access token!"
date: 2021-01-06T09:42:16+01:00
draft: false
tags: ["rest-assured", "quarkus", "keycloak"]
---

I was recently playing around with quarkus and keycloak (using the openid-connect protocol) and I wanted to create an automated test for a protected resource
<!--more-->
With curl you can get the access token like this
```bash
export access_token=$(
    curl --insecure -X POST yourKeycloakServerUrl/protocol/openid-connect/token 
    --user clientId:secret 
    -H 'content-type: application/x-www-form-urlencoded' 
    -d 'username=umberto&password=password&grant_type=password' | jq --raw-output '.access_token' 
)
```
And then use it for a protected resource as following
```bash
curl -v -X GET   http://localhost:8080/api/v1/users/umberto/articles   -H "Authorization: Bearer "$access_token
```

With REST Assured you can get it in 12 easy steps :)
```java
String accessToken = given()
                .auth()
                .preemptive()
                .basic(clientId, secret)
                .header("Content-Type", "application/x-www-form-urlencoded")
                .baseUri(serverUrl)
                .body("username=umberto&password=password&grant_type=password")
                .post("/protocol/openid-connect/token")
                .then().extract().response().jsonPath().getString("access_token");
```
and if you are using quarkus with quarkus-oidc you can retrieve the config parameters as following

```java
@ConfigProperty(name="quarkus.oidc.auth-server-url")
String serverUrl;

@ConfigProperty(name="quarkus.oidc.client-id")
String clientId;

@ConfigProperty(name="quarkus.oidc.credentials.secret")
String secret;
```