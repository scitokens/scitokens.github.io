# Using Ory Hydra to Generate SciTokens

[Ory Hydra](https://www.ory.sh/docs/hydra/) is an [open-source](https://github.com/ory/hydra) OAuth 2.0 and OpenID Connect Provider. It provides the OAuth2 server, a login and consent application, and an OAuth2 client tool as Docker containers. Here, we demonstrate how Hydra can be configured to generate [SciTokens.](https://scitokens.org)

This documents assumes that the reader is familar with OAuth2 and [Hydra](https://www.ory.sh/docs/hydra/). The document [Run your own OAuth2 Server](https://www.ory.sh/run-oauth2-server-open-source-api-security/) provides a good introduction to these topics. More details on SciTokens can be found on the [SciTokens](https://scitokens.org) web pages.

This tutorial shows how to set up the server and create an OAuth2 client in the server's database. In order for clients to be able to use [SciTokens](https://scitokens.org), we need to configure the `--audience` and `--scope` for the tokens that will be issues to the client. This can be done in two ways: 

 * Using the [command line tool used in the tutorial](https://www.ory.sh/docs/hydra/5min-tutorial)
 * Directly using the [REST API.](https://www.ory.sh/docs/hydra/sdk/api#create-an-oauth-20-client)

We give examples of both methods below.

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

Once this message is displayed, you can go to server's [.well-known/openid-configuration](http://127.0.0.1:9000/.well-known/openid-configuration) to see the `authorization_endpoint` and `token_endpoint` and [.well-known/jwks.json](http://127.0.0.1:9000/.well-known/jwks.json) to see the keys for the tokens.

For more information on interacting with the Hydra server, see the [documentation for the REST API.](https://www.ory.sh/docs/hydra/sdk/api)

### Login and Consent Container

This tutorial uses an instance of the [Hydra Login and Consent Node](https://github.com/ory/hydra-login-consent-node) running on port 9020 to provide the OIDC and consent parts of the OAuth2 flow. The server as one user with the username `foo@bar.com` and password `foobar`.

Start the server with the command:
```sh
docker run -d \
  --name ory-hydra-example--consent \
  -p 9020:3000 \
  --network hydraguide \
  -e HYDRA_ADMIN_URL=http://ory-hydra-example--hydra:4445 \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  oryd/hydra-login-consent-node:v1.0.8
```

## Create and Test OAuth2 Clients

We first create a simple client that can obtain an authorization token using the command line tool, and then a client that can obtain a refresh token using [Insomnia](https://insomnia.rest/) to talk to the [Hydra Admistrative REST API.](https://www.ory.sh/docs/hydra/sdk/api#administrative-endpoints)

### Command Line tool

To create a client using the command line tool, run the command below to talk to the server's administrative endpoint on the Docker internal network (`http://ory-hydra-example--hydra:4445`):
```sh
docker run --rm -it \
   --network hydraguide \
   oryd/hydra:v1.0.8 \
   clients create \
     --endpoint http://ory-hydra-example--hydra:4445 \
     --id scitokens_cmd \
     --secret badgers \
     -g authorization_code \
     -r code \
     --scope read:/public,write:/home/dbrown \
     --audience sugwg-scitokens.phy.syr.edu \
     --callbacks http://127.0.0.1:9010/callback
```

This creates an OAuth2 client with the Client ID `scitokens_cmd` and the Client Secret `badgers`. The grant types are set with the `-g` option to be `authorization_code`, as this is the primary grant used to get a [SciTokens.](https://scitokens.org). We set the response types with `-r` to be `code` since we want to perform an [Authorization Code grant.](https://auth0.com/docs/protocols/oauth2#how-response-type-works) In the above examples `--audience` is set to the name of the server where we wish to use the token, and the `--scope` is set to the allowed tokens scopes for the audience.

To obtain a token, run the Hydra token client with the following command:
```sh
docker run --rm -it \
  --network hydraguide \
  -p 9010:9010 \
  oryd/hydra:v1.0.8 \
  token user \
    --port 9010 \
    --auth-url http://127.0.0.1:9000/oauth2/auth \
    --token-url http://ory-hydra-example--hydra:4444/oauth2/token \
    --client-id scitokens_cmd \
    --client-secret badgers \
    --audience sugwg-scitokens.phy.syr.edu \
    --scope write:/home/dbrown \
    --redirect http://127.0.0.1:9010/callback
 ```

This will respond with the message below:
```
Setting up home route on http://127.0.0.1:9010/
Setting up callback listener on http://127.0.0.1:9010/callback
Press ctrl + c on Linux / Windows or cmd + c on OSX to end the process.
If your browser does not open automatically, navigate to:

	http://127.0.0.1:9010/
```
Using a browser, visit the link provided and follow the login and consent flow to login as `foo@bar.com` and authorize the grant. The client will return a token that can be used for authorization, or decrypted by pasting it into the encoded box at [https://demo.scitokens.org/](https://demo.scitokens.org/)

The headers and payload of a token created following the above example will look like this:
```json
{
  "alg": "RS256",
  "kid": "public:040819c0-9112-4851-aeba-62d868776dbf",
  "typ": "JWT"
}
{
  "aud": [
    "sugwg-scitokens.phy.syr.edu"
  ],
  "client_id": "scitokens_cmd",
  "exp": 1573584333,
  "ext": {},
  "iat": 1573580733,
  "iss": "http://127.0.0.1:9000/",
  "jti": "c74578c9-c213-42ac-80d5-e4ca238490fd",
  "nbf": 1573580733,
  "scp": [
    "write:/home/dbrown"
  ],
  "sub": "foo@bar.com"
}
```

### REST API

Now we will use the [Insomnia REST Client](https://insomnia.rest) to create an OAuth2 client and issue a token. [Create a new request](https://support.insomnia.rest/article/11-getting-started) in Insomnia that will send a `POST` to the server at `http://127.0.0.1:9001/clients` to create a new client. The body of the request should be set to JSON and should contain:
```json
{
  "audience": ["sugwg-scitokens.phy.syr.edu"],
  "client_id": "scitokens_rest",
  "client_name": "scitokens_rest",
  "client_secret": "herecomethebadgers",
  "client_uri": "https://scitokens.org/",
  "contacts": ["dabrown@syr.edu"],
  "grant_types": ["authorization_code", "refresh_token"],
  "owner": "dabrown@syr.edu",
  "post_logout_redirect_uris": ["https://scitokens.org/"],
  "policy_uri": "https://scitokens.org/",
  "redirect_uris": ["https://scitokens.org", "http://127.0.0.1:9010/callback"],
  "response_types": ["code"],
  "scope": "offline read:/public write:/home/dbrown",
	"token_endpoint_auth_method": "client_secret_basic"
}
```

The client created will be similar to the previous example, but we also add the grant type `refresh_token` so that a refresh token can be generated. If the post is successful the server will return `201 Created` and the response will contain the full configuratuon of the client:
```json
{
  "client_id": "scitokens_rest",
  "client_name": "scitokens_rest",
  "client_secret": "herecomethebadgers",
  "redirect_uris": [
    "https://scitokens.org",
    "http://127.0.0.1:9010/callback"
  ],
  "grant_types": [
    "authorization_code",
    "refresh_token"
  ],
  "response_types": [
    "code"
  ],
  "scope": "offline read:/public write:/home/dbrown",
  "audience": [
    "sugwg-scitokens.phy.syr.edu"
  ],
  "owner": "dabrown@syr.edu",
  "policy_uri": "https://scitokens.org/",
  "allowed_cors_origins": null,
  "tos_uri": "",
  "client_uri": "https://scitokens.org/",
  "logo_uri": "",
  "contacts": [
    "dabrown@syr.edu"
  ],
  "client_secret_expires_at": 0,
  "subject_type": "public",
  "token_endpoint_auth_method": "client_secret_basic",
  "userinfo_signed_response_alg": "none",
  "created_at": "2019-11-12T17:56:50Z",
  "updated_at": "2019-11-12T17:56:50Z",
  "post_logout_redirect_uris": [
    "https://scitokens.org/"
  ]
}
```

To use this client, create a new request to send a `POST` to `http://127.0.0.1:9001/oauth2/introspect` so that we can obtain and introspect a token. Set the request's Auth type to be `OAuth2` and configure the query as shown below. 

<img src="https://github.com/duncan-brown/scitokens.github.io/raw/master/technical_docs/insomnia_ory_example.png" alt="Insomnia OAuth2 Configuration" data-canonical-src="https://github.com/duncan-brown/scitokens.github.io/raw/master/technical_docs/insomnia_ory_example.png" width="400" />

The click `Refresh Token` to obtain refresh and access tokens.

To check the contents of the access token, you can paste it into [https://demo.scitokens.org](https://demo.scitokens.org), which will display a payload similar to the following:
```json
{
  "aud": [
    "sugwg-scitokens.phy.syr.edu"
  ],
  "client_id": "scitokens_rest",
  "exp": 1573585481,
  "ext": {},
  "iat": 1573581880,
  "iss": "http://127.0.0.1:9000/",
  "jti": "2dfe35a3-261d-4d31-b111-1a40f623d718",
  "nbf": 1573581880,
  "scp": [
    "read:/public",
    "offline"
  ],
  "sub": "foo@bar.com"
}
```

Alternatively, you can set the Insomnia request body to `Form URL Encoded` and create a field with the name `token` and paste in the access token as the value. Posting this to the introspection endpoint will show
```json
{
  "active": true,
  "scope": "offline read:/public",
  "client_id": "scitokens_rest",
  "sub": "foo@bar.com",
  "exp": 1573585048,
  "iat": 1573581447,
  "aud": [
    "sugwg-scitokens.phy.syr.edu"
  ],
  "iss": "http://127.0.0.1:9000/",
  "token_type": "access_token"
}
```
