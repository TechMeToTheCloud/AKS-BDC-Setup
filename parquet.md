This also works for delta format although there may be a better way to do that ([see here](https://docs.microsoft.com/en-us/sql/big-data-cluster/package-management-delta-lake?view=sql-server-ver15))

```sql
CREATE EXTERNAL FILE FORMAT file_format_name
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec' --or GzipCodec
);

```