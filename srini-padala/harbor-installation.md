
## Harbor Installation

```
kubectl create namespace harbor-system
helm repo add harbor https://helm.goharbor.io

helm install harbor --namespace harbor-system harbor/harbor --set expose.ingress.hosts.core=harbor.aks.srinipadala.xyz --set expose.ingress.hosts.notary=notary.aks.srinipadala.xyz  --set expose.tls.secretName=ingress-cert-harbor --set expose.ingress.annotations.kubernetes\.io/ingress\.class=nginx --set expose.ingress.annotations.cert-manager\.io/cluster-issuer=letsencrypt --set persistence.enabled=false --set externalURL=https://harbor.aks.srinipadala.xyz --set harborAdminPassword=admin
```