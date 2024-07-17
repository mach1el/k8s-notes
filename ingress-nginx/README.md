## Install Ingress-Nginx

* Add helm repo
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

* Install ingress withou using CloudFlare
```
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.10.0 \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.config.enable-real-ip="true" \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.service.external.enabled=true \
  --set controller.service.externalTrafficPolicy="Local" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-proxy-protocol"="*" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type"="ip" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-backend-protocol"="tcp"
```

* Install ingress with CloudFlare
```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.10.0 \
  --create-namespace \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --set controller.config.enable-real-ip="true"
```

## White list IPs

Add annotations to `Ingress` resource

```
annotations:
  nginx.ingress.kubernetes.io/whitelist-source-range: list-of-ip-range
```

## Configure to get a real client IP from CloudFlare.

Add data for your ingress-nginx config map

```
data:
  enable-real-ip: "true"
  server-snippet: |
    real_ip_header CF-Connecting-IP;
```