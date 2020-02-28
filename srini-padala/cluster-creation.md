# AKS Lab Setup

This document outlines the steps to setup a fully fucntional AKS cluster with required components

 - AKS cluster creation
 - Connect from Local Machine
 - Access the Kubernetes Dashboard
 - Create a DNS Zone
 - Setup Ingress Controller
 - Setup Cert Manager
 - Setup ExternalDNS

## Prerequisites

 - [Install Azure Cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest)
 - Install [Docker for Windows](https://docs.docker.com/docker-for-windows/install/)
 - Install Choclatey
    - Open Powershell in "Run as Administrator" mode and run the below command
    ```
        Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')) 
    ```
 - Install Helm
    ```
        choco install kubernetes-helm
    ```
 - Purchase a domain from [Godaddy](https://www.godaddy.com/)
    `Mostly the doamin xyz costs $1 `

## AKS Cluster creation

```
# Create the resource group
az group create --name aks-demo --location eastus
```

```
# Create vnet and subnet
az network vnet create --resource-group aks-demo --name aks-demo-vnet --address-prefixes 10.0.0.0/8 --subnet-name default --subnet-prefix 10.240.0.0/16
# Note the vnetid, subnetid from the output

# Create the service principal
az ad sp create-for-rbac --skip-assignment --name aks-demo-sp
# Note the appid, password from the output

# Assignt the contributor role to the SP on the Vnet
az role assignment create --assignee <<appid>> --scope <<vnetid>> --role Contributor
```

```
# Create the AKS cluster
az aks create --resource-group aks-demo --name aks --node-count 2 --kubernetes-version 1.15.7 --generate-ssh-keys --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --network-plugin azure --network-policy calico --service-cidr 10.0.0.0/16 --dns-service-ip 10.0.0.10 --docker-bridge-address 172.17.0.1/16 --vnet-subnet-id <<subnetid>> --service-principal <<appid>> --client-secret <<password>> --enable-cluster-autoscaler --min-count 2 --max-count 3 --node-count 2 --zones 1 2
```

```
# Create Container Registry
az acr create --resource-group aks-demo --name <<alias>> --sku Basic

# Attach the container registry to the cluster
az aks update -n aks -g aks-demo --attach-acr <<alias>>
```

```
# [Create Log Analytics Workspace](https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace)
# Enable Monitoring on the cluster
az resource list --resource-type Microsoft.OperationalInsights/workspaces -o json
# Note the workspaceid from the output
az aks enable-addons -a monitoring -n aks -g aks --workspace-resource-id <<workspaceid>>
```

## Connect from local machine
Required to issue kubectl commands and interact with the cluster

```
#Assign user permissions for RBAC
az aks show --resource-group aks-demo --name aks --query id -o tsv
# Note the <<clusterid>> from the output

# Get the current user
az account show --query user.name -o tsv
# Note the <<userprincipal>> from the output

# Get the account id for the current user
az ad user show --upn-or-object-id <<userprincipal>> --query objectId -o tsv
# Note the <<account id>> from the output

# Assign the 'Cluster Admin' role to the user
az role assignment create  --assignee <<account id>>  --scope <<clusterid>> --role "Azure Kubernetes Service Cluster Admin Role"

# Get the kubeconfig
az aks get-credentials --name aks --resource-group aks-demo

# Get the nodes to verify the connectivity
kubectl get nodes
```

## Access Kubernetes Dashboard
Use the dashboard to get familiar with objects

```
# Create the cluster role and binding
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard

# Access the kubernetes dashboard (open in new terminal)
az aks browse --resource-group aks-demo --name aks
```

## Create DNS Zone
DNS zone to facilitate automatic creation of the A and alias record for teh services exposed through ingress

```
# Create the DNS Zone in the resource group
az network dns zone create -g aks-demo -n <<dns purchased>>

# Update the Names servers in godaddy website
Get the [Nameservers](https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns) from the dns zone created and update them in [godaddy website](https://www.godaddy.com/help/change-nameservers-for-my-domains-664)
```

## Create Ingress
Ingress is the gateway for all communication to the cluster

```
# Create a namespace
kubectl create namespace ingress-basic

# Add the helm chart to local
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

# Install nginx ingress using helm
helm install nginx stable/nginx-ingress --namespace ingress-basic --set controller.replicaCount=2 --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux

# Get the external IP of the load balancer
kubectl get svc --all-namespaces
# Identify the service which has external ip and note the external IP

# Create an A record in the dns zone with name as "aks" and IP as noted above
az network dns record-set a add-record --resource-group aks-demo --zone-name <<purchased domain>>  --record-set-name "aks" --ipv4-address <<External IP noted above>>
```

## Install Cert Manager for automatic certificate creation
Facilitates automatic certs creation to enable ssl

```
# Install the CustomResourceDefinition resources separately
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml --namespace ingress-basic

# Label the ingress-basic namespace to disable resource validation
kubectl label namespace ingress-basic certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager --namespace ingress-basic --version v0.12.0 jetstack/cert-manager --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer
```

```
# Update the email adress to yours and save the below content as cluster-issuer.yaml

apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: MY_EMAIL_ADDRESS
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          class: nginx

```
```
# apply the yaml to create the cluster issuer
kubectl apply -f cluster-issuer.yaml --namespace ingress-basic
```
## Setup External DNS
Facilitates the creation of the DNS record in the DNS zone 
```
# Create a service principal to setup external dns
az ad sp create-for-rbac -n aks-demo-externaldns-sp
# Note the appid, password, tenant

# Get the id of the resource group
az group show --name aks-demo
# Note the <<id>> from the output

# Assign reader permission to the sp on the resourcegroup
az role assignment create --role "Reader" --assignee <<appid>> --scope <<resourcegroup id>>  

# Get the id of the dns zone
az network dns zone show --name <<purchased dns>> -g aks-demo 
# Note the <<id>> from the output

# Assign contributor permission to the sp on the dns zone
az role assignment create --role "Contributor" --assignee <<appid>> --scope <<dns zone id>> 
```

```
# Create a config file for external dns to access the dns zone to create the records
# Copy the contents below, Update the values and save it as azure.json
{
  "tenantId": "<<tenant>>",
  "subscriptionId": "<<aks-demo subscription id>>",
  "resourceGroup": "aks-demo",
  "aadClientId": "<<appid>>",
  "aadClientSecret": "<<password>>"
}
```

```
# Create a namespace for external-dns
kubectl create ns external-dns

# Create a kubernetes secret from teh above json file
kubectl create secret generic azure-config-file --from-file=azure.json --namespace external-dns
```

```
# Copy the below yaml to a file external-dns.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"] 
  resources: ["ingresses"] 
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:latest
        args:
        - --source=ingress
        - --domain-filter=<<purchased domain>>
        - --provider=azure
        - --azure-resource-group=aks-demo
        - --log-level=debug
        - --registry=txt
        - --txt-owner-id=k8s
        volumeMounts:
        - name: azure-config-file
          mountPath: /etc/kubernetes
          readOnly: true
      volumes:
      - name: azure-config-file
        secret:
          secretName: azure-config-file

```

```
# Update the purchased domain in the config and apply the yaml to create the deployment
kubectl apply -f external-dns.yaml --namespace external-dns
```

## Build Hello World Docker Image

```
# Download the contents of nginx-hellow-world folder to a local folder

# Open command prompt to the folder

# build the docker image
docker build -t <<alias>.azurecr.io/demo/nginx-hello-world:latest .

# Login to ACR
az acr login --name <<alias>>

# Push the docker image to acr
docker push <<alias>.azurecr.io/demo/nginx-hello-world:latest
```

## Deploy the container

```
# Copy the contents below into nginx-hello-world.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: <<alias>>.azurecr.io/demo/nginx-hello-world:latest
        ports:
        - containerPort: 80
      nodeSelector: 
        "beta.kubernetes.io/os": linux
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: frontend-ingress-rules
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    external-dns.alpha.kubernetes.io/hostname: hello.aks.<<purchased domain>>
    external-dns.alpha.kubernetes.io/target: aks.<<purchased domain>>
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - hosts:
      - hello.aks.<<purchased domain>>
      secretName: tls-frontend-secret
  rules:
    - host: hello.srinipadala.xyz
      http:
       paths:
       - backend:
            serviceName: frontend-svc
            servicePort: 80
         path: /(.*)
```

```
# Update the alias for the acr name and the host name on the ingress to the purchased domain

# Create the deployment
kubectl apply -f nginx-hello-world.yaml

# Access the application using hello.aks.<<purchased domain>>
```
