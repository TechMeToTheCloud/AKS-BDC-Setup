[From here](https://docs.microsoft.com/en-us/sql/big-data-cluster/hdfs-tiering-mount-adlsgen2?view=sql-server-ver15)

* We will mount with storage access keys

from cmd locally:

```cmd

# changeme
set MOUNT_CREDENTIALS=fs.azure.abfs.account.name=<your-storage-account-name>.dfs.core.windows.net,fs.azure.account.key.<your-storage-account-name>.dfs.core.windows.net=<storage-account-access-key>

#my example
set MOUNT_CREDENTIALS=fs.azure.abfs.account.name=davewdemodata.dfs.core.windows.net,fs.azure.account.key.davewdemodata.dfs.core.windows.net=...

echo %MOUNT_CREDENTIALS%

# this will get the IP address
kubectl get svc controller-svc-external -n bdc-aks

azdata login -e https://20.75.133.232:30080

azdata bdc hdfs mount create --remote-uri abfs://lake@davewdemodata.dfs.core.windows.net/ --mount-path /mounts/lake

azdata bdc hdfs mount status

# delete the mount
azdata bdc hdfs mount delete --mount-path <mount-path-in-hdfs>
```

We should now be able to see the mount in ADS and query it.  