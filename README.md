# AKS-BDC-Setup

branches:
* `main`:  public AKS with public endpoints
* `private`:  enable private AKS instructions

## AKS Deployment 

[This is the source](https://docs.microsoft.com/en-us/sql/big-data-cluster/active-directory-deployment-aks-tutorial?view=sql-server-ver15) of the following code.  
* this branch uses an AKS private cluster
* we are not using AD auth mode

From shell.azure.com:

```bash
# change vars as needed
# this block needs to be rerun if cloudshell times out
# run `watch ls` in cloudshell to avoid the timeout and ctl+c to abort it
export SUBSCRIPTION=davew
export RG=BDC_POC_private
export LOCATION=eastus
export VNET=aks_net_private
export ADDR_PREFIX="10.38.0.0/16"
export SUBNET=bdc_subnet1_private
export SUB_PREFIX="10.38.1.0/24"
export AKS_NAME=BDC-AKS-PRIVATE
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
    --client-secret $SP_PWD \
    --enable-private-cluster

# test connectivity
# this can also be run locally later to add the context to .kube/config file on my laptop
az aks get-credentials \
  --resource-group $RG \
  --name $AKS_NAME \
  --overwrite-existing \
  --admin

# verify we are connected to the correct AKS
kubectl config current-context
kubectl get nodes


# to delete the Az resources later
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
azdata bdc config init --source aks-dev-test --path custom --force

# cd to wherever you cloned this repo
# this will change the bdc name from whatever you generated above
azdata bdc config replace -p custom\bdc.json -j metadata.name=$BDC_NAME
azdata bdc config replace -p custom\control.json -j metadata.name=$BDC_NAME
# create the cluster
azdata bdc create -c custom --accept-eula yes
# admin
# Password01!!

# to watch the status from cloudshell
kubectl get pods -n bdc-aks

kubectl get pods -n $BDC_NAME
azdata login -n $BDC_NAME

# list the endpoints
azdata bdc endpoint list -o table

# get external IP
kubectl get svc controller-svc-external -n $BDC_NAME
# this is mine:  20.75.133.232
# alternative login
azdata login --endpoint https://IP:30080 --username admin

# healthcheck
azdata bdc status show
azdata bdc sql status show

# if needed, remove the cluster
azdata bdc delete --name $BDC_NAME
```
## Setup 

You can connect to the SQL master instance front-end from ADS by using this syntax: 
you can find this Endpoint using 

`20.75.131.120,31433`

In ADS you can also connect to the BDC dashboard

## Firewalling

The default is no NSG so everything should connect.  However, that may not be what is desired.  


## Next Steps

* [Mount ADLS2 data lake](mount_storage.md)
* [Mount blob storage](mount_blob.md)
* [Mount S3](https://docs.microsoft.com/en-us/sql/big-data-cluster/hdfs-tiering-mount-s3?view=sql-server-ver15)
* [Prep the cluster for data virtualization](virtualization.ipynb):  Open this in ADS and follow along.
* [Connect to on-prem SQL Server](sql.md)
* [Query Parquet and Delta Files](parquet.md)

## Create a SQL Server on a VM to test virtualization

```Powershell
$ResourceGroupName = "sqlvm1"
$Location = "East US"
$SubnetName = $ResourceGroupName + "subnet"
$VnetName = $ResourceGroupName + "vnet"
$PipName = $ResourceGroupName + $(Get-Random)

#Create a resource group ir required
New-AzResourceGroup -Name $ResourceGroupName -Location $Location

# Create a subnet configuration
$SubnetConfig = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix 192.168.1.0/24

# Create a virtual network
$Vnet = New-AzVirtualNetwork -ResourceGroupName $ResourceGroupName -Location $Location `
   -Name $VnetName -AddressPrefix 192.168.0.0/16 -Subnet $SubnetConfig

# Create a public IP address and specify a DNS name
$Pip = New-AzPublicIpAddress -ResourceGroupName $ResourceGroupName -Location $Location `
   -AllocationMethod Static -IdleTimeoutInMinutes 4 -Name $PipName

# Rule to allow remote desktop (RDP)
$NsgRuleRDP = New-AzNetworkSecurityRuleConfig -Name "RDPRule" -Protocol Tcp `
   -Direction Inbound -Priority 1000 -SourceAddressPrefix * -SourcePortRange * `
   -DestinationAddressPrefix * -DestinationPortRange 3389 -Access Allow

#Rule to allow SQL Server connections on port 1433
$NsgRuleSQL = New-AzNetworkSecurityRuleConfig -Name "MSSQLRule"  -Protocol Tcp `
   -Direction Inbound -Priority 1001 -SourceAddressPrefix * -SourcePortRange * `
   -DestinationAddressPrefix * -DestinationPortRange 1433 -Access Allow

# Create the network security group
$NsgName = $ResourceGroupName + "nsg"
$Nsg = New-AzNetworkSecurityGroup -ResourceGroupName $ResourceGroupName `
   -Location $Location -Name $NsgName `
   -SecurityRules $NsgRuleRDP,$NsgRuleSQL

$InterfaceName = $ResourceGroupName + "int"
$Interface = New-AzNetworkInterface -Name $InterfaceName `
   -ResourceGroupName $ResourceGroupName -Location $Location `
   -SubnetId $VNet.Subnets[0].Id -PublicIpAddressId $Pip.Id `
   -NetworkSecurityGroupId $Nsg.Id


# Define a credential object
#CHANGE THE PASSWORD
#azureadmin is the login
$SecurePassword = ConvertTo-SecureString '<password>' `
   -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential ("azureadmin", $securePassword)


# Create a virtual machine configuration
$VMName = $ResourceGroupName + "VM"
$VMConfig = New-AzVMConfig -VMName $VMName -VMSize Standard_DS13_V2 |
   Set-AzVMOperatingSystem -Windows -ComputerName $VMName -Credential $Cred -ProvisionVMAgent -EnableAutoUpdate |
   Set-AzVMSourceImage -PublisherName "MicrosoftSQLServer" -Offer "SQL2019-WS2019" -Skus "SQLDEV" -Version "latest" |
   Add-AzVMNetworkInterface -Id $Interface.Id

# Create the VM
New-AzVM -ResourceGroupName $ResourceGroupName -Location $Location -VM $VMConfig

# Get the existing compute VM
$vm = Get-AzVM -Name <vm_name> -ResourceGroupName <resource_group_name>
        
# Register SQL VM with 'Lightweight' SQL IaaS agent
New-AzSqlVM -Name $vm.Name -ResourceGroupName $vm.ResourceGroupName -Location $vm.Location `
  -LicenseType PAYG -SqlManagementType LightWeight

Get-AzPublicIpAddress -ResourceGroupName $ResourceGroupName | Select IpAddress

```

# Setup Virtualization
 There are multiple ways to setup the virtualization:
 * From Azure Data Studio
 <br> After creating a database you can right click and then choose the "Virtualize Data" option then follow the wizard
 <br>![image](https://user-images.githubusercontent.com/49620357/122288734-efab7a00-cebf-11eb-899d-1601f8d3a0af.png)
 
 * Use polybase to
   * Create a Master Key
   * Create a Database Scoped Credential
   * Create an external source (SQL server, Oracle, Terradata, etc)
   * Create an external table
  ```
  CREATE MASTER KEY ENCRYPTION BY PASSWORD = N'MYPASSORD';
  CREATE DATABASE SCOPED CREDENTIAL [MYCREDENTIAL]
    WITH IDENTITY = N'LOGIN', SECRET = N'mySECRET';
  CREATE EXTERNAL DATA SOURCE [MYREMOTESQLSERVER]
  WITH (LOCATION = N'sqlserver://MYIP:1433', CREDENTIAL = [MYCREDENTIAL]);
  ```
 <br>![image](https://user-images.githubusercontent.com/49620357/122289904-38affe00-cec1-11eb-959b-a330f7db3ab1.png)
 

 
 
# Resources
* https://docs.microsoft.com/en-us/sql/big-data-cluster/deployment-guidance?view=sql-server-ver15
* https://docs.microsoft.com/en-us/sql/big-data-cluster/tutorial-load-sample-data?view=sql-server-ver15
* https://docs.microsoft.com/en-us/sql/azdata/reference/reference-azdata-bdc?view=sql-server-ver15
* https://github.com/microsoft/sql-server-samples/tree/master/samples/features/sql-big-data-cluster 
* https://docs.microsoft.com/en-us/sql/big-data-cluster/tutorial-load-sample-data?view=sql-server-ver15 
