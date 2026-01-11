kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.crds.yaml


# Install cert-manager
helm repo add jetstack https://charts.jetstack.io --force-update

helm install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.0 \
  --set crds.enabled=true

