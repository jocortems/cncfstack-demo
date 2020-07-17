
Refer the [cncf](yml\readme.md) folder for the yamls

Application Urls

- [OpenFaas UI](https://openfaas.cncftrail.mycompany.com/ui/)
- [Promethues](https://prometheus.cncftrail.mycompany.com/)
- [Jaeger](https://tracing.cncftrail.mycompany.com/)
- [Harbor](https://registry.cncftrail.mycompany.com/)
- [Linkerd](https://linkerd-dashboard.cncftrail.mycompany.com/)
- [Tekton](https://tekton.cncftrail.mycompany.com/)
- [Grafana](https://grafana.cncftrail.mycompany.com/)
- [Application](https://conexp.cncftrail.mycompany.com/)

- [Sourcode in github](https://github.com/seenu433/conexp-mvp)

#Setup

Create CNAME alias to the ingress IP for services to be deployed
- registry
- notary
- openfaas
- grafana
- prometheus
- tracing
- linkerd-dashboard
- tekton
- conexp
- conexp-mvp.cicd


Clone the repo

Set the variable to be used as the top level domain for this exercise
```
topLevelDomain=cncftrail.mycompany.com
```

##Rook Installation

```
kubectl apply -f yml\rook-common.yaml
kubectl apply -f yml\rook-operator.yaml
kubectl apply -f yml\rook-cluster.yaml
kubectl apply -f yml\rook-storageclass.yaml
```

##Harbor Installation
```
registryHost=registry.$topLevelDomain
notaryHost=notary.$topLevelDomain
externalUrl=http://registry.$topLevelDomain

kubectl create namespace harbor-system
helm repo add harbor https://helm.goharbor.io

helm install harbor --namespace harbor-system harbor/harbor --set expose.ingress.hosts.core=$registryHost --set expose.ingress.hosts.notary=$notaryHost --set expose.ingress.annotations.kubernetes\.io/ingress\.class=nginx --set persistence.enabled=true --set externalURL=$externalUrl --set harborAdminPassword=admin  --set persistence.persistentVolumeClaim.registry.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.chartmuseum.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.jobservice.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.database.storageClass=rook-ceph-block --set persistence.persistentVolumeClaim.redis.storageClass=rook-ceph-block
```

```
echo $externalUrl
admin
admin
```

##MySQL installation

Deploy Mysql
```
kubectl create ns mysql
helm install mysql stable/mysql  --set mysqlRootPassword=FTA@CNCF0n@zure3,mysqlUser=ftacncf,mysqlPassword=FTA@CNCF0n@zure3,mysqlDatabase=conexp-mysql,persistence.storageClass=rook-ceph-block -n mysql
```
Create the databases 
```
kubectl run -n mysql -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
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

INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (1, 'user1@mycompany.com', 'user1@mycompany.com','123E42');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (2, 'user2@mycompany.com', 'user2@mycompany.com','456C14');
INSERT INTO CostCenters (CostCenterId, SubmitterEmail,ApproverEmail,CostCenterName)  values (3, 'user3@mycompany.com', 'user3@mycompany.com','456C14');

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
openfaasHost=openfaas.$topLevelDomain
sed -i "s/{openfaas-host}/$openfaasHost/g" yml\openfaas-values.yaml

helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update

kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml

kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password="FTA@CNCF0n@zure3"

helm install openfaas openfaas/openfaas -f yml\openfaas-values.yaml -n openfaas
```

```

echo http://$openfaasHost/ui/
admin
FTA@CNCF0n@zure3
```

Install the Nats Connector
```
kubectl apply -f yml\openfaas-nats-connector.yaml
```
##Prometheus

```
grafanaHost=grafana.$topLevelDomain
prometheusHost=prometheus.$topLevelDomain
sed -i "s/{grafanaHost}/$grafanaHost/g" yml\prometheus-values.yaml 
sed -i "s/{prometheusHost}/$prometheusHost/g" yml\prometheus-values.yaml 

kubectl create ns monitoring
helm install prometheus stable/prometheus-operator -f yml\prometheus-values.yaml -n monitoring
```
```
echo http://$grafanaHost/
admin
FTA@CNCF0n@zure3
echo http://$prometheusHost/
```

##Jaeger

```
tracingHost=tracing.$topLevelDomain
sed -i "s/{tracingHost}/$tracingHost/g" yml\jaeger-values.yaml 

helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

kubectl create ns tracing
helm install jaeger jaegertracing/jaeger -f yml\jaeger-values.yaml -n tracing
```
```
echo http://$tracingHost/
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

Expose the dashboard
```
linkerdDashboard=linkerd-dashboard.$topLevelDomain

sed -i "s/{linkerdDashboard}/$linkerdDashboard/g" yml\linkerd-ingress.yaml

kubectl apply -f yml\linkerd-ingress.yaml -n linkerd
```

Linkerd metrics integration with Prometheus
```
kubectl create secret generic additional-scrape-configs --from-file=linkerd-prometheus-additional.yaml -n monitoring
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
kubectl  apply -f yml\linkerd-opencesus-collector.yaml -n tracing

kubectl annotate namespace openfaas-fn config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace openfaas config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace ingress-basic config.linkerd.io/trace-collector=oc-collector.tracing:55678
kubectl annotate namespace conexp-mvp config.linkerd.io/trace-collector=oc-collector.tracing:55678
```

```
echo http://$linkerdDashboard/
admin/admin
```

##Tekton
Install Tekton pipelines
```
kubectl apply -f https://storage.googleapis.com/tekton-releases/latest/release.yaml

kubectl apply -f yml\tekton-default-configmap.yaml  -n  tekton-pipelines
kubectl apply -f yml\tekton-pvc-configmap.yaml -n  tekton-pipelines
kubectl apply -f yml\tekton-feature-flags-configmap.yaml -n  tekton-pipelines
```
Install Tekton Triggers
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```
Install Tekton Dashboard
```
tektonHost=tekton.$topLevelDomain

sed -i "s/{tektonHost}/$tektonHost/g" yml\tekton-dashboard-ingress.yaml

kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.5.2/tekton-dashboard-release.yaml
kubectl apply -f yml\tekton-dashboard-ingress.yaml
```
```
echo http://$tektonHost/
```

##App Installation

Create the Project and User in Harbor
- Login to Harbor
- Add a new Project with name conexp
- Add a new User under Administration with username as concep and password as FTA@CNCF0n@zure3
- Associate the user with the conexp project under Memebers tab with a Developer role

Build and push the containers
```
docker login $registryHost
conexp
FTA@CNCF0n@zure3

cd src\Contoso.Expenses.API
docker build -t $registryHost/conexp/api:latest .
docker push $registryHost/conexp/api:latest

cd ..
docker build -t $registryHost/conexp/web:latest -f Contoso.Expenses.Web/Dockerfile .
docker push $registryHost/conexp/web:latest

docker build -t $registryHost/conexp/emaildispatcher:latest -f Contoso.Expenses.OpenFaaS/Dockerfile .
docker push $registryHost/conexp/emaildispatcher:latest

cd ..

```

```
kubectl create ns conexp-mvp
kubectl annotate namespace conexp-mvp linkerd.io/inject=enabled
kubectl annotate namespace config.linkerd.io/skip-outbound-ports="4222"
```

Create the registry credentials in teh deployment namespaces
```
kubectl create secret docker-registry regcred --docker-server="https://$registryHost" --docker-username=conexp  --docker-password=FTA@CNCF0n@zure3  --docker-email=user@mycompany.com -n conexp-mvp
kubectl create secret docker-registry regcred --docker-server="https://$registryHost" --docker-username=conexp  --docker-password=FTA@CNCF0n@zure3  --docker-email=user@mycompany.com -n openfaas-fn
```

##Tekton - App Deployment


```
kubectl create ns conexp-mvp-devops

kubectl apply -f yml\app-webhook-role.yaml -n conexp-mvp-devops
kubectl apply -f yml\app-admin-role.yaml -n conexp-mvp-devops

kubectl apply -f yml\app-create-ingress.yaml -n conexp-mvp-devops
kubectl apply -f yml\app-create-webhook.yaml -n conexp-mvp-devops
```

Update Secret (basic-user-pass) for registry credentails, TriggerBinding for registry name,namespaces in triggers.yaml
Create a SendGrid Account and set an API key for use
```
sendGridApiKey=<<set the api key>>
appHostName=conexp.$topLevelDomain

sed -i "s/{registryHost}/$registryHost/g" yml\app-triggers.yaml

sed -i "s/{SENDGRIDAPIKEYRELACE}/$sendGridApiKey/g" yml\app-pipeline.yaml
sed -i "s/{APPHOSTNAMEREPLACE}/$appHostName/g" yml\app-pipeline.yaml

kubectl apply -f yml\app-pipeline.yaml -n conexp-mvp-devops
kubectl apply -f yml\app-triggers.yaml -n conexp-mvp-devops
```

Roles and bindings in the deployment namespace
```
kubectl apply -f yml\app-deploy-rolebinding.yaml -n conexp-mvp
kubectl apply -f yml\app-deploy-rolebinding.yaml -n openfaas-fn
```

Generate PAT token for the repo -> public_repo, admin:repo_hook, set the pat token below
```
patToken=<<set the pat tokne>>

sed -i "s/{patToken}/$patToken/g" yml\app-github-secret.yaml

kubectl apply -f yml\app-github-secret.yaml -n conexp-mvp-devops
```

set org/user/repo of the source code repo variables below
```
cicdWebhookHost=conexp-mvp.cicd.$topLevelDomain

gitHubOrg=<<set the name of the github org>>
gitHubUser=<<set the name of the github user>>
gitHubRepo=<<set the name of the github repo>>

sed -i "s/{cicdWebhook}/$cicdWebhookHost/g" yml\app-ingress-run.yaml

kubectl apply -f yml\app-ingress-run.yaml  -n conexp-mvp-devops

sed -i "s/{cicdWebhook}/$cicdWebhookHost/g" yml\app-webhook-run.yaml
sed -i "s/{mygithub-org-replace}/$gitHubOrg/g" yml\app-webhook-run.yaml
sed -i "s/{mygithub-user-replace}/$gitHubUser/g" yml\app-webhook-run.yaml
sed -i "s/{mygithub-repo-replace}/$gitHubRepo/g" yml\app-webhook-run.yaml

kubectl apply -f yml\app-webhook-run.yaml -n conexp-mvp-devops
```
