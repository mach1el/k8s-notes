## White list IPs

Add annotations to `Ingress` resource

```
annotations:
  nginx.ingress.kubernetes.io/whitelist-source-range: list-of-ip-range
```

## Configure to get a real client IP if it is hosted by some cloud hoster.

Add data for your ingress-nginx config map

```
data:
  enable-real-ip: "true"
  server-snippet: |
    real_ip_header CF-Connecting-IP;
```