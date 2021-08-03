```sql

CREATE EXTERNAL DATA SOURCE BlobStorage 
with (
    TYPE = HADOOP,
    LOCATION='wasbs://<blob_container_name>@<azure_storage_account_name>.blob.core.windows.net',
    CREDENTIAL = AzureStorageCredential
);

```