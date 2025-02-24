# Deployment Process Log
  Hi, this is a log about how I employed a MRC based question-answear syster on docker and Azure
## 1.Environment Setup

- Opreating System:
  - Ubuntu24.04 across WSL
  - Win10 professional 
- Installed Dependencies:
  - Azure, Docker, TorchServe, Venv
  - Pytorch, Transformers
  - Gunicorn, Flask

## 2.TorchServe Configuration

(1) Docker Configuration
- Dockerfile
```yaml
# set the python version to 3.8
FROM python:3.8

# set the workdir in Docker
WORKDIR /home/model-server/

# the port must same as the front-end
EXPOSE 8080

# copy documents to docker enviorment
COPY model_store/ model_store/
COPY config.properties config.properties
COPY requirements.txt requirements.txt
COPY drqa-webui-master/ drqa-webui-master/
COPY start.sh start.sh

RUN pip install --no-cache-dir -r requirements.txt
# cause can't find torch1.6 in pip
RUN pip install --no-cache-dir torch==1.6.0 -f https://download.pytorch.org/whl/torch_stable.html

# run torchserve+Gunicorn
CMD ["./start.sh"]
```
- start.sh
```shell

#!/bin/bash
echo "Starting torchserve"
torchserve --start --ts-config config.properties --model-store model_store \
--models reader=reader.mar,NER=NER.mar,W2V=W2V.mar

echo "Starting Guniocrn..."
cd drqa-webui-master
gunicorn --timeout 300 -w 4 -b 0.0.0.0:8080 index:app
```
- start Docker
```bash

docker run -d -p 5000:5000 -v /absolute/path/to/iamQA-main/iamQA-main/start.sh:/home/app/start.sh my-qa-api
docker run -d -p 8080:8080 --name my-qa-container my-qa-api
```

# 3.Azure Configuration
(1)First Time 

At your first time, to connect with Docker,you need to
create Azure Account, Resource Group, AKS, ACR, and configure Kubecel to interact with the aks cluster


- Create Azure Account
```bash 

az login --use-code-device
```
- Create Resource Group
``` bash

az group create --name <resource-group-name> --location <region>
# my code
az group create --name myResourceGroup --location australiaeast
```

- Create ACR
```bash

az acr create --resource-group <resource-group-name> --name <acr-name> --sku Basic --location <region>

#my code
az acr create --resource-group myResourceGroup --name sherryacr --sku Basic --location australiaeast
```

- Create AKS
```bash
az aks create --resource-group <resource-group-name> --name <aks-cluster-name> --node-count 1 --enable-addons monitoring --generate-ssh-keys
# my code
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --generate-ssh-keys

```
- Configure Kubectl with AKS
```bash

az aks get-credentials --resource-group <resource-group-name> --name <aks-cluster-name>

#my code
az aks get-credentials --resource-group myReourceGroup --name myAKSCluster

#verify
kubectl get nodes
```
- rember to login next time
```bash

az login --user-code-device
az acr login --name <acr-name>
az aks get-credentials --resource-group <resource-group-name> --name <aks-cluster-name>
```
(2) Measure twice, cut once   

Before we go to next setp, we need to check some right of ACR.
This experice got from the **ImagePullBack** error XDD.
- check acr health  
```bash

az acr check-health -n <acr-name> --yes
#the result of this command maybe tell you that you need download more depencies
``` 
- give acr pull right
``` bash

# get cliendId and objectId
az aks show \
	  --resource-group <your-resource-group> \
	  --name <your-aks-cluster-name> \
  	  --query "identityProfile"
  	  
# give acr pull right
az role assignment create \
	  --assignee <clientId> \
	  --role AcrPull \
	  --scope /subscriptions/<objectID>/resourcegroups/<Resource-Group-Name>/providers/Microsoft.ContainerRegistry/registries/sherryacr
  
```

# Configure docker on Azure
Finally, we can arrive our goal!
- upload docker image to ACR
``` bash

docker tag <local-image> <acr-name>.azurecr.io/<repository>/<image-name>:<tag>
docker push <acr-name>.azurecr.io/<repository>/<image-name>:<tag>
```
- create deployment.yaml and service.yaml
```yaml
#deploment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-qa-api
spec:
  replicas: 3  
  selector:
    matchLabels:
      app: mrc
  template:
    metadata:
      labels:
        app: mrc
    spec:
      containers:
        - name: my-qa-container
          image: sherryacr.azurecr.io/my-qa-api:v2
          ports:
          - containerPort: 8080
      resources:
        requests:
          memory: "256Mi"
          cpu: "500m"
        limits:
          memory: "512Mi"
          cpu: "1"
```

```yaml
#service.yaml
apiVersion: v1
kind: Service
metadata:
    name: my-qa-api-service
spec:
  selector:
    app: my-qa-api 
  ports:
    - protocol: TCP
      port: 80      
      targetPort: 8080  
  type: LoadBalancer  
```

- Run
```bash

#about AKS
kubectl apply -f deployment.yaml

#about Kubernetes
kubectl expose deployment my-app --type=LoadBalancer --port=80 --target-port=80
```

# Conlusion
- logistic between Docker, ACR, AKS, Kubernetes









  
