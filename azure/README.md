# Deploying IBM Operational Decision Manager on Azure AKS

This project demonstrates how to deploy an IBM® Operational Decision Manager (ODM) clustered topology on the Azure Kubernetes Service (AKS)  (AKS) cloud service. This deployment implements Kubernetes and Docker technologies. 
Homepage of Azure : https://portal.azure.com/?feature.quickstart=true#home

<img width="800" height="560" src='./images/aks-schema.jpg'/>

The ODM Docker material is available in Passport Advantage. It includes Docker container images and Helm chart descriptors. 

## Included components
The project comes with the following components:
- [IBM Operational Decision Manager](https://www.ibm.com/support/knowledgecenter/en/SSQP76_8.10.x/com.ibm.odm.kube/kc_welcome_odm_kube.html)
- [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/)
- [Azure Database for PostgreSQL](https://docs.microsoft.com/en-us/azure/postgresql/)
- [AKS Networking](https://docs.microsoft.com/en-us/azure/aks/concepts-network)

## Tested environment
The commands and tools have been tested on MacOS.

## Prerequisites
First, install the following software on your machine:

* [Install Azure cli Az CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) 
* [Helm](https://github.com/helm/helm/releases)


Then, create an  [Create an Azure account and pay as you go](https://azure.microsoft.com/en-us/pricing/purchase-options/pay-as-you-go/)

## Steps to deploy ODM on Kubernetes from Azure AKS

 * [1. Prepare your AKS instance](#1-prepare-your-aks-instance)
     * [a. Login to Azure](#a-login-to-azure)
     * [b. Create a resource group](#b-create-a-resource-group)
     * [c. Create an AKS cluster (30 min)](#c-create-an-aks-cluster-30-min)
     * [d. Set up your environment to this cluster](#d-set-up-your-environment-to-this-cluster)
  * [2. Create the Postgreql Azure instance](#2-create-the-postgreql-azure-instance)
     * [Create an Azure Database for PostgreSQL](#create-an-azure-database-for-postgresql)
     * [Create a firewall rule that allows access from Azure services](#create-a-firewall-rule-that-allows-access-from-azure-services)
  * [Preparing your environment for ODM installation](#preparing-your-environment-for-odm-installation)
     * [Create a pull secret to pull images from the IBM Entitled Registry that contains ODM Docker images](#create-a-pull-secret-to-pull-images-from-the-ibm-entitled-registry-that-contains-odm-docker-images)
     * [Create the datasource Secrets for Azure Postgresql](#create-the-datasource-secrets-for-azure-postgresql)
     * [5. Install an IBM Operational Decision Manager release (10 min)](#5-install-an-ibm-operational-decision-manager-release-10-min)
        * [a. Prerequisites](#a-prerequisites)
        * [b. Install an ODM Helm release](#b-install-an-odm-helm-release)
        * [c. Check the topology](#c-check-the-topology)
     * [6. Access the ODM services  ](#6-access-the-odm-services)
        * [a. Create an Application Load Balancer](#a-create-an-application-load-balancer)
  

## 1. Prepare your AKS instance

Source from : https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

### a. Login to Azure

After installing the Azure cli use this command line.
   ```console
   az login 
   ```

It will open a web browser where you can connect using your Azure credentials

### b. Create a resource group
 An Azure resource group is a logical group in which Azure resources are deployed and managed. When you create a resource group, you are asked to specify a location. This location is where resource group metadata is stored, it is also where your resources run in Azure if you don't specify another region during resource creation. Create a resource group using the az group create command.
   ```console
az group create --name odm-group --location francecentral
   ```

The following example output shows the resource group created successfully:
   ```json
    {
      "id": "/subscriptions/<guid>/resourceGroups/odm-group",
      "location": "eastus",
      "managedBy": null,
      "name": "odm-group",
      "properties": {
        "provisioningState": "Succeeded"
      },
      "tags": null
    }
   ```
   

### c. Create an AKS cluster (30 min)
Use the az aks create command to create an AKS cluster. The following example creates a cluster named myAKSCluster with one node. Azure Monitor for containers is also enabled using the --enable-addons monitoring parameter.  This will take several minutes to complete.
 Note
When creating an AKS cluster a second resource group is automatically created to store the AKS resources. For more information see Why are two resource groups created with AKS?
   ```console
az aks create --resource-group odm-group --name odm-cluster --node-count 2 \
                  --location francecentral --enable-addons monitoring --generate-ssh-keys
   ```

After a few minutes, the command completes and returns JSON-formatted information about the cluster.

> NOTE: By default this will create a Kubernetes version 1.16 or higher.
       
### d. Set up your environment to this cluster
manage a Kubernetes cluster, you use kubectl, the Kubernetes command-line client. If you use Azure Cloud Shell, kubectl is already installed. To install kubectl locally, use the az aks install-cli command:

```console
az aks install-cli
```

To configure kubectl to connect to your Kubernetes cluster, use the az aks get-credentials command. This command downloads credentials and configures the Kubernetes CLI to use them.

```console
az aks get-credentials --resource-group odm-group --name odm-cluster
```

To verify the connection to your cluster, use the kubectl get command to return a list of the cluster nodes.

```console
kubectl get nodes
```

The following example output shows the single node created in the previous steps. Make sure that the status of the node is Ready:
Output
NAME                       STATUS   ROLES   AGE     VERSION
aks-nodepool1-31718369-0   Ready    agent   6m44s   v1.12.8


To further debug and diagnose cluster problems, run the command:

```console
kubectl cluster-info dump
```

## 2. Create the Postgreql Azure instance

### Create an Azure Database for PostgreSQL

Create an Azure Database for PostgreSQL server using the az postgres server create command. A server can contain multiple databases.

```console
 az postgres server create --resource-group odm-group --name odmpsqlserver \
                           --admin-user myadmin --admin-password 'passw0rd!' \
                           --sku-name GP_Gen5_2 --version 9.6 --location francecentral
```

Verify the database
To connect to your server, you need to provide host information and access credentials.

```console
 az postgres server show --resource-group odm-group --name odmpsqlserver
```

Result:
   ```javascript
   {
        "administratorLogin": "myadmin",
        "byokEnforcement": "Disabled",
        "earliestRestoreDate": "2020-07-13T06:59:05.050000+00:00",
        "fullyQualifiedDomainName": "odmpsqlserver.postgres.database.azure.com",
        "id": "/subscriptions/18583f39-c9d2-4813-beba-1a633f94cdaa/resourceGroups/odm-group/providers/Microsoft.DBforPostgreSQL/servers/odmpsqlserver",
        "identity": null,
        "infrastructureEncryption": "Disabled",
        "location": "francecentral",
        "masterServerId": "",
        "minimalTlsVersion": "TLSEnforcementDisabled",
        "name": "odmpsqlserver",
        "privateEndpointConnections": [],
        "publicNetworkAccess": "Enabled",
        "replicaCapacity": 5,
        "replicationRole": "None",
        "resourceGroup": "odm-group",
        "sku": {
            "capacity": 2,
            "family": "Gen5",
            "name": "GP_Gen5_2",
            "size": null,
            "tier": "GeneralPurpose"
         },
        "sslEnforcement": "Enabled",
        "storageProfile": {
           "backupRetentionDays": 7,
           "geoRedundantBackup": "Disabled",
           "storageAutogrow": "Enabled",
           "storageMb": 5120
        },
        "tags": null,
        "type": "Microsoft.DBforPostgreSQL/servers",
        "userVisibleState": "Ready",
        "version": "9.6"
    }
```

###  Create a firewall rule that allows access from Azure services
To be able your database and your AKS cluster can communicate you should put in place Firewall rules with this following command:

```console
az postgres server firewall-rule create -g odm-group -s odmpsqlserver \
                       -n myrule --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

## Preparing your environment for ODM installation

### Create a pull secret to pull images from the IBM Entitled Registry that contains ODM Docker images
1. Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.

2. In the **Container software library** tile, verify your entitlement on the **View library** page, and then go to **Get entitlement key** to retrieve the key.

3. Create a pull secret by running a `kubectl create secret` command.

   ```console
   kubectl create secret docker-registry admin.registrykey --docker-server=cp.icr.io --docker-username=cp --docker-password="<API_KEY_GENERATED>" --docker-email=<USER_EMAIL>```

   > **Note**: The `cp.icr.io` value for the **docker-server** parameter is the only registry domain name that contains the images.
   
   > **Note**: Use “cp” for the docker-username. The docker-email has to be a valid email address (associated to your IBM ID). Make sure you are copying the Entitlement Key in the docker-password field within double-quotes.

4. Take a note of the secret and the server values so that you can set them to the **pullSecrets** and **repository** parameters when you run the helm install for your containers.

### Create the datasource Secrets for Azure Postgresql
Copy the files [ds-bc.xml.template](ds-bc.xml.template]) and [ds-res.xml.template](ds-res.xml.template) on your local machine and copy it to ds-bc.xml / ds-res.xml

 Replace placeholer  
- DBNAME : The db name.
- USERNAME : The db username. 
- PASSWORD : The db password
- SERVERNAME : The name of the db server name
  
Should be something like that if you have not change the value of the cmd line.

```xml
 <properties
  databaseName="postgres"
  user="myadmin@odmpsqlserver"
  password="passw0rd!"
  portNumber="5432"
  sslMode="require"
  serverName="odmpsqlserver.postgres.database.azure.com" />
```

### Create a secrets with this 2 files


```console
kubectl create secret generic customdatasource-secret --from-file datasource-ds.xml=ds-res.xml --from-file datasource-dc.xml=ds-bc.xml
```



### Create a database secret

To secure access to the database, you must create a secret that encrypts the database user and password before you install the Helm release.

```console
kubectl create secret generic <odm-db-secret> --from-literal=db-user=<rds-postgresql-user-name> --from-literal=db-password=<rds-postgresql-password> 
```


Example:
```console
kubectl create secret generic odm-db-secret --from-literal=db-user=postgres --from-literal=db-password=postgres
```
### Create a Kubernetes secret with the certificate.
TODO EXPLAIN HOW TO GENERATE CERTIFICATE.
```console
kubectl create secret generic mycompany-secret --from-file=keystore.jks=mycompany.jks \
                                               --from-file=truststore.jks=truststore.jks \
                                               --from-literal=keystore_password=password \
                                               --from-literal=truststore_password=password
```

The certificate must be the same as the one you used to enable TLS connections in your ODM release. For more information, see [Defining the security certificate](https://www.ibm.com/support/knowledgecenter/SSQP76_8.10.x/com.ibm.odm.icp/topics/tsk_replace_security_certificate.html?view=kc) and [Working with certificates and SSL](https://www.ibm.com/links?url=https%3A%2F%2Fdocs.oracle.com%2Fcd%2FE19830-01%2F819-4712%2Fablqw%2Findex.html).

## Install an ODM Helm release and expose it with the service type loadbalalncer

### Allocate public IP.
```console
 az aks update \                                                                                                                                                                          
    --resource-group odm-group \
    --name odm-cluster \
    --load-balancer-managed-outbound-ip-count 4
```

### Install the ODM Release
```console
helm install mycompany --set image.repository=cp.icr.io/cp/cp4a/odm --set image.pullSecrets=admin.registrykey \
                       --set image.arch=amd64 --set image.tag=8.10.4.0 --set service.type=LoadBalancer \
                       --set externalCustomDatabase.datasourceRef=customdatasource-secret \
                       --set customization.securitySecretRef=mycompany-secret ibm-odm-prod
```

### Check the topology
Run the following command to check the status of the pods that have been created: 
```console
kubectl get pods
```


| *NAME* | *READY* | *STATUS* | *RESTARTS* | *AGE* |
|---|---|---|---|---|
| mycompany-odm-decisioncenter-*** | 1/1 | Running | 0 | 44m |  
| mycompany-odm-decisionrunner-*** | 1/1 | Running | 0 | 44m | 
| mycompany-odm-decisionserverconsole-*** | 1/1 | Running | 0 | 44m | 
| mycompany-odm-decisionserverruntime-*** | 1/1 | Running | 0 | 44m | 

Table 1. Status of pods

### Access ODM services
By setting the **service.type=LoadBalancer** the service are exposed with public ip to access it use this command:

```console
kubectl get svc
```

| NAME | TYPE | CLUSTER-IP | EXTERNAL-IP | PORT(S) | AGE |
| --- | --- | --- | -- | --- | --- | 
| mycompany-odm-decisionserverruntime | LoadBalancer  |  10.0.118.182 |   xx.xx.xxx.xxx  |     9443:32483/TCP  | 9s |
| mycompany-odm-decisioncenter | LoadBalancer  |  10.0.118.181 |   xx.xx.xxx.xxx  |     9453:32483/TCP  | 9s |
| mycompany-odm-decisionrunner | LoadBalancer  |  10.0.166.199 |   xx.xx.xxx.xxx   |  9443:31367/TCP  | 2d17h |
| mycompany-odm-decisionserverconsole | LoadBalancer |   10.0.224.220  |  xx.xx.xxx.xxx   |      9443:30874/TCP |   9s |
| mycompany-odm-decisionserverconsole-notif | ClusterIP | 10.0.103.221 |  \<none\> | 1883/TCP |        9s |
    
  Then you can open a browser to the https://xx.xx.xxx.xxx:9443 for Decision Server console/runtime and runner and https://xx.xx.xxx.xxx:9453 for Decision center.
    

### 6. Access the ODM services via ingress

This section explains how to implement an  Application Load Balancer (ALB) to expose the ODM services to Internet connectivity.

* Create an Application Load Balancer
* Implement an ingress for ODM services

#### a. Create an Application Load Balancer
Find more information about ALB here
https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/load-balancer-getting-started.html

***TODO***


## Troubleshooting

If your microservice instances are not running properly, check the logs by running the following command:
```console
kubectl logs <your-pod-name>
```


## References
https://docs.microsoft.com/fr-fr/azure/aks/


# License
[Apache 2.0](LICENSE)