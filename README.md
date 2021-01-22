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

## Run the Application

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

## Create an Ingress Route

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

## Create the ingress resource
`kubectl apply -f ingress.yaml`

## Try it!

The Kubernetes load balancer service is created for the NGINX ingress controller, we can access our app with the assigned dynamic public IP address.

`kubectl --namespace ingress-basic get services -o wide -w nginx-ingress-ingress-nginx-controller`
