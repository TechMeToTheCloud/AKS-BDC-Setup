# AKS-BDC-Setup

## AKS Deployment 

[This is the source](https://docs.microsoft.com/en-us/sql/big-data-cluster/active-directory-deployment-aks-tutorial?view=sql-server-ver15) of the following code.  
* we did not use an AKS private cluster
* we are not using AD auth mode

From shell.azure.com:

```bash
# change vars as needed
# this block needs to be rerun if cloudshell times out
# run `watch ls` in cloudshell to avoid the timeout and ctl+c to abort it
export SUBSCRIPTION=airs
export RG=BDC_POC
export LOCATION=eastus
export VNET=aks_net
export ADDR_PREFIX="10.18.0.0/16"
export SUBNET=bdc_subnet1
export SUB_PREFIX="10.18.1.0/24"
export AKS_NAME=BDC-AKS
export SP_NAME="${LOCATION}_${RG}_${AKS_NAME}"
az account set --subscription $SUBSCRIPTION

# rg creation
az group create --name $RG --location $LOCATION

# service principal
result=$(az ad sp create-for-rbac --skip-assignment --name http://${SP_NAME})
SP_PRINCIPAL=$(echo $result | jq -r '.appId')
SP_PWD=$(echo $result | jq -r '.password')

# networking
az network vnet create \
  --name $VNET \
  --resource-group $RG \
  --address-prefix $ADDR_PREFIX \
  --subnet-name $SUBNET \
  --subnet-prefix $SUB_PREFIX

SUBNET_ID=$(az network vnet subnet show \
 --resource-group $RG \
 --vnet-name $VNET \
 --name $SUBNET \
 --query id -o tsv)

# AKS
az aks create \
    --resource-group $RG \
    --name $AKS_NAME \
    --load-balancer-sku standard \
    --network-plugin azure \
    --vnet-subnet-id $SUBNET_ID \
    --dns-service-ip 10.18.2.10 \
    --service-cidr 10.18.2.0/24 \
    --node-vm-size Standard_D13_v2 \
    --node-count 8 \
    --generate-ssh-keys \
    --service-principal $SP_PRINCIPAL \
    --client-secret $SP_PWD

# test connectivity
# this can also be run locally later to add the context to .kube/config file
az aks get-credentials \
  --resource-group $RG \
  --name $AKS_NAME \
  --overwrite-existing \
  --admin

# verify we are connected to the correct AKS
kubectl config current-context
kubectl get nodes


# to delete the Az resources
# az group delete -g $RG
```

## SQL BDC Deployment

We are now ready to install BDC using `azdata` but azdata is not available in cloudshell.  

### Prerequisites on local Windows Laptop

* [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15)
  * or run `winget install Microsoft.AzureDataStudio` from an ELEVATED prompt
* open ADS and install the extension `microsoft.datavirtualization` ([Data Virtualization extension for Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/extensions/data-virtualization-extension?view=sql-server-ver15))
* [azdata cli tool download](https://aka.ms/azdata-msi)
* [kubectl command line tool](https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-kubectl/)


###  Deployment Options

1. the Wizard in ADS
    * Help|Welcome and then `Deploy a Server`
2. `azdata`
    * this is the method we will use below.  


### Deployment from local windows machine

From PoSh:

```powershell
$RG = "BDC_POC"
$AKS_NAME = "BDC-AKS"
$BDC_NAME = "bdc-aks"
$SUBSCRIPTION = "airs"

az login 
az account set --subscription $SUBSCRIPTION

az aks get-credentials --resource-group $RG --name $AKS_NAME --overwrite-existing --admin
kubectl config current-context
kubectl get nodes
# we can now connect to AKS from our local machine

# run this if you want to build a new set of configuration files
# I've done this already and saved the files to ./custom folder in the repo
azdata bdc config init --source aks-dev-test --target custom --force

# cd to wherever you cloned this repo
# this will change the bdc name from whatever you generated above
azdata bdc config replace -p custom\bdc.json -j metadata.name=$BDC_NAME
# create the cluster
azdata bdc create -c custom --accept-eula yes
# admin
# Password01!!

kubectl get pods -n $BDC_NAME
azdata login -n $BDC_NAME

# list the endpoints
azdata bdc endpoint list -o table

# get external IP
kubectl get svc controller-svc-external -n $BDC_NAME
# alternative login
azdata login --endpoint https://IP:30080 --username admin

# healthcheck
azdata bdc status show
azdata bdc sql status show
```



## Setup 

You can connect to the SQL master instance front-end from ADS by using this syntax: 

`20.81.21.223,31433`

you can find this Endpoint using `azdata bdc endpoint list -o table`

## Next Steps

* [Prep the cluster for data virtualization](virtualization.ipynb):  Open this in ADS and follow along.


# Step 4 Setup Virtualization
 There are multiple ways to setup the virtualization:
 * From Azure Data Studio
 <br> After creating a database you can right click and then choose the "Virtualize Data" option then follow the wizard
 <br>![image](https://user-images.githubusercontent.com/49620357/122288734-efab7a00-cebf-11eb-899d-1601f8d3a0af.png)
 
 * Use a SQL script to
   * Create a Master Key
   * Create a Database Scoped Credential
   * Create an external source (SQL server, Oracle, Terradata, etc)
   * Create an external table
 <br>![image](https://user-images.githubusercontent.com/49620357/122289904-38affe00-cec1-11eb-959b-a330f7db3ab1.png)
 
 # Step 5 Mount a S3 storage
  * https://docs.microsoft.com/en-us/sql/big-data-cluster/hdfs-tiering-mount-s3?view=sql-server-ver15
 
 # Step 6 Mount an ADLS storage
  * https://docs.microsoft.com/en-us/sql/big-data-cluster/hdfs-tiering-mount-adlsgen2?view=sql-server-ver15
 
 
# Resources
* https://docs.microsoft.com/en-us/sql/big-data-cluster/deployment-guidance?view=sql-server-ver15
* https://docs.microsoft.com/en-us/sql/big-data-cluster/tutorial-load-sample-data?view=sql-server-ver15
* https://docs.microsoft.com/en-us/sql/azdata/reference/reference-azdata-bdc?view=sql-server-ver15
 
