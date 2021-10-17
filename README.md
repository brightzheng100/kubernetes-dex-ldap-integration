# Kubernetes + Dex + LDAP Integration

A simple walk-through guide for how to integrate `Kubernetes` with `Dex` + `LDAP`.

In this experiment, we're going to use these major components:

- Kubernetes v1.21.x, powered by [`kind` v0.11.1](https://kind.sigs.k8s.io/);
- [Dex](https://github.com/dexidp/dex) v2.30.x;
- [OpenLDAP](https://www.openldap.org/) with [osixia/openldap:1.5.x](https://github.com/osixia/docker-openldap)

A Medium article was posted too, here: https://brightzheng100.medium.com/kubernetes-dex-ldap-integration-f305292a16b9

The overall idea can be illustrated as below:

![architecture](misc/architecture.png)

## Get Started

```sh
git clone https://github.com/brightzheng100/kubernetes-dex-ldap-integration.git
cd kubernetes-dex-ldap-integration
```

## The TL;DR Guide

### Setup

The TD;DR guide uses the script here: [setup.sh](setup.sh), which will:
1. check the required tools -- there are some of them: `docker`, `git`, `cfssl`, `cfssljson`, `kind`, `kubectl`;
2. generate the necessary TLS certs/keys for both Kubernetes and Dex;
3. create `kind`-powered Kubernetes with OIDC configured with Dex;
4. deploy OpenLDAP in namespace `ldap` as the LDAP Server with some dummy entities;
5. deploy Dex in namespace `dex`;
6. create a proxy so that we can access Dex from our laptop (e.g. my MBP)

```sh
./setup.sh
```

> Note: the populated dummy LDAP entities, all with password `secret`, include:
> - `admin1@example.org`
> - `admin2@example.org`
> - `developer1@example.org`
> - `developer2@example.org`

### Use

It's common to set up the kube config, e.g. `~/.kube/config`, for daily use.

For that, we may simply follow these steps:

1. Bind some users, say **"admin1@example.org"**, as the **"cluster-admin"**

    ```sh
    $ kubectl create clusterrolebinding oidc-cluster-admin \
      --clusterrole=cluster-admin \
      --user="admin1@example.org"
    ```

2. Use [`kubelogin`](https://github.com/int128/kubelogin) plugin to simplify the integration

    ```sh
    $ echo "127.0.0.1 dex.dex.svc" | sudo tee -a /etc/hosts

    $ SVC_PORT="$(kubectl get -n dex svc/dex -o json | jq '.spec.ports[0].nodePort')"
    $ kubectl config set-credentials oidc \
        --exec-api-version=client.authentication.k8s.io/v1beta1 \
        --exec-command=kubectl \
        --exec-arg=oidc-login \
        --exec-arg=get-token \
        --exec-arg=--oidc-issuer-url=https://dex.dex.svc:$SVC_PORT \
        --exec-arg=--oidc-redirect-url-hostname=dex.dex.svc \
        --exec-arg=--oidc-client-id=example-app \
        --exec-arg=--oidc-client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0 \
        --exec-arg=--oidc-extra-scope=email \
        --exec-arg=--certificate-authority=`pwd`/tls-setup/_certs/ca.pem
    ```

3. Use the user to access Kubernetes

    ```sh
    $ kubectl --user=oidc get nodes
    ```

    This will prompt us a authentication UI in our default browser, key in the credential of abovementioned LDAP user:
    - Email Address: `admin1@example.org`
    - Password: `secret`

    ![screenshot-with-kubelogin](misc/screenshots-with-kubelogin.png)

    It will be authenticated by Dax + LDAP, and once the authentication is done we can see the output like:

    ```
    $ kubectl --user=oidc get nodes
    NAME                             STATUS   ROLES                  AGE     VERSION
    dex-ldap-cluster-control-plane   Ready    control-plane,master   8m55s   v1.21.1
    dex-ldap-cluster-worker          Ready    <none>                 8m30s   v1.21.1
    ```

    > Notes: 
    > - as the login will be cached so the subsequent access will be transparent;
    > - if you want to change the user, you may remove the cache (`rm -rf ~/.kube/cache/oidc-login/*`) first and you would be prompted again for what user you want to use;
    
4. (optional) Keep multiple users logged in?
   
    So far I haven't found a good way to have multiple users coexist and maintain logged in with `kubelogin`. But we can try generate tokens for different users and set the credentials accordingly:

    ```sh
    cd example-app
    go run . --issuer https://dex.dex.svc:32000 --issuer-root-ca `pwd`/../tls-setup/_certs/ca.pem
    ```

    Then open your browser and navigate to http://127.0.0.1:5555, log into it with the users you want, one at a time to generate different tokens, then add different users with the `ID Token` and `Refresh Token` into your kubeconfig file, e.g `~/.kube/config`, like this:

    ```sh
    # Add user admin1@example.org as oidc-admin1
    kubectl config set-credentials oidc-admin1 \
      --auth-provider=oidc \
      --auth-provider-arg=idp-issuer-url=https://dex.dex.svc:32000 \
      --auth-provider-arg=client-id=example-app \
      --auth-provider-arg=client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0 \
      --auth-provider-arg=refresh-token=<THE REFRESH TOKEN FOR admin1@example.org> \
      --auth-provider-arg=idp-certificate-authority=`pwd`/tls-setup/_certs/ca.pem \
      --auth-provider-arg=id-token=<THE ID TOKEN FOR admin1@example.org>

    # Add user developer1@example.org as oidc-developer1
    kubectl config set-credentials oidc-developer1 \
      --auth-provider=oidc \
      --auth-provider-arg=idp-issuer-url=https://dex.dex.svc:32000 \
      --auth-provider-arg=client-id=example-app \
      --auth-provider-arg=client-secret=ZXhhbXBsZS1hcHAtc2VjcmV0 \
      --auth-provider-arg=refresh-token=<THE REFRESH TOKEN FOR developer1@example.org> \
      --auth-provider-arg=idp-certificate-authority=`pwd`/tls-setup/_certs/ca.pem \
      --auth-provider-arg=id-token=<THE ID TOKEN FOR developer1@example.org>
    ```


## The Step-by-step Guide

### Generating TLS PKI files for both Dex and K8s

> Note: 
> 1. [`cfssl` and `cfssljson`](https://github.com/cloudflare/cfssl/releases) are required to generate certs/keys
> 2. You may try using [cert-manager](https://github.com/jetstack/cert-manager), if you want

```sh
cd tls-setup

make ca req-dex req-k8s
```

> OUTPUT: a folder `_certs` will be created and a couple of pairs of certs/keys will be generated.

```sh
$ tree _certs
_certs
├── ca-key.pem
├── ca.csr
├── ca.pem
├── dex-key.pem
├── dex.csr
├── dex.pem
├── k8s-key.pem
├── k8s.csr
└── k8s.pem

0 directories, 9 files
```

### Creating Kubernetes cluster with API Server configured

> Note: Here I'm going to use `kind`, you may try any other ways too, like `minikube`, `k3s/k3d`, but the process might have to tune a little bit.

```sh
# Make sure we're working from the Git repo's root folder
cd "$( git rev-parse --show-toplevel )"

PROJECT_ROOT="$(pwd)" envsubst < kind/kind.yaml | kind create cluster --name dex-ldap-cluster --config -
```

### Deploying OpenLDAP as the LDAP Server

```sh
# Make sure we're working from the Git repo's root folder
cd "$( git rev-parse --show-toplevel )"

kubectl create ns ldap

kubectl create secret generic openldap \
    --namespace ldap \
    --from-literal=adminpassword=adminpassword

kubectl create configmap ldap \
    --namespace ldap \
    --from-file=ldap/ldif

kubectl apply --namespace ldap -f ldap/ldap.yaml

# Load ldif data after the OpenLDAP is ready
# Note: by right, they should be loaded automatically but it doesn't work so we load them manually
LDAP_POD=$(kubectl -n ldap get pod -l "app.kubernetes.io/name=openldap" -o jsonpath="{.items[0].metadata.name}")
kubectl -n ldap exec $LDAP_POD -- ldapadd -x -D "cn=admin,dc=example,dc=org" -w adminpassword -H ldap://localhost:389 -f /ldifs/0-ous.ldif
kubectl -n ldap exec $LDAP_POD -- ldapadd -x -D "cn=admin,dc=example,dc=org" -w adminpassword -H ldap://localhost:389 -f /ldifs/1-users.ldif
kubectl -n ldap exec $LDAP_POD -- ldapadd -x -D "cn=admin,dc=example,dc=org" -w adminpassword -H ldap://localhost:389 -f /ldifs/2-groups.ldif

# Check the users loaded
kubectl -n ldap exec $LDAP_POD -- \
    ldapsearch -LLL -x -H ldap://localhost:389 -D "cn=admin,dc=example,dc=org" -w adminpassword -b "ou=people,dc=example,dc=org" dn
```

You should see some users that have been created:

```
dn: ou=people,dc=example,dc=org
dn: cn=admin1,ou=people,dc=example,dc=org
dn: cn=admin2,ou=people,dc=example,dc=org
dn: cn=developer1,ou=people,dc=example,dc=org
dn: cn=developer2,ou=people,dc=example,dc=org
```

### Deploying Dex on Kubernetes with LDAP integrated

```sh
# Make sure we're working from the Git repo's root folder
cd "$( git rev-parse --show-toplevel )"

kubectl create ns dex

kubectl create secret tls dex-tls \
    --namespace dex \
    --cert=tls-setup/_certs/dex.pem \
    --key=tls-setup/_certs/dex-key.pem

kubectl apply --namespace dex -f dex/dex.yaml
```

### Enabling proxy to Dex

As the Kubernetes is powered by `kind`, we need to do something extra to access the `Dex`.

There are some other ways [kind#702](https://github.com/kubernetes-sigs/kind/issues/702) but let's try this:

```sh
$ SVC_PORT="$(kubectl get -n dex svc/dex -o json | jq '.spec.ports[0].nodePort')"

# Create this proxy container
$ docker run -d --restart always \
    --name dex-kind-proxy-$SVC_PORT \
    --publish 127.0.0.1:$SVC_PORT:$SVC_PORT \
    --link kind-control-plane:target \
    --network kind \
    alpine/socat -dd \
    tcp-listen:$SVC_PORT,fork,reuseaddr tcp-connect:target:$SVC_PORT
```

Now we can access Dex by: `https://127.0.0.1:$SVC_PORT/`.

For example, issuing an HTTPS request to the discovery endpoint can verify the installation of Dex:

```sh
$ curl -k https://127.0.0.1:$SVC_PORT/.well-known/openid-configuration
{
  "issuer": "https://dex.dex.svc:32000",
  "authorization_endpoint": "https://dex.dex.svc:32000/auth",
  "token_endpoint": "https://dex.dex.svc:32000/token",
  "jwks_uri": "https://dex.dex.svc:32000/keys",
  "response_types_supported": [
    "code"
  ],
  "subject_types_supported": [
    "public"
  ],
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "scopes_supported": [
    "openid",
    "email",
    "groups",
    "profile",
    "offline_access"
  ],
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic"
  ],
  "claims_supported": [
    "aud",
    "email",
    "email_verified",
    "exp",
    "iat",
    "iss",
    "locale",
    "name",
    "sub"
  ]
}
```

But the issuer has exposed its URL though domain of `dex.dex.svc`, so we have to edit the `/etc/hosts` as the easy fix.

```sh
$ echo "127.0.0.1 dex.dex.svc" | sudo tee -a /etc/hosts
```

### Logging into the cluster

> Note: 
> 1. this `example-app` was copied from Dex's repo, [here](https://github.com/dexidp/dex/tree/master/examples/example-app`).
> 2. I've enabled `go mod` support so the life of playing with it is much easier.

```sh
# Make sure we're working from the Git repo's root folder
cd "$( git rev-parse --show-toplevel )"

cd example-app

go run . \
    --issuer https://dex.dex.svc:$SVC_PORT \
    --issuer-root-ca ../tls-setup/_certs/ca.pem \
    --debug
```

Now open browser and access: `http://127.0.0.1:5555/`
- Leave the form as is and click "Login";
- In "Log in to Your Account" page:
  - Email Address: `admin1@example.org`
  - Password: `secret`
- A series of tokens will be generated, copy down the **`ID Token`**, something like this:
```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjkzMzBkOTRhNGIzZTYwNjNiZTFmMmFhN2JhMWExMzY1ODZlY2MzMWMifQ.eyJpc3MiOiJodHRwczovL2RleC5kZXguc3ZjOjMyMDAwIiwic3ViIjoiQ2lWamJqMWhaRzFwYmpFc2IzVTljR1Z2Y0d4bExHUmpQV1Y0WVcxd2JHVXNaR005YjNKbkVnUnNaR0Z3IiwiYXVkIjoiZXhhbXBsZS1hcHAiLCJleHAiOjE2MDc3NTgxODgsImlhdCI6MTYwNzY3MTc4OCwiYXRfaGFzaCI6IlB3NWJxNF9TYkcwYUZtUkYyZDQwV3ciLCJlbWFpbCI6ImFkbWluMUBleGFtcGxlLm9yZyIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoiYWRtaW4xIn0.D_7kzzwlT5u9eq0KYrL64K_az2sO7iQ_5-Oz7nYHcHWQ8bBmxkH5NldsaZjzHKi0myo7EBJtb_6fqT4817h8Tf-FmGw_Ig0Fx-iA8c651L563qsy86s1usrrKyxQo-B6nZi-gvbY_K27KemNhgyGfLjl0PlvNWSUhoA94E3mpnEkdHs0H7Ni8iOgyOoNQV6TisrQgcr6blaVFJoMVhx4_XP1WnC3YZBX3vbGMCamu67BUP1KgnRbUwGqsuWntT-MuNuu8nOaBeIDGSrXFmqkUVGqIwGsG5bBsHqsfXtgePkhxXChhMUwQbUs3B4FkWITSJsjyrvCGEeGBjRtEH1w7A
```

The screenshots are captured like this:

![screenshots](misc/screenshots.png)

### Access Kubernetes by the token retrieved

Now we have the token, let's access it through raw API first:

```sh
# Retrieve the API Endpoint
$ kubectl cluster-info
Kubernetes master is running at https://127.0.0.1:55662
KubeDNS is running at https://127.0.0.1:55662/api/v1/namespaces/kube-system/services/kube-dns:dns/prox

$ APISERVER=https://127.0.0.1:55662 && \
BEARER_TOKEN="eyJhbGciOiJSUzI1NiIsImtpZCI6IjkzMzBkOTRhNGIzZTYwNjNiZTFmMmFhN2JhMWExMzY1ODZlY2MzMWMifQ.eyJpc3MiOiJodHRwczovL2RleC5kZXguc3ZjOjMyMDAwIiwic3ViIjoiQ2lWamJqMWhaRzFwYmpFc2IzVTljR1Z2Y0d4bExHUmpQV1Y0WVcxd2JHVXNaR005YjNKbkVnUnNaR0Z3IiwiYXVkIjoiZXhhbXBsZS1hcHAiLCJleHAiOjE2MDc3NTgxODgsImlhdCI6MTYwNzY3MTc4OCwiYXRfaGFzaCI6IlB3NWJxNF9TYkcwYUZtUkYyZDQwV3ciLCJlbWFpbCI6ImFkbWluMUBleGFtcGxlLm9yZyIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoiYWRtaW4xIn0.D_7kzzwlT5u9eq0KYrL64K_az2sO7iQ_5-Oz7nYHcHWQ8bBmxkH5NldsaZjzHKi0myo7EBJtb_6fqT4817h8Tf-FmGw_Ig0Fx-iA8c651L563qsy86s1usrrKyxQo-B6nZi-gvbY_K27KemNhgyGfLjl0PlvNWSUhoA94E3mpnEkdHs0H7Ni8iOgyOoNQV6TisrQgcr6blaVFJoMVhx4_XP1WnC3YZBX3vbGMCamu67BUP1KgnRbUwGqsuWntT-MuNuu8nOaBeIDGSrXFmqkUVGqIwGsG5bBsHqsfXtgePkhxXChhMUwQbUs3B4FkWITSJsjyrvCGEeGBjRtEH1w7A"

$ curl -k $APISERVER/api/v1/namespaces/default/pods/ --header "Authorization: Bearer $BEARER_TOKEN"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"admin1@example.org\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}

$ kubectl auth can-i --as admin1@example.org -n dex list pods
no
```

The good news is that Kubernetes has recognized the login user as `admin1@example.org` but still declined the access with `403 Forbidden`.

Why? It's because there is no permission, by default, granted to a new user like `admin1@example.org`.

### Kubernetes Authorization

As you may have seen, authentication is delegated to `Dex` but authorization is handled by Kubernetes itself.

```sh
# Make sure we're working from the Git repo's root folder
cd "$( git rev-parse --show-toplevel )"

kubectl apply -f manifests/authorization.yaml

kubectl auth can-i --as admin1@example.org -n dex list pods
yes

curl -k -s $APISERVER/api/v1/namespaces/dex/pods/ -H "Authorization: Bearer $BEARER_TOKEN" | jq '.items[].metadata.name'
"dex-5f97556766-kcfvl"
```

Yes! We now can access pods within `dex` namespace, as per the permissions granted.

### Generate `kubeconfig`

It's common to generate and distribute such a token by constructing a `kubeconfig` file.

```sh
# Generate a kubeconfig
$ cat > ~/.kube/config-kind <<EOF
apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    server: $APISERVER
    insecure-skip-tls-verify: true
  name: kind
contexts:
- context:
    cluster: kind
    namespace: default
    user: admin1@example.org
  name: admin1@example.org
current-context: admin1@example.org
users:
- name: admin1@example.org
  user:
    token: $BEARER_TOKEN
EOF

$ kubectl get pods -n dex --kubeconfig ~/.kube/config-kind
NAME                   READY   STATUS    RESTARTS   AGE
dex-5f97556766-kcfvl   1/1     Running   1          1h28m
```

Perfect!

## Clean Up

```sh
# Delete the kind cluster
kind delete cluster --name dex-ldap-cluster

# Delete the Socat proxy container
docker rm -f dex-kind-proxy-32000
```

## References

- https://dexidp.io/docs/kubernetes/
- https://gist.github.com/yifan-gu/6fce1016a4cfe4c40d9c
