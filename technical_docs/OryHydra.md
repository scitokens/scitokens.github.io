https://www.ory.sh/run-oauth2-server-open-source-api-security/

```sh
docker network create hydraguide

docker run --network hydraguide \
   --name ory-hydra-example--postgres \
   -e POSTGRES_USER=hydra \
   -e POSTGRES_PASSWORD=secret \
   -e POSTGRES_DB=hydra \
   -d postgres:9.6
   
export SECRETS_SYSTEM=$(export LC_CTYPE=C; cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)

export DSN=postgres://hydra:secret@ory-hydra-example--postgres:5432/hydra?sslmode=disable

docker run -it --rm \
  --network hydraguide \
  oryd/hydra:v1.0.8 \
  migrate sql --yes $DSN
  
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
  oryd/hydra:v1.0.8 serve all --dangerous-force-http

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
 
 http://127.0.0.1:9000/.well-known/openid-configuration
