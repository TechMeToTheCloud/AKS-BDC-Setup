

With a private AKS cluster we can't connect without doing some additional steps.  

We need to be able to connect into our vnet to manage AKS and run the azdata commands.  

To do this we can build a VM in the vnet where we can install the necessary tooling.

Other options:  
* ExpressRoute
* vnet peering
* Azure Point-to-Site VPN

## Deploying a VM to manage the AKS cluster

We will deploy a windows VM on the vnet/subnet and then use Azure Bastion to connect to it to install SQL BDC.  

Open cloudshell to a bash terminal

```bash
export SUBSCRIPTION=davew
export RG=BDC_POC_private
export LOCATION=eastus
export VNET=aks_net_private
export ADDR_PREFIX="10.38.0.0/16"
export SUBNET=bdc_subnet1_private
export MGMTSUBNET=management
export MGMT_SUB_PREFIX="10.38.2.0/24"
export SUB_PREFIX="10.38.1.0/24"
export VMName=winvm
export VMUSER=azureuser
export SECRT="AdminAdmin01##"
export MGMTIP="10.38.2.10"

# create a subnet for management vm
az network vnet subnet create \
    --resource-group $RG \
    --vnet-name $VNET \
    --name $MGMTSUBNET \
    --address-prefixes $MGMT_SUB_PREFIX

az vm create \
    --resource-group $RG \
    --name $VMName \
    --image win2016datacenter \
    --admin-password $SECRT \
    --location $LOCATION \
    --size Standard_DS2_v2 \
    --nsg "" \
    --nsg-rule NONE \
    --private-ip-address $MGMTIP \
    --subnet $MGMTSUBNET \
    --vnet-name $VNET \
    --public-ip-address ""
 #   --admin-username $VMUser 

```

Now create Azure Bastion service from the portal  

Connect to the vm with bastion and install the needed software from README.md