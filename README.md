# Envoy JWT Claim Authorization

A Envoy reverse proxy setup to authenticate and authorise requests based on OIDC JWT claims.

Envoy Modules:  
- envoy.filters.http.jwt_authn for OIDC Authentication
- envoy.filters.http.rbac for access based on HTTP Path, Header, Method and JWT claims

Relevant Documentation
- [config.rbac.v3.RBAC proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-msg-config-rbac-v3-rbac)
- [extensions.filters.http.jwt_authn.v3.JwtAuthentication proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto)


## Start Dev Cluster
1. Install [Task](https://taskfile.dev/docs/installation)
2. Run `task setup`
3. (Optionaly) Reload envoy config with `task apply-envoy`


## Get ID Tokens
The `jwt_authn` filter module has 2 test issuer/provider configured in [envoy.yaml](k8s/envoy-app/envoy.yaml).  

1. A local keycloak realm ([realm-config](k8s/keycloak/config.yaml)) with static user+password and group assigments which we map into the ID-Token via `oidc-group-membership-mapper`.
2. The OIDC Endpont of our local kind cluster. The JWKS endpoints got allowed by a [ClusterRole+Binding for unauthenicated users](k8s/kind-rbac/rbac.yaml).

Since we define port forwarding ([kind config](k8s/config.yaml)) for our node-port services keycloak (:30000) and envoy (:30001), we can reach them from localhost.

As envoy upstream we have set a [http-echo](https://github.com/mendhak/docker-http-https-echo) web server which returns the http request it got from envoy. This helps us to check if envoy handled e.g. header mutation correct on an allowed request.

### Local Keycloak
```sh
KEYCLOAK_USER=test
KEYCLOAK_PASSWORD=test
response=$(curl -s -X POST \
    http://localhost:30000/realms/demo/protocol/openid-connect/token \
    -H "Content-Type: application/x-www-form-urlencoded" \
    -d "grant_type=password" \
    -d "client_id=my-app" \
    -d "username=$KEYCLOAK_USER" \
    -d "password=$KEYCLOAK_PASSWORD" \
    -d "scope=openid")
TOKEN=$(echo "$response" | jq -r '.id_token')
echo $TOKEN
```

### Local Kubernetes ServiceAccount Token
```sh
TOKEN=$(kubectl create token test-sa -n default)
echo $TOKEN
```

### Application Request
```sh
# Only allowed for Keycloak User
curl -v http://localhost:30001/read \
    -X GET \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-Scope-OrgID: pvt.local-development.test"

# Only allowed for Kubernetes ServiceAccount
curl -v http://localhost:30001/write \
    -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "X-Scope-OrgID: pvt.local-development.test"
```


### Create Debug Pod
```sh
kubectl delete pod tmp-shell --ignore-not-found && \
kubectl run tmp-shell --restart=Never --rm -i --tty --image=nicolaka/netshoot
```