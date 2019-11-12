# Using Ory Hydra to Generate SciTokens

[Ory Hydra](https://www.ory.sh/docs/hydra/) is an [open-source](https://github.com/ory/hydra) OAuth 2.0 and OpenID Connect Provider. It provides the OAuth2 server, a login and consent application, and an OAuth2 client tool as Docker containers. Here, we demonstrate how Hydra can be configured to generate [SciTokens.](https://scitokens.org)

This documents assumes that the reader is familar with OAuth2 and [Hydra](https://www.ory.sh/docs/hydra/). The document [Run your own OAuth2 Server](https://www.ory.sh/run-oauth2-server-open-source-api-security/) provides a good introduction to these topics. More details on SciTokens can be found on the [SciTokens](https://scitokens.org) web pages.

## Setting up the Server Containers

Note that this tutorial is meant to illustrate the principles of setup, it is not suitable for production as it uses http endpoints on an internal network that are not protected by encryption.

We first create a Docker network that the containers will use to communicate with each other:
```sh
docker network create hydraguide
```

### Database Container

Start the container that Hydra used for its OAuth2 database with:

```sh
docker run --network hydraguide \
   --name ory-hydra-example--postgres \
   -e POSTGRES_USER=hydra \
   -e POSTGRES_PASSWORD=secret \
   -e POSTGRES_DB=hydra \
   -d postgres:9.6
```

Create a system secret that Hydra uses to secure itself with the command:
```sh
export SECRETS_SYSTEM=$(export LC_CTYPE=C; cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
```
and set an environment variable that tells the other Docker containers how to communicate with the database:
```sh
export DSN=postgres://hydra:secret@ory-hydra-example--postgres:5432/hydra?sslmode=disable
```
Communication is performed over the `hydraguide` Docker network.

Next run the Hydra container with the `migrate sql` command to create the database:
```sh
docker run -it --rm \
  --network hydraguide \
  oryd/hydra:v1.0.8 \
  migrate sql --yes $DSN
```

### Hydra OAuth Server

Once the database container is running and configured, we need to start the contaier that will run the OAuth2 server itself.

By default Hydra creates opaque OAuth2 tokens, so we add the environment variable `OAUTH2_ACCESS_TOKEN_STRATEGY=jwt` so that it returns [JSON Web Tokens](https://jwt.io/), which is the format used by [SciTokens.](https://scitokens.org) For discussion of Hydra's implementation of JWTs see the discussion in [ory/hydra/issues/248](https://github.com/ory/hydra/issues/248) and [ory/hydra/pull/947](https://github.com/ory/hydra/pull/947).

To demonstrate a login and consent workflow, we will configure the server to use an instance of the [Hydra Login and Consent Node](https://github.com/ory/hydra-login-consent-node) running on port 9020 by setting the environment variables `URLS_CONSENT=http://127.0.0.1:9020/consent` and `URLS_LOGIN=http://127.0.0.1:9020/login`.

```sh
docker run -d \
  --name ory-hydra-example--hydra \
  --network hydraguide \
  -p 9000:4444 \
  -p 9001:4445 \
  -e SECRETS_SYSTEM=$SECRETS_SYSTEM \
  -e DSN=$DSN \
  -e URLS_SELF_ISSUER=http://127.0.0.1:9000/ \
  -e URLS_CONSENT=http://127.0.0.1:9020/consent \
  -e URLS_LOGIN=http://127.0.0.1:9020/login \
  -e OAUTH2_ACCESS_TOKEN_STRATEGY=jwt\
  oryd/hydra:v1.0.8 serve all --dangerous-force-http
```
Note that this server is not suitable for production use, as it runs on the internal network and allows regular http connections with the `--dangerous-force-http` argument.

You can check that the server is running with the command:
```sh
docker logs ory-hydra-example--hydra
```
If the container is started correctly, this should show:
```
time="2019-11-12T16:22:40Z" level=info msg="Setting up http server on :4445"
time="2019-11-12T16:22:40Z" level=info msg="Setting up http server on :4444"
time="2019-11-12T16:22:40Z" level=warning msg="HTTPS disabled. Never do this in production."
time="2019-11-12T16:22:40Z" level=warning msg="HTTPS disabled. Never do this in production."
```

Once this message is displayed, you can go to server's [.well-known/openid-configuration](http://127.0.0.1:9000/.well-known/openid-configuration) to see the `authorization_endpoint` and `token_endpoint`. 

## Create an OAuth2 Client

The next step is to create an OAuth2 client in the server's database. In order for the client to request [SciTokens](https://scitokens.org), we need to configure the `--audience` and `--scope` for the tokens that will be issues to the client. 

docker run --rm -it \
   --network hydraguide \
   oryd/hydra:v1.0.8 \
   clients create \
     --endpoint http://ory-hydra-example--hydra:4445 \
     --id scitokens \
     --secret badgers \
     -g authorization_code,refresh_token \
     -r token,code,id_token \
     --scope offline,openid,read:/public,write:/home/dbrown \
     --audience sugwg-scitokens.phy.syr.edu \
     --callbacks http://127.0.0.1:9010/callback
     
docker run -d \
  --name ory-hydra-example--consent \
  -p 9020:3000 \
  --network hydraguide \
  -e HYDRA_ADMIN_URL=http://ory-hydra-example--hydra:4445 \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  oryd/hydra-login-consent-node:v1.0.8
  
docker run --rm -it \
  --network hydraguide \
  -p 9010:9010 \
  oryd/hydra:v1.0.8 \
  token user \
    --port 9010 \
    --auth-url http://127.0.0.1:9000/oauth2/auth \
    --token-url http://ory-hydra-example--hydra:4444/oauth2/token \
    --client-id scitokens \
    --client-secret badgers \
    --scope openid,offline \
    --redirect http://127.0.0.1:9010/callback
 ```
 



