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

## Basic calculator app in azure

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

### Kubernetes parts

Application is deployed by applying a YAML file to the cluster. The two deployments and each corresponding service are created using the YAML file below.

Create `calculator.yml`
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: chonyy/calculator_api:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  ports:
  - port: 80
  selector:
    app: api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website
spec:
  replicas: 3
  selector:
    matchLabels:
      app: website
  template:
    metadata:
      labels:
        app: website
    spec:
      containers:
      - name: website
        image: chonyy/calculator_website:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: website
spec:
  ports:
  - port: 3000
  selector:
    app: website
```

Run the frontend and backend in the namespace we created using kubectl apply

```kubectl apply -f calculator.yaml --namespace ingress-basic```

### Create an Ingress Route

Both the frontend and backend are now running on the Kubernetes Cluster. Now we create an ingress resource to configure the rules that route traffic to our website or API. This resource can be created with a similar method by vim `ingress.yaml`

```yml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: calculator-ingress
  namespace: ingress-basic
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /?(.*)
        backend:
          serviceName: website
          servicePort: 3000
      - path: /api/?(.*)   
        backend:
          serviceName: api
          servicePort: 80
```
The most important part of this file is from line 15 ~ 22. We route the traffic from the root URL to our website service. Similarly, we route the traffic from `/api` to our API service.

### Create the ingress resource
`kubectl apply -f ingress.yaml`

### Try it!

The Kubernetes load balancer service is created for the NGINX ingress controller, we can access our app with the assigned dynamic public IP address.

`kubectl --namespace ingress-basic get services -o wide -w nginx-ingress-ingress-nginx-controller`


## Helm Chart example

```console
git clone https://github.com/Azure/dev-spaces
cd dev-spaces/samples/nodejs/getting-started/webfrontend
```

### Create a Dockerfile

Create a new *Dockerfile* file using the following:

```dockerfile
FROM node:latest

WORKDIR /webfrontend

COPY package.json ./

RUN npm install

COPY . .

EXPOSE 80
CMD ["node","server.js"]
```

### Build and push the sample application to the ACR

Use the [az acr build][az-acr-build] command to build and push an image to the registry, using the preceding Dockerfile. The `.` at the end of the command sets the location of the Dockerfile, in this case the current directory.

```azurecli
az acr build --image webfrontend:v1 \
  --registry azure-workshop \
  --file Dockerfile .
```

### Create your Helm chart

Generate your Helm chart using the `helm create` command.

```console
helm create webfrontend
```

Make the following updates to *webfrontend/values.yaml*. Substitute the loginServer of your registry that you noted in an earlier step, such as *myhelmacr.azurecr.io*:

* Change `image.repository` to `<loginServer>/webfrontend`
* Change `service.type` to `LoadBalancer`

For example:

```yml
# Default values for webfrontend.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: myhelmacr.azurecr.io/webfrontend
  pullPolicy: IfNotPresent
...
service:
  type: LoadBalancer
  port: 80
...
```

Update `appVersion` to `v1` in *webfrontend/Chart.yaml*. For example

```yml
apiVersion: v2
name: webfrontend
...
# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application.
appVersion: v1
```

### Run your Helm chart

Use the `helm install` command to install your application using your Helm chart.

```console
helm install webfrontend webfrontend/
```

It takes a few minutes for the service to return a public IP address. To monitor the progress, use the `kubectl get service` command with the *watch* parameter:

```console
$ kubectl get service --watch

NAME                TYPE          CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
webfrontend         LoadBalancer  10.0.141.72   <pending>     80:32150/TCP   2m
...
webfrontend         LoadBalancer  10.0.141.72   <EXTERNAL-IP> 80:32150/TCP   7m
```

Navigate to the load balancer of your application in a browser using the `<EXTERNAL-IP>` to see the sample application.

### Delete the cluster

When the cluster is no longer needed, use the [az group delete][az-group-delete] command to remove the resource group, the AKS cluster, the container registry, the container images stored there, and all related resources.

```azurecli-interactive
az group delete --name MyResourceGroup --yes --no-wait
```
