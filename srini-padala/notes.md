
Refer the [cncf](cncf\readme.md) folder for the yamls

Application Urls

- [OpenFaas UI](https://openfaas.aks.srinipadala.xyz/ui/)
- [Promethues](https://prometheus.aks.srinipadala.xyz/)
- [Jaeger](https://tracing.aks.srinipadala.xyz/)
- [Harbor](https://registry.aks.srinipadala.xyz/)
- [Linkerd](https://linkerd-dashboard.aks.srinipadala.xyz/)
- [Tekton](https://tekton.aks.srinipadala.xyz/)
- [Grafana](https://grafana.aks.srinipadala.xyz/)
- [Application](https://conexp.aks.srinipadala.xyz/)

- [Sourcode in github](https://github.com/seenu433/conexp-mvp)

##Rook Installation

```
kubectl apply -f cncf/common.yaml
kubectl apply -f cncf\operator.yaml
kubectl apply -f cncf\cluster.yaml
kubectl apply -f cncf\storageclass.yaml
```

##Harbor Installation
```
kubectl create namespace harbor-system
helm repo add harbor https://helm.goharbor.io

helm install harbor --namespace harbor-system harbor/harbor --set expose.ingress.hosts.core=registry.aks.srinipadala.xyz --set expose.ingress.hosts.notary=notary.aks.srinipadala.xyz  --set expose.tls.secretName=ingress-cert-harbor --set expose.tls.notarySecretName=ingress-cert-notary --set expose.ingress.annotations.kubernetes\.io/ingress\.class=nginx --set expose.ingress.annotations.cert-manager\.io/cluster-issuer=letsencrypt  --set expose.ingress.annotations.external-dns\.alpha\.kubernetes\.io/target=aks.srinipadala.xyz --set persistence.enabled=true --set externalURL=https://registry.aks.srinipadala.xyz --set harborAdminPassword=admin  --set persistence.persistentVolumeClaim.registry.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.chartmuseum.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.jobservice.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.database.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.redis.storageClass=rook-ceph-block
```

```
https://registry.aks.srinipadala.xyz/
admin
FTA@CNCF0n@zure3
```

##MySQL installation

Deploy Mysql
```
kubectl create ns mysql
helm install mysql stable/mysql  --set mysqlRootPassword=FTA@CNCF0n@zure3,mysqlUser=ftacncf,mysqlPassword=FTA@CNCF0n@zure3,mysqlDatabase=conexp-mysql,persistence.storageClass=rook-ceph-block -n mysql
```
Create the databases 
```
kubectl run -n conexp-mysql -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
apt-get update && apt-get install mysql-client -y
mysql -h mysql -p
show databases;

CREATE DATABASE conexpweb;

CREATE DATABASE conexpapi;
USE conexpapi;

CREATE TABLE CostCenters(
   CostCenterId int(11)  NOT NULL,
   SubmitterEmail text NOT NULL,
   ApproverEmail text NOT NULL,
   CostCenterName text NOT NULL,
   PRIMARY KEY ( CostCenterId )
);

INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (1, 'faisalm@microsoft.com', 'faisalm@microsoft.com','123E42');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (2, 'srpadala@microsoft.com', 'srpadala@microsoft.com','456C14');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (3, 'ssarwa@microsoft.com', 'ssarwa@microsoft.com','456C14');

USE conexpapi;
GRANT ALL PRIVILEGES ON *.* TO 'ftacncf'@'%';

USE conexpweb;
GRANT ALL PRIVILEGES ON *.* TO 'ftacncf'@'%';
```

## Vitess Installation (Note: Either do the SQL Installtion as above or Vitess, not both)

Clone the Vitess Git repo
```
sudo git clone https://github.com/vitessio/vitess.git 
```
Navigate to the following folder:
```
~/Vitess/vitess/examples/operator
```
Run the following kubectl commands:
```
kubectl apply -f operator.yaml 

kubectl apply -f 101_initial_cluster.yaml
```
Get the POD running the Vitess (from the pf.sh file):
```
kubectl get deployment --selector="planetscale.com/component=vtgate"
```
Expose the deploy as a svc, get the service YAML, and edit it/clean it
To Do: Add the example YAML file.

Install MySQL Client Locally
```
apt install mysql-client
```

##OpenFaaS

```
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update

kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password="FTA@CNCF0n@zure3"

helm install openfaas openfaas/openfaas -f cncf\openfaas-values.yaml -n openfaas
```

```
https://openfaas.aks.srinipadala.xyz/ui/
admin
FTA@CNCF0n@zure3
```

Install the Nats Connector
```
kubectl apply -f cncf\nats-connector.yaml
```
##Prometheus

```
kubectl create ns monitoring
helm install prometheus stable/prometheus-operator -f cncf\prometheus-values.yaml -n monitoring
```
```
https://grafana.aks.srinipadala.xyz/
admin
FTA@CNCF0n@zure3
https://prometheus.aks.srinipadala.xyz/
```

##Jaeger

```
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

kubectl create ns tracing
helm install jaeger jaegertracing/jaeger -f cncf\jaeger-values.yaml -n tracing
```
```
https://tracing.aks.srinipadala.xyz/
```

##Linkerd

Deploy Linkered
```
curl -sL https://run.linkerd.io/install | sh

export PATH=$PATH:$HOME/.linkerd2/bin

linkerd version

linkerd check --pre

sudo apt install step

step certificate create identity.linkerd.cluster.local ca.crt ca.key --profile root-ca --no-password --insecure

step certificate create identity.linkerd.cluster.local issuer.crt issuer.key --ca ca.crt --ca-key ca.key --profile intermediate-ca --not-after 8760h --no-password --insecure

linkerd install --identity-trust-anchors-file ca.crt --identity-issuer-certificate-file issuer.crt --identity-issuer-key-file issuer.key | kubectl apply -f -
```

Integrate Openfaas with Linkerd
```
kubectl -n openfaas get deploy gateway -o yaml | linkerd inject --skip-outbound-ports=4222 - | kubectl apply -f -
kubectl annotate namespace openfaas-fn linkerd.io/inject=enabled
```

Integrate Nginx Ingress controller with Linkerd
```
kubectl get deploy/nginx-nginx-ingress-controller -n ingress-basic -o yaml | linkerd inject - | kubectl apply -f - 
```

Enable Oauth on Linkerd Dashboard

- Register an application in Azure AD to match the Sign-on URL as https://your_website_fqdn/oauth2/callback
- update OAUTH2_PROXY_AZURE_TENANT, OAUTH2_PROXY_CLIENT_ID, OAUTH2_PROXY_CLIENT_SECRET

```
kubectl apply -f cncf\linkerd-ingress.yaml -n linkerd
```

Linkerd metrics integration with Prometheus
```
kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml -n monitoring
kubectl edit prometheus  prometheus-prometheus-oper-prometheus  -n monitoring

Add the additionalScrapeConfigs as below
  ....
  ....
  serviceMonitorSelector:
    matchLabels:
      team: frontend
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: prometheus-additional.yaml
  ....
  ....
```

Linkerd integration with Jaeger
```
kubectl  apply -f cncf\opencesus-collector.yaml -n tracing

kubectl annotate namespace openfaas-fn config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace openfaas config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace ingress-basic config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace conexp-mvp config.linkerd.io/trace-collector=oc-collector.tracing:55678
```

https://linkerd-dashboard.aks.srinipadala.xyz/

##Tekton
Install Tekton pipelines
```
kubectl apply -f https://storage.googleapis.com/tekton-releases/latest/release.yaml

kubectl apply -f tekton-default-configmap  -n  tekton-pipelines
kubectl apply -f tekton-pvc-configmap -n  tekton-pipelines
kubectl apply -f tekton-feature-flags-configmap.yaml -n  tekton-pipelines
```
Install Tekton Triggers
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```
Install Tekton Dashboard
```
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.5.2/tekton-dashboard-release.yaml
kubectl apply -f tekton-dashboard-ingress.yaml
```

##App Installation

Build and push the containers
```
docker login registry.aks.srinipadala.xyz
conexp
FTA@CNCF0n@zure3

cd Contoso.Expenses.API
docker build -t registry.aks.srinipadala.xyz/conexp/api:latest .
docker push registry.aks.srinipadala.xyz/conexp/api:latest

cd ..
docker build -t registry.aks.srinipadala.xyz/conexp/web:latest -f Contoso.Expenses.Web/Dockerfile .
docker push registry.aks.srinipadala.xyz/conexp/web:latest

docker build -t registry.aks.srinipadala.xyz/conexp/emaildispatcher:latest -f Contoso.Expenses.OpenFaaS/Dockerfile .
docker push registry.aks.srinipadala.xyz/conexp/emaildispatcher:latest
```

```
kubectl create ns conexp-mvp
kubectl annotate namespace conexp-mvp linkerd.io/inject=enabled
kubectl annotate namespace config.linkerd.io/skip-outbound-ports="4222"
```

Create the registry credentials in teh deployment namespaces
```
kubectl create secret docker-registry regcred --docker-server="https://registry.aks.srinipadala.xyz" --docker-username=conexp  --docker-password=FTA@CNCF0n@zure3  --docker-email=srpadala@microsoft.com -n conexp-mvp
kubectl create secret docker-registry regcred --docker-server="https://registry.aks.srinipadala.xyz" --docker-username=conexp  --docker-password=FTA@CNCF0n@zure3  --docker-email=srpadala@microsoft.com -n openfaas-fn
```

##Tekton - App Deployment


```
kubectl create ns conexp-mvp-devops

kubectl apply -f cncf\webhook-role.yaml -n conexp-mvp-devops
kubectl apply -f cncf\admin-role.yaml -n conexp-mvp-devops

kubectl apply -f cncf\create-ingress.yaml -n conexp-mvp-devops
kubectl apply -f cncf\create-webhook.yaml -n conexp-mvp-devops
```

Update Secret (basic-user-pass) for registry credentails, TriggerBinding for registry name,namespaces in triggers.yaml
```
kubectl apply -f cncf\pipeline.yaml -n conexp-mvp-devops
kubectl apply -f cncf\triggers.yaml -n conexp-mvp-devops
```

Roles and bindings in the deployment namespace
```
kubectl apply -f cncf\deploy-rolebinding.yaml -n conexp-mvp
kubectl apply -f cncf\deploy-rolebinding.yaml -n openfaas-fn
```

Generate PAT token for the repo -> public_repo, admin:repo_hook, update the token in github-secret.yaml
```
kubectl apply -f cncf\github-secret.yaml -n conexp-mvp-devops
```

Update servicename, secretname, domain in the ingress-run.yaml
Update org/user/repo/domain in the webhook-run.yaml
```
kubectl apply -f cncf\ingress-run.yaml  -n conexp-mvp-devops
kubectl apply -f cncf\webhook-run.yaml -n conexp-mvp-devops
```
