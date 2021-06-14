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

# Step 1 BDC creation

You will be having different options to create the BDC.
<br> You can see the list of options by running the command : **azdata bdc config list -o table**
<br> Choose option number 1 aka AKS \o/
