# Introduction 
Labs for AKS training.

## Labs Overview

| Lab  | Description |
| ---- | ---- |
| Lab 1 - Containerize a web application  | Package the web app into a Docker container and run it locally.   |
| Lab 2 - Publish a container to a registry  | Deploy an Azure Container Registry and publish the container.  |
| Lab 3 - Create an AKS cluster  | Install the Kubernetes CLI tool, deploy an AKS cluster in Azure, and verify it is running.  |
| Lab 4 - Deploy a web application to an AKS cluster  | Pods, Services, Deployments: Deploy the web app to the AKS cluster with configuring Liveness and Readiness Probes.  |
| Lab 5 - Scale a web application  | Flex Kubernetes' muscles by scaling pods, and then nodes. Observe how Kubernetes responds to resource limits.  |
| Lab 6 - Configure an application with a NGINX Ingress Controller  | Explore integrating DNS with Kubernetes services and explore different ways of routing traffic to the web app by configuring an Ingress Controller.  |
| Lab 7 - Monitor an AKS Cluster and applications  | Explore the logs provided by Kubernetes using the Kubernetes CLI, configure Azure Monitor and build a dashboard that monitors the AKS cluster.  |
| Lab 8 - Manage secrets with Azure Keyvault  | Manage the secrets with Azure Key Vault Provider for Secrets Store CSI Driver. |
| Lab 9 - Store persistent data with Persistent Volume Claims  | Create Azure data disks and using the Kubernetes storage mechanism to make them available to be attached to running containers.  |

## Prerequisites

### Tooling for the labs

