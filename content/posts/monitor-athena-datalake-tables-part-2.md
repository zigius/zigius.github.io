---
title: "Monitor Athena Datalake Tables Part 2"
date: 2023-01-01T16:12:24+02:00
draft: false
---

In the [previous post](https://zigius.github.io/posts/monitor-athena-datalake-tables/) we explored various monitoring solutions 
for monitoring athena tables. 
One of the scenarios we wanted to be able to monitor, was a case where a partition has been unintentinally deleted from an athena table

We decided we will use S3 inventory to monitor this flow. 

This is how we do it:

First let's try and break down the pseudo code we need to write:
```text
1. get all athena tables partitions, using glue
2. get all paths in our s3 datalake bucket
3. group paths list by the location of the date partition. *ORDER MATTERS* (example: purchase_events table has three partitions - purchase_date, supplier, provider. will be placed in group 0).
This is required in order to query the data efficiently.
4. for each group, get size of each partition in group
5. Filter partitions with invalid dates
6. send metric for each date partition with it's size, date, and table properties.
```

First, let's look at the code snippet of the module and go over the code implementation. Don't worry, we will soon break it down to smaller pieces.
```python
def monitor_athena_partitions_deletions(self):

    athena_tables = self._get_all_tables()
    all_bucket_paths = self._get_all_bucket_paths()
    tagged_paths = [self._get_tags(bucket_path_id=o) for o in all_bucket_paths.bucket_path.values]
    paths_grouped_by_date_partition = self._get_paths_grouped_by_date_partition(tagged_paths, athena_tables)

    dfs = []

    for k, v in paths_grouped_by_date_partition.items():
        bucket_paths_ids = [f"'{o['bucket_path_id']}'" for o in v]
        query = f"""
        select
            split_part(key, '/', 5) as bucket_path,
            split_part(key, '/', {8 + k}) as partition_field,
            sum(size) as sum_size
        from "{AsIs(self._config["s3_inventory_table_schema"])}"."{AsIs(self._config["table_name"])}"
        where dt = '{self._get_dt()}'
            and is_latest
            and not is_delete_marker
            and split_part(key, '/', 4) = 'athena'
            and split_part(key, '/', 2) = 'production'
            and split_part(key, '/', 7) = 'output' -- only check output folder
            and split_part(key, '/', 5) in ({",".join(bucket_paths_ids)})
        group by 1, 2
        """
        self._logger.info(f"query: {query}")
        res = pd.read_sql(query, self.connection)
        dfs.append(res)

    df = pd.concat(dfs)

    # filter outputs where the date partition column in not really a date.
    df["mask"] = df["partition_field"].str.contains(r".*=\d{4}-\d{2}-\d{2}")
    df = df[df["mask"] == True]
    df = df.drop(columns=["mask"])

    # filter column where the date partition is invalid. example: 2339-06-03
    df["partition_fields_clean"] = df["partition_field"].apply(lambda x: x.split("=")[1])
    df["is_valid_date"] = pd.to_datetime(df["partition_fields_clean"], format="%Y-%m-%d", errors="coerce").notna()
    df = df[df["is_valid_date"] == True]
    df = df.drop(columns=["partition_fields_clean", "is_valid_date"])

    # filter column where the date partition is in the future
    df["is_future"] = df["partition_field"].apply(lambda x: datetime.datetime.strptime(x.split("=")[1], "%Y-%m-%d") > datetime.datetime.now())
    df = df[df["is_future"] == False]
    df = df.drop(columns=["is_future"])

    for _, row in df.iterrows():

        tags = self._get_tags(bucket_path_id=row["bucket_path"])
        self._metrics.value(
            measurement_name="athena_delete_monitoring_partition_sizes",
            value=row["sum_size"],
            tags={**tags, **{"partition_date": row["partition_field"].split("=")[1]}},
        )
```

The first line in the code snippet uses glue to get all the athena tables
```python
def monitor_athena_partitions_deletions(self):
    athena_tables = self._get_all_tables()
    # ...

def _get_all_tables(self, database_name="datalake"):
    import boto3

    client = boto3.client("glue")

    # Create an empty list to store the tables
    response = client.get_tables(DatabaseName=database_name)
    tables = response["TableList"]

    # Use a loop to retrieve all the tables from the database
    while response.get("NextToken", None):
        # Call the get_tables method of the AWS Glue client, passing the NextToken parameter if it's not empty
        response = client.get_tables(DatabaseName=database_name, NextToken=response["NextToken"])

        # Append the tables from the response to the list of tables
        tables.extend(response["TableList"])

    return tables
```
We use the glue boto client to retrieve all the metadata for all of our athena tables.
We need the metadata of the tables to determine which table has which partitions.

Now, we continue to get all the S3 bucket paths
```python
def monitor_athena_partitions_deletions(self):
    # ...
    all_bucket_paths = self._get_all_bucket_paths()
    # ...

def _get_all_bucket_paths(self):
    query = f"""
    select
        split_part(key, '/', 5) as bucket_path
    from "{AsIs(self._config["s3_inventory_table_schema"])}"."{AsIs(self._config["table_name"])}"
    where dt = '{self._get_dt()}'
        and split_part(key, '/', 4) = 'athena'
        and split_part(key, '/', 2) = 'production'
        and split_part(key, '/', 7) = 'output' -- only check output folder
    group by 1
    """
    self._logger.info(f"query: {query}")
    res = pd.read_sql(query, self.connection)
    return res
```
We use our s3 inventory table to get all the paths that for all of our athena tables. An example key looks something like this:

```fs
<bucket-name>/datalake/production/outputs/athena/<output-id>/<output-version>/output/<partition_field1>=<partition_field1value>/<partition_date>=<partition_date_value>/compaction_id=6/2022_12_22_11_51_output.parquet.p00017
```

We want to find all the output ids (located at the fifth slash in the key). 
Check out the previous article for more info.

Let's continue and get all the paths grouped by date.
```python
def monitor_athena_partitions_deletions(self):
    # ...
    tagged_paths = [self._get_tags(bucket_path_id=o) for o in all_bucket_paths.bucket_path.values]
    paths_grouped_by_date_partition = self._get_paths_grouped_by_date_partition(tagged_paths, athena_tables)
    # ...

def _get_tags(self, bucket_path_id):
    return {
        "table_name": self._get_table_name(bucket_path_id),
        "bucket_path_id": bucket_path_id,
    }
    return tags

def _get_paths_grouped_by_date_partition(self, tagged_paths, athena_tables):
    grouped_paths = {}
    for tagged_path in tagged_paths:
        athena_table = [o for o in athena_tables if o.get("Name", None) == tagged_path.get("table_name", "")]
        if not athena_table:
            continue
        for i, v in enumerate(athena_table[0].get("PartitionKeys", [])):
            if v["Type"] == "date":
                if i not in grouped_paths:
                    grouped_paths[i] = [tagged_path]
                else:
                    grouped_paths[i].append(tagged_path)

    return grouped_paths
```
This code snippet groups the athena tables by the location of their date partition. We assume all athena tables have 1 date partition key, but can have multiple other partitions. 
We are writing a query that aggregates data by the date partition, so to reduce calls to S3 inventory we group all the tables where the date partition is in the same location in the path together.

Now to run the actual queries
```python
def monitor_athena_partitions_deletions(self):
    # ...
    dfs = []

    for k, v in paths_grouped_by_date_partition.items():
        bucket_paths_ids = [f"'{o['bucket_path_id']}'" for o in v]
        query = f"""
        select
            split_part(key, '/', 5) as bucket_path,
            split_part(key, '/', {8 + k}) as partition_field,
            sum(size) as sum_size
        from "{AsIs(self._config["s3_inventory_table_schema"])}"."{AsIs(self._config["table_name"])}"
        where dt = '{self._get_dt()}'
            and is_latest
            and not is_delete_marker
            and split_part(key, '/', 4) = 'athena'
            and split_part(key, '/', 2) = 'production'
            and split_part(key, '/', 7) = 'output' -- only check output folder
            and split_part(key, '/', 5) in ({",".join(bucket_paths_ids)})
        group by 1, 2
        """
        self._logger.info(f"query: {query}")
        res = pd.read_sql(query, self.connection)
        dfs.append(res)

    df = pd.concat(dfs)
    # ...
```
This code runs the actual queries against the S3 inventory table, and aggregates the results in a single datafram, `df`.

Now we clean our dataframe and remove invalid rows
```python
def monitor_athena_partitions_deletions(self):

    # ...
    # filter outputs where the date partition column in not really a date.
    df["mask"] = df["partition_field"].str.contains(r".*=\d{4}-\d{2}-\d{2}")
    df = df[df["mask"] == True]
    df = df.drop(columns=["mask"])

    # filter column where the date partition is invalid. example: 2339-06-03
    df["partition_fields_clean"] = df["partition_field"].apply(lambda x: x.split("=")[1])
    df["is_valid_date"] = pd.to_datetime(df["partition_fields_clean"], format="%Y-%m-%d", errors="coerce").notna()
    df = df[df["is_valid_date"] == True]
    df = df.drop(columns=["partition_fields_clean", "is_valid_date"])

    # filter column where the date partition is in the future
    df["is_future"] = df["partition_field"].apply(lambda x: datetime.datetime.strptime(x.split("=")[1], "%Y-%m-%d") > datetime.datetime.now())
    df = df[df["is_future"] == False]
    df = df.drop(columns=["is_future"])
    # ...
```

And lastly, we send a metric for every partition size we have gathered
```python
def monitor_athena_partitions_deletions(self):
    # ...
    for _, row in df.iterrows():

        tags = self._get_tags(bucket_path_id=row["bucket_path"])
        self._metrics.value(
            measurement_name="athena_delete_monitoring_partition_sizes",
            value=row["sum_size"],
            tags={**tags, **{"partition_date": row["partition_field"].split("=")[1]}},
        )
```

This task sends a metric with the size of each date partition, tagged with the name and id of the table. 
We can create a dashboard in grafana that will monitor empty partitions.
Here is how the dashboard looks like: 
![Empty partitions panel](https://drive.google.com/uc?id=11hKLY-2l0PIx8QrFjXoxWuLj9-6y78Oo)

This is how the panel is defined:
![Alert details](https://drive.google.com/uc?id=11nGJ3soe2VBunlnon8rtbaVDZv8EbA61)
We send an alert in case we found a single partition with a size that is lower than 1000 bytes 
(we currently have a table with a partition that we know is smaller than 1000 bytes, so we alert of more than one partition with that size, but the core idea remains the same). 

This concludes our datalake monitoring efforts. There is always a place to improve and we will surely fine tune the process as we continue
to improve our monitoring tools and capabilities, for example anomaly detection will be very useful here, 
but for now i feel like this solution gives us a pretty high coverage for a low cost.
