# AKS-BDC-Setup

# Prerequisites
* azdata tool
  * https://docs.microsoft.com/en-us/sql/azdata/install/deploy-install-azdata?view=sql-server-ver15
* kubectl command line tool
  * https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-kubectl/
* Azure Data Studio
  * https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15
* Data Virtualization extension for Azure Data Studio
  * https://docs.microsoft.com/en-us/sql/azure-data-studio/extensions/data-virtualization-extension?view=sql-server-ver15
* A managed Kubernetes container service in Azure

# Step 1 BDC creation

You will have different options to create the BDC.
<br> You can see the list of options by running the command : **azdata bdc config list -o table**
<br> You can set up Environment Variables before creating the BDC (otherwise it will be asked when creating it)
<br> **AZDATA_USERNAME** and **AZDATA_PASSWORD** are the environment to be set
<br> Choose option number 1 aka AKS \o/ when running the command **azdata bdc create** to create your bdc

# Step 2 Deployment
BDC creation can take time.
<br> The final ouput should be
**Cluster deployed successfully.**
