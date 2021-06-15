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
<br> **AZDATA_USERNAME** and **AZDATA_PASSWORD** are the environment variavkes to be set
<br> Choose option number 1 aka AKS \o/ when running the command **azdata bdc create** to create your bdc

# Step 2 Deployment
BDC creation can take time.
<br> The final ouput should be:
<br>_**Cluster deployed successfully.**_
<br> Except modified via a configuration file (we did not do it) the default BDC name should be **mssql-cluster**.

# Step 3 Save couple of information/handy command
<br> External IP by running the following command: **kubectl get svc controller-svc-external -n mssql-cluster**
<br> Log to the BDC by running **azdata login --endpoint https://<ip-address-of-controller-svc-external>:30080 --username <user-name>**
<br> Get all the endpoints of the BDC by running **azdata bdc endpoint list -o table** or **kubectl get svc -n mssql-cluster**
<br> Check BDC status **azdata bdc status show**
<br> Check BDC SQL server services **azdata bdc sql status show**
