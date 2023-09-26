## Configure cert-manager for SSL without CloudFlare proxied

Some of our domain names are not issued certification by CloudFlare, so when we want to route our service through SSL, we are able to manage this using cert-manager. cert-manager is an open source project that builds on top of Kubernetes to provide X.509 certificates and issuers as first class resource types. For more detail, take a look to its [documentation](https://cert-manager.io/docs/).

### Install cert-manager to K8s cluster using Helm chart

Steps

- **1. Add Helm repository**
  ```
  helm repo add jetstack https://charts.jetstack.io
  ```

- **2. Update Helm chart repository**
  ```
  helm repo update
  ```

- **3. Install `CustomResourceDefinitions`**
  ```
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml
  ```

- **4. Install cert-manager**
  ```
  helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0
  ```

Now, wait the pods to be coming up, you can check it by command
```
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6957cdbc5d-r8d58              1/1     Running   0          3h36m
cert-manager-cainjector-56c499cdb8-ps7jz   1/1     Running   0          3h36m
cert-manager-webhook-5585dcddb-trnjx       1/1     Running   0          3h36m
```

![cert-manager-overview](/ops/assets/images/kubernetes/cert-manager-overview.svg)

### Create ClusterIssuer

`Issuers`, and `ClusterIssuers`, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests.

```
$ cat << EOF | kubectl apply -f 
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: user@mail
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

> *Note: should use validately email address*

Validate the ClusterIssuer

```
$ kubectl get clusterissuer
NAME               READY   AGE
letsencrypt   True    0h1m
```

### Create Certificate resource

cert-manager has the concept of Certificates that define a desired X.509 certificate which will be renewed and kept up to date. A Certificate is a namespaced resource that references an Issuer or ClusterIssuer that determine what will be honoring the certificate request.

```
$ cat << EOF | kubectl apply -f 
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp
  namespace: myapp-space
spec:
  secretName: myapp-secret
  dnsNames:
    - myapp.example.com
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
EOF
```

Check the certificate

```
$ kubectl get certificate -n myapp-space
NAME                  READY   SECRET                AGE
myapp          True    myapp-secret   0h1m
myapp-secret   True    myapp-secret   0h1m
```

### Update ingress object for our service

Here is the ingress object for our application to serve the external traffic to the cluster over HTTPS securely:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    acme.cert-manager.io/http01-edit-in-place: 'true'
    cert-manager.io/cluster-issuer: letsencrypt
  name: myapp
  namespace: myapp-space
spec:
  defaultBackend:
    service:
      name: myapp
      port:
        number: 3333
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-secret
```

## Configure cert-manager for SSL with CloudFlare proxied

To use Cloudflare, you may use one of two types of tokens. API Tokens allow application-scoped keys bound to specific zones and permissions, while API Keys are globally-scoped keys that carry the same permissions as your account.

API Tokens are recommended for higher security, since they have more restrictive permissions and are more easily revocable.

### API Tokens

Tokens can be created at User Profile > API Tokens > API Tokens. The following settings are recommended:

* Permissions:
  * Zone - DNS - Edit
  * Zone - Zone - Read
* Zone Resources:
  * Include - All Zones

To create a new `Issuer`, first make a Kubernetes secret containing your new API token:

```
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <API Token>
```

Then in your `Issuer` manifest:

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: user@mail
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - dns01:
          cloudflare:
            email: user@mail
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

### Create wildcard certificate

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-cert
  namespace: myapp-space
spec:
  secretName: wildcard-cert-tls
  dnsNames:
  - "*.example.com"
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
```