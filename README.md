# Azure-workshop
My very basic workshop about Azure

## Install Requirements

### Azure CLI
- OSX: *brew update && brew install azure-cli*
- Debian/Ubuntu: *curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash* 
- Arch: *yaourt -S azure-cli*
- Other Linux: *curl -L https://aka.ms/InstallAzureCli | bash*

### kubectl
- *az aks install-cli*

### helm
- *curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash*

## Basic app in azure
```bash
az aks get-credentials --resource-group azure-workshop --name azure-workshop

# Create a K8s namespace for the ingress resources
kubectl create namespace ingress-basic# Add the ingress-nginx repository

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx  # Use Helm to deploy an NGINX ingress controller 

helm install nginx-ingress ingress-nginx/ingress-nginx \
     --namespace ingress-basic \
     --set controller.replicaCount=2 \
     --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
     --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
     
```


 
