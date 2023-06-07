---
pageTitle: Integrate Okta with Arcion Replicant
title: Okta
description: "Authenticate your Replicant users by using Okta integration."
weight: 17
url: docs/references/okta-setup
---

# Set up Okta as an authentication provider

## Requirements
- A valid Okta account
- A Replicant self-hosted docker container
- A database for Replicant container metadata

## Create an Okta Application

1. Log into your Okta account
2. In the Admin Console, go to **Applications > Applications** .
3. Click Create App Integration.
4. Select **OIDC - OpenID Connect** as the **Sign-in method**.
5. Select **Single-Page Application** as the **Application type**.
6. The **New Single-Page App Integration** window appears.
  i. Select both **Authorization Code** and **Refresh Token** as the **Grant type**s. The login process requires **Authorization Code** and the re-authentication mechanism requires **Refresh Token**. 
  ii. Enter `http://localhost:8080/auth-callback/` in the **Sign-in redirect URIs** box. We run the container on our local machine without any additional setup. Since the port maps to 8080, the base URL becomes `http://localhost:8080` and the complete **Sign-in redirect URI** becomes `http://localhost:8080/auth-callback/`. 
  iii. In the **Sign-out redirect URIs** box, enter the base URL. In this case, it's `http://localhost:8080/`. 
  iv. For **Assignments**, select **Allow everyone in your organization to access** . Keep in mind that you can set up user assignments in any way that allows the configured application to authenticate users.

After creating the app integration connection, a window appears with all the details of your application. The **Client Credentials** section contains your application's **Client ID**. This works as your app's public identifier and required by all OAuth flows. Therefore, make sure to save the client ID for later.


## Configure self-hosted container to authenticate against Okta

A Docker-compose file which spins up a PostgreSQL container and a Replicant-on-premises container can be found below. Please note that the
configuration below is invalid and specific variables for your setup should be assigned to the docker compose file. A 
list of all environment variables relevant for Okta integration is available below the Docker-compose file contents.

```YAML
version: '3.8'
services:
  postgresql-12.4-meta:
    image: postgres:12.4
    hostname: postgres
    ports:
      - "5432:5432"
    networks:
      Replicant:
        aliases:
          - postgres
    environment:
      - 'POSTGRES_PASSWORD=password'
      - 'POSTGRES_USER=admin'
      - 'POSTGRES_DB=Replicant'
    container_name: replicate-postgresql-12.4-meta
  Replicant-on-premises:
    ports:
      - '8080:8080'
    environment:
      - 'DB=POSTGRESQL'
      - 'DB_HOST=postgres'
      - 'DB_PORT=5432'
      - 'DB_DATABASE=Replicant'
      - 'DB_USERNAME=admin'
      - 'DB_PASSWORD=password'
      - 'AUTHENTICATION_TYPE=OAUTH2'
      - 'ISSUER_URI=https://trial-12345678.okta.com'
      - 'USER_INFO_URI=https://trial-12345678.okta.com/oauth2/v1/userinfo'
      - 'CLIENT_ID=w7n8s0j144nk9laf1qwm'
      - 'AUTHORIZATION_URI=https://trial-12345678.okta.com/oauth2/v1/authorize'
      - 'ARCION_LICENSE=Base64 encoded license'
    image: arcionlabs/Replicant-on-premises:latest
    depends_on:
      - postgresql-12.4-meta
    networks:
      Replicant:
        aliases:
          - Replicant
    extra_hosts:
      - 'host.docker.internal:host-gateway'
networks:
  Replicant:
```
## Environment variables

- `ARCION_LICENSE` specifies the base64 encoded Replicant license.
- `AUTHENTICATION_TYPE` specifies the authentication protocol. Since we're using OIDC/OAuth2 as our authentication protocol, set this parameter to `OAUTH2`.
- `CLIENT_ID` specifies the client id of the corresponding Okta application you create in the [Create an Okta Application](#create-an-okta-application) section.

You can find Okta URIs in the OIDC Discovery configuration file. Default location for this file takes the form `https://YOUR_Okta_DOMAIN/.well-known/openid-configuration`,
where `YOUR_Okta_DOMAIN` represents [your Okta domain](https://developer.okta.com/docs/guides/find-your-domain/main/#find-your-okta-domain). 

For more information on OIDC Discovery, see [OpenID Connect & OAuth 2.0 API](https://developer.okta.com/docs/reference/api/oidc/#well-known-openid-configuration).
`ISSUER_URI` specifies URI of the authentication issuer (https://YOUR_Okta_DOMAIN)</br>
`USER_INFO_URI` specifies URI used to check token validity (https://YOUR_Okta_DOMAIN/oauth2/v1/userinfo)</br>
`AUTHORIZATION_URI` specifies URI used to exchange authorization codes (https://YOUR_Okta_DOMAIN/oauth2/v1/authorize)</br>


### OAuth2 scopes

Client registration requires the following scopes:
- `offline_access`
- `openid`
- `profile`
- `email`