* [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)
* [Docker Desktop on Windows](https://docs.docker.com/desktop/install/windows-install/)
* [Visual Studio Code](https://code.visualstudio.com/download)
* [Kubectl CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
* [Helm v3 CLI](https://helm.sh/docs/intro/install/)
* [Git CLI](https://git-scm.com/downloads)

### Variables

There are variables that you have to replace with your own values.

| Variable  | Description |
| --- | --- |
| $AZ_TENANT_ID  | The Azure Tenant Id. |
| $AZ_SUBSCRIPTION_ID | The Azure Subscription Id. |
| $AZ_RESOURCE_GROUP  | The Resource Group where the AKS cluster and the Azure Container Registry will be deployed. |
| $AZ_CONTAINER_REGISTRY | The Azure Container Registry Name. |
| $AZ_KUBERNETES_CLUSTER | The AKS Cluster name.  |
| $NEW_DNS_LABEL | The DNS Label name to set on the ingress controller external ip address. |
| $AZ_KEYVAULT_NAME | The AKS Cluster name.  |

Open a Windows PowerShell session, and set the following variables, with replacing <numbering\> by your number.

````Powershell
$AZ_TENANT_ID = "2bf80a97-feeb-4ffa-be69-14c56f49312d"
$AZ_SUBSCRIPTION_ID = "16cfc856-c2f1-4103-80c3-8deba1743963"
$AZ_RESOURCE_GROUP = "rg_trainee_<numbering>"
$AZ_CONTAINER_REGISTRY = "acrakstrainee<numbering>"
$AZ_KUBERNETES_CLUSTER = "akstrainee<numbering>"
$NEW_DNS_LABEL = "akstrainee<numbering>"
$AZ_KEYVAULT_NAME = "kvakstrainee<numbering>"
````

# Lab 1 - Containerize a web application

* Clone the Java Application: Flight Booking System for Airline Reservations

````Powershell
git clone https://github.com/Azure-Samples/containerize-and-deploy-Java-app-to-Azure.git
````
* cd to the Airlines web application project folder

````Powershell
cd containerize-and-deploy-Java-app-to-Azure/Project/Airlines
````

* Create a file called Dockerfile, then add the following DockerFile contents
````Powershell
New-Item -Path Dockerfile
````

````Dockerfile
#
# Build stage
#
FROM maven:3.6.0-jdk-11-slim AS build
WORKDIR /build
COPY pom.xml .
COPY src ./src
COPY web ./web
RUN mvn clean package

#
# Package stage
#
FROM tomcat:8.5.72-jre11-openjdk-slim
COPY tomcat-users.xml /usr/local/tomcat/conf
COPY --from=build /build/target/*.war /usr/local/tomcat/webapps/FlightBookingSystemSample.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
````
* Build the Docker image
````Powershell
docker build -t flightbookingsystemsample .
````
* Run the Docker image
````Powershell
docker run -d -p 8080:8080 flightbookingsystemsample
````

* Verify that you can access to the application at http://localhost:8080/FlightBookingSystemSample 
* Open Docker Desktop to see the container running, then click on it to see the logs

# Lab 2 - Publish a container to a registry

* Login to Azure
````Powershell
az login --use-device-code -t $AZ_TENANT_ID
````
````Powershell
az account set --subscription $AZ_SUBSCRIPTION_ID
````
* Create a new ACR

````Powershell
az acr create --resource-group $AZ_RESOURCE_GROUP --name $AZ_CONTAINER_REGISTRY --sku Basic
````
* Login to the ACR
````Powershell
az acr login --name $AZ_CONTAINER_REGISTRY
````
* Tag the Docker image with the ACR name
````Powershell
docker tag flightbookingsystemsample "$AZ_CONTAINER_REGISTRY.azurecr.io/flightbookingsystemsample"
````
* Push the Docker image to the ACR
````Powershell
docker push "$AZ_CONTAINER_REGISTRY.azurecr.io/flightbookingsystemsample"
````
* Verify via CLI and from Azure Portal you can see the image in the ACR
````Powershell
az acr repository show -n $AZ_CONTAINER_REGISTRY --image flightbookingsystemsample:latest
````
# Lab 3 - Create an AKS cluster

* Install the Kubernetes CLI

````Powershell
az aks install-cli
````
* Create the AKS Cluster

````Powershell
az aks create -g $AZ_RESOURCE_GROUP -n $AZ_KUBERNETES_CLUSTER `
--enable-managed-identity `
--node-count 1 `
--generate-ssh-keys
````
* Connect to cluster using the az aks get-credentials command

````Powershell
az aks get-credentials -g $AZ_RESOURCE_GROUP -n $AZ_KUBERNETES_CLUSTER 
````
* To verify the connection to your cluster, run the kubectl get nodes command to return a list of the cluster nodes

````Powershell
kubectl get nodes
````
* Attach the ACR to the AKS

Option #1
````Powershell
az aks update `
-n $AZ_KUBERNETES_CLUSTER `
-g $AZ_RESOURCE_GROUP `
--attach-acr $AZ_CONTAINER_REGISTRY
````
Option #2 (If you are not owner on the subscription level)

````Powershell
$acrResourceID=$(az acr show --resource-group $AZ_RESOURCE_GROUP --name $AZ_CONTAINER_REGISTRY --query id --output tsv)
$spID=$(az aks show -g $AZ_RESOURCE_GROUP -n $AZ_KUBERNETES_CLUSTER --query identityProfile.kubeletidentity.clientId -o tsv)
az role assignment create --assignee $spID --scope $acrResourceID --role acrpull
````

# Lab 4 - Deploy a web application to an AKS cluster

* Create a file called deployment.yaml, then add the following YAML contents, with replacing the <AZ_CONTAINER_REGISTRY> by the ACR name

````Powershell
New-Item -Path deployment.yaml
````

````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemsample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flightbookingsystemsample
  template:
    metadata:
      labels:
        app: flightbookingsystemsample
    spec:
      containers:
      - name: flightbookingsystemsample
        image: <AZ_CONTAINER_REGISTRY>.azurecr.io/flightbookingsystemsample:latest
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            port: 8080
            path: /FlightBookingSystemSample
          initialDelaySeconds: 10
          periodSeconds: 5          
---
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemsample
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: flightbookingsystemsample
````

* Deploy the application into the AKS Cluster
````Powershell
kubectl apply -f ./deployment.yaml
````
* You can now use kubectl to monitor the status of the deployment
````Powershell
kubectl get all
````
If your POD status is Running then the app should be accessible.  

* You can now use the EXTERNAL-IP from the following command line output to access the running app within Azure Kubernetes
````Powershell
kubectl get services flightbookingsystemsample
````
* Open up a browser and visit the Flight Booking System Sample landing page at http://EXTERNALIP:8080/FlightBookingSystemSample 

# Lab 5 - Scale a web application

* Increase the replicas count property of the deployment in the existing ./deployment.yaml file from 1 to 3:
````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemsample
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flightbookingsystemsample
  template:
    metadata:
      labels:
        app: flightbookingsystemsample
    spec:
      containers:
      - name: flightbookingsystemsample
        image: <AZ_CONTAINER_REGISTRY>.azurecr.io/flightbookingsystemsample:latest
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            port: 8080
            path: /FlightBookingSystemSample
          initialDelaySeconds: 10
          periodSeconds: 5          
---
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemsample
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
  selector:
    app: flightbookingsystemsample
````
* Then, deploy the configuration
````Powershell
kubectl apply -f ./deployment.yaml
````
Note: The scaling was also possible by running this command line:
````Powershell
kubectl scale flightbookingsystemsample --replicas=3
````
* Verify the deployment status
````Powershell
kubectl describe deployment flightbookingsystemsample
````
````Powershell
kubectl get pods
````
* Verify the pod description of those with “Pending” status
````Powershell
kubectl describe pod <flightbookingsystemsample-pod-name>
````
What's happen? The pods can’t be created because the cpu requested total exceed the cpu available! So, you can scale the AKS cluster nodes to increase cpu.  

* Scale the AKS cluster nodes from 1 to 3

````Powershell
az aks scale `
--resource-group $AZ_RESOURCE_GROUP `
--name $AZ_KUBERNETES_CLUSTER `
--node-count 3 `
--nodepool-name nodepool1
````

* Verify that all the pods have been successfully provisioned this time
````Powershell
kubectl get all
````

# Lab 6 – Configure an application with a NGINX Ingress Controller

We will install the NGINX ingress controller into the AKS cluster with Helm.

* Install the NGINX ingress controller

````Powershell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx `
ingress-nginx/ingress-nginx `
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz `
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="$NEW_DNS_LABEL"
````

* Verify the load balancer service status
````Powershell
kubectl get services -o wide -w ingress-nginx-controller
````

When the Kubernetes load balancer service is created for the NGINX ingress controller, an IP address is assigned under EXTERNAL-IP and the DNS label is applied on it.

For testing the traffic routes of the Ingress controller, we will deploy a second sample service, the flight booking system (REST) APIs service endpoint.

* Create a file called flightbookingsystem-apis-deployment.yaml, then add the following YAML contents 

````Powershell
New-Item -Path flightbookingsystem-apis-deployment.yaml
````

````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemapis  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flightbookingsystemapis
  template:
    metadata:
      labels:
        app: flightbookingsystemapis
    spec:
      containers:
      - name: flightbookingsystemapis
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "Flight Booking System APIs"
---
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemapis  
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: flightbookingsystemapis
````

* Then, deploy the configuration
````Powershell
kubectl apply -f ./flightbookingsystem-apis-deployment.yaml
````

Configure the existing web application flightbookingsystemsample. 
* In the Service resource, update the type from LoadBalancer to ClusterIP and the port from 8080 to 80  in the existing ./deployment.yaml file

````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemsample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flightbookingsystemsample
  template:
    metadata:
      labels:
        app: flightbookingsystemsample
    spec:
      containers:
      - name: flightbookingsystemsample
        image: <AZ_CONTAINER_REGISTRY>.azurecr.io/flightbookingsystemsample:latest
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            port: 8080
            path: /FlightBookingSystemSample
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemsample
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: flightbookingsystemsample
````

* Then, deploy the configuration
````Powershell
kubectl apply -f ./deployment.yaml
````
To route traffic to the applications, we will create a Kubernetes ingress resource configured with a DNS Label.

* Create a file called flightbookingsystemsample-ingress-routes.yaml, add the following YAML contents, with replacing NEW-DNS-LABEL with the DNS label name

````Powershell
New-Item -Path flightbookingsystemsample-ingress-routes.yaml
````

````Yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: flightbookingsystemsample-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: <NEW-DNS-LABEL>.westeurope.cloudapp.azure.com
    http:
      paths:
      - path: /website(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: flightbookingsystemsample
            port:
              number: 80
      - path: /_apis(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: flightbookingsystemapis
            port:
              number: 80
````

* Deploy the configuration

````Powershell
kubectl apply -f ./flightbookingsystemsample-ingress-routes.yaml
````

To test the ingress controller, browse to the applications:   
* http://\<NEW-DNS-LABEL>.westeurope.cloudapp.azure.com/website/FlightBookingSystemSample/
* http://\<NEW-DNS-LABEL>.westeurope.cloudapp.azure.com/_apis/

# Lab 7 - Monitor an AKS Cluster and applications

We will be enabling "Azure Monitor for Containers" on the AKS cluster

* Enable the monitoring

````Powershell
az aks enable-addons -a monitoring `
-n $AZ_KUBERNETES_CLUSTER `
-g $AZ_RESOURCE_GROUP `
--enable-msi-auth-for-monitoring
````

Verify that you can see the Monitoring live insights under the Monitoring -> Insights section of the AKS cluster through the Azure Portal

* Navigate to the Insights, Metrics, Logs and explore Alerts under the section Monitoring of the AKS cluster through the Azure Portal

* Check for the logs of a pod with:
````Powershell
kubectl logs <pod-name>
````
* Pod failures can be investigated during troubleshooting with:
````Powershell
kubectl describe pod <pod-name>
````
* A bash shell can be opened on any pod so you can poke around on the filesystem to debug issues. You can open the shell with:
````Powershell
kubectl exec -it <pod-name> -- /bin/bash
````


# Lab 8 - Manage secrets with Azure Keyvault
We will be enabling "Azure Key Vault Provider for Secrets Store CSI Driver" on the AKS cluster.  

* Enable the Azure Key Vault Provider 

````Powershell
az aks enable-addons `
-n $AZ_KUBERNETES_CLUSTER `
-g $AZ_RESOURCE_GROUP `
--addons azure-keyvault-secrets-provider
````

* Verify the Azure Key Vault Provider for Secrets Store CSI Driver is installed successfully

````Powershell
kubectl get pods -n kube-system
````

Create a Keyvault and a secret

* Create a Keyvault
````Powershell
az keyvault create -n $AZ_KEYVAULT_NAME -g $AZ_RESOURCE_GROUP -l westeurope
````
* Create a secret

````Powershell
az keyvault secret set --vault-name $AZ_KEYVAULT_NAME -n ExampleSecret --value MyAKSExampleSecret
````

Grant the permissions for the AKS Cluster to access to the Keyvault.  

* Get the AKS cluster identity client Id
````Powershell
$identityClientId=(az aks show `
-g $AZ_RESOURCE_GROUP `
-n $AZ_KUBERNETES_CLUSTER `
--query addonProfiles.azureKeyvaultSecretsProvider.identity.clientId -o tsv)
````

* Set policy to access secrets in your key vault for the AKS cluster identity client Id
````Powershell
az keyvault set-policy -n $AZ_KEYVAULT_NAME --secret-permissions get list --spn $identityClientId
````

Create a SecretProviderClass.

* Create a file called secretproviderclass.yaml, then add the following YAML contents, using your own values for:
  * userAssignedIdentityID  
  * keyvaultName  
  * tenantId  

````Powershell
New-Item -Path secretproviderclass.yaml

Write-Host "client-id=$identityClientId"
Write-Host "key-vault-name=$AZ_KEYVAULT_NAME"
Write-Host "tenant-id=$AZ_TENANT_ID"
````

````Yaml
# This is a SecretProviderClass example using user-assigned identity to access your key vault
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-user-msi
spec:
  provider: azure
  secretObjects:                              # [OPTIONAL] SecretObjects defines the desired state of synced Kubernetes secret objects
  - data:
    - key: secretexample                           # data field to populate
      objectName: ExampleSecret                        # name of the mounted content to sync; this could be the object name or the object alias
    secretName: kubernetessecret                     # name of the Kubernetes secret object
    type: Opaque 
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"          # Set to true for using managed identity
    userAssignedIdentityID: "<client-id>"   # Set the clientID of the user-assigned managed identity to use
    keyvaultName: "<key-vault-name>"        # Set to the name of your key vault
    cloudName: ""                         # [OPTIONAL for Azure] if not provided, the Azure environment defaults to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: ExampleSecret
          objectType: secret              # object types: secret, key, or cert
          objectVersion: ""               # [OPTIONAL] object versions, default to latest if empty
    tenantId: "<tenant-id>"                 # The tenant ID of the key vault
````

* Apply the SecretProviderClass to your cluster
````Powershell
kubectl apply -f secretproviderclass.yaml
````
Configure the deployment with including the secrets.

* Create a file called deployment-with-secrets.yaml, then add the following YAML contents, with replacing the <AZ_CONTAINER_REGISTRY> with the ACR name

````Powershell
New-Item -Path deployment-with-secrets.yaml
````

````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemsample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flightbookingsystemsample
  template:
    metadata:
      labels:
        app: flightbookingsystemsample
    spec:
      containers:
      - name: flightbookingsystemsample
        image: <AZ_CONTAINER_REGISTRY>.azurecr.io/flightbookingsystemsample:latest
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            port: 8080
            path: /FlightBookingSystemSample
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: secrets-store01-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
        env:
        - name: SECRET_EXAMPLE
          valueFrom:
            secretKeyRef:
              name: kubernetessecret
              key: secretexample
      volumes:
        - name: secrets-store01-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "azure-kvname-user-msi"
        
---     
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemsample
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: flightbookingsystemsample
````

* Apply the deployment to your cluster
````Powershell
kubectl apply -f deployment-with-secrets.yaml
````
Verify that you can access to the secrets within the pod

* Retrieve the flightbookingsystemsample pod name
````Powershell
kubectl get pods
````
* Show secrets held in secrets-store
````Powershell 
kubectl exec <flightbookingsystemsample-pod-name> -- ls /mnt/secrets-store/
````
* Print a test secret 'ExampleSecret' held in secrets-store
````Powershell 
kubectl exec <flightbookingsystemsample-pod-name> -- cat /mnt/secrets-store/ExampleSecret
````
* Show the secret as an Environment Variable
````Powershell
kubectl exec -it <flightbookingsystemsample-pod-name> -- printenv
````
# Lab 9 - Store persistent data with Persistent Volume Claims

We will be dynamically creating and using a persistent volume with Azure Disks. 

* Create a file called azure-pvc.yaml, then add the following YAML contents 

````Powershell
New-Item -Path azure-pvc.yaml
````

````Yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-managed-disk
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 2Gi
````

* Deploy the configuration
````Powershell
kubectl apply -f azure-pvc.yaml
````

* Verify in the AKS cluster, through the Azure Portal, that you can see in the Storage section, the Persistent Volume Claim has been created successfully.
````Powershell
kubectl get pvc azure-managed-disk
````

Before to add a persistent volume, we will see what’s happen when we add a new file directly within a pod, then we delete this pod to trigger a new pod creation. We will verify then if the file is still here. 

* Retrieve the flightbookingsystemsample pod name
````Powershell
kubectl get pods
````

* Run a shell within the pod to add a new file data.csv into a /storage directory to simulate data to persist
````Powershell
kubectl exec -it <flightbookingsystemsample-pod-name> -- /bin/bash -c "cd / && mkdir storage && cd storage && touch data.csv"
````
* Check that the file is created correctly
````Powershell
kubectl exec -it <flightbookingsystemsample-pod-name> -- dir /storage
````
* Delete the pod instance
````Powershell
kubectl delete pod <flightbookingsystemsample-pod-name>
````
* Wait that the pod will be re-created, then verify if the file /storage/data.csv is still here. 

````Powershell
kubectl get pods
````

````Powershell
kubectl exec -it <flightbookingsystemsample-new-pod-name> -- dir /storage
````

We can see that the storage folder and the data.csv file are no longer present.

Now, we will add a persistent volume and run the same test again.

* Create a file called deployment-with-pvc-disk.yaml, then add the following YAML contents, with replacing the <AZ_CONTAINER_REGISTRY> with the ACR name

````Powershell
New-Item -Path deployment-with-pvc-disk.yaml
````

````Yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flightbookingsystemsample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flightbookingsystemsample
  template:
    metadata:
      labels:
        app: flightbookingsystemsample
    spec:
      containers:
      - name: flightbookingsystemsample
        image: <AZ_CONTAINER_REGISTRY>.azurecr.io/flightbookingsystemsample:latest
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
          limits:
            cpu: "1"
            memory: "2Gi"
        ports:
        - containerPort: 8080
        readinessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            port: 8080
            path: /FlightBookingSystemSample
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - mountPath: "/storage"
          name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: azure-managed-disk
---
apiVersion: v1
kind: Service
metadata:
  name: flightbookingsystemsample
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: flightbookingsystemsample
````

* Deploy the configuration
````Powershell
kubectl apply -f deployment-with-pvc-disk.yaml
````

* Verify in the AKS cluster, through the Azure Portal, that you can see in the Storage section that the volume for the container has been created successfully


Verify that the persistence is working correctly by adding a data.csv file directly within the pod again. 

* Run a shell within the pod to add a new file data.csv into the /storage directory to simulate data to persist
````Powershell
kubectl exec -it <flightbookingsystemsample-pod-name> -- touch /storage/data.csv
````

* Delete the pod instance
````Powershell
kubectl delete pod <flightbookingsystemsample-pod-name>
````
* Wait that the pod will be re-created, then verify that the file /storage/data.csv is still here.

````Powershell
kubectl get pods
````

````Powershell
kubectl exec -it <flightbookingsystemsample-pod-name> -- dir /storage
````

## Useful links

* https://github.com/spring-guides/gs-spring-boot-docker#build-a-docker-image-with-maven 
* https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli
* https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli
* https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
* https://learn.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli
* https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication-managed-identity?tabs=azure-cli#grant-identity-access-to-the-container-registry
* https://learn.microsoft.com/en-us/azure/aks/scale-cluster?tabs=azure-cli
* https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli
* https://learn.microsoft.com/en-us/azure/aks/concepts-storage
* https://learn.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv?source=recommendations
* https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-overview
* https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard
* https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver
