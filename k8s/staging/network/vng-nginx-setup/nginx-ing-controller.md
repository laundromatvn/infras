1. clear all nginx services

2. run this
```
helm install nginx-ingress-controller oci://ghcr.io/nginxinc/charts/nginx-ingress --namespace kube-system
```

3. add ing