---
title: "Monitor Athena Datalake Tables"
date: 2022-12-26T19:42:36+02:00
draft: false
---

Today we are going to talk about monitoring our datalake tables. 

Let's assume we have a datalake containing more than 3PB of data. 
The raw data is saved as compressed parquet files in S3. 
We query our datalake using athena. 
We needed a way to monitor our data to be able to quickly detect issues in our data pipeline. 

We decided we want to be able to monitor various scenarios. 
1. all of our tables size to prevent incorrect lifecycle policies and retention of the data or accidental
deletions of entire tables. We will create an alert in case the table size 
is lower than an allowed threshold.
1. delete events in our tables.
2. partitions that have been deleted, and their size is now 0MB.

We looked at a couple of different tools and solutions to try and give us the most coverage with the lowest cost.

## First approach - direct querying of datalake table

The advantages of this approach are clear. It is of course the simplest. We could monitor the size of the table using a 
simple ETL process that will query the athena table to get the count of rows in the table.
For simplicity, from now on we will assume our datalake consists of a single table - `datalake.purchase_events`
A simple query in the form of `select count(*) from datalake.purchase_events` can retrieve the amount of rows in the table. 
The query does not scan any data, it only queries the metadata files of the underlying data in s3.
The issue is that although the query does not scan the data itself, this solution still cost us ~200$ a day!
why, you ask? the reason is that most of our data is stored in SIA (AWS standard infrequent access) and the cost 
of even retrieving the metadata file from the SIA is very high. That is why we decided to try a different approach


## Second approach - access logs

Due to security requirements, we already store access logs on the datalake bucket. Access logs can notify us on all the 
events that take place on files in the bucket (GET, DELETE, POST, LIST). We can use the logs to alert us in case large 
quantities of data have been marked for deletion, or deleted. Also, we can create an aggregate ETL that will index the size 
of the tables and change it according to the events on the table. We started with a POC to test this solution.

The first step was to create an athena table from the access logs so it will be more convenient for us to query the logs.
I used [this guide](https://aws.amazon.com/blogs/big-data/easily-query-aws-service-logs-using-amazon-athena/) to set up the athena table.

This is how you create the table:
```sql
CREATE EXTERNAL TABLE `s3_access_logs_db.mybucket_logs`(
  `bucketowner` STRING,
  `bucket_name` STRING,
  `requestdatetime` STRING,
  `remoteip` STRING,
  `requester` STRING,
  `requestid` STRING,
  `operation` STRING,
  `key` STRING,
  `request_uri` STRING,
  `httpstatus` STRING,
  `errorcode` STRING,
  `bytessent` BIGINT,
  `objectsize` BIGINT,
  `totaltime` STRING,
  `turnaroundtime` STRING,
  `referrer` STRING,
  `useragent` STRING,
  `versionid` STRING,
  `hostid` STRING,
  `sigv` STRING,
  `ciphersuite` STRING,
  `authtype` STRING,
  `endpoint` STRING,
  `tlsversion` STRING)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
  'input.regex'='([^ ]*) ([^ ]*) \\[(.*?)\\] ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) (\"[^\"]*\"|-) (-|[0-9]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) (\"[^\"]*\"|-) ([^ ]*)(?: ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*))?.*$')
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://awsexamplebucket1-logs/prefix/'
```

to fetch the access logs data we can use the following query:
```sql
SELECT * FROM "s3_access_logs_db.mybucket_logs" limit 10;
```

After creating the table we noticed that there is no easy way to partition the data. 
The only way we found that can reduce the query cost is using aggressive lifecycle policy on the bucket to delete old records,
which we could not use due to security requirements. Also, querying access logs of a single day still holds a lot of data - around 70GB per query.
Without date partitions, querying the access logs table proved too expensive.


## Third approach - using cloudtrail
When creating an athena table to query cloudtrail logs, it is possible to add partitions to the table.
you can follow [this guide](https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html) to create the athena table:
```sql
CREATE EXTERNAL TABLE `cloudtrail_logs_mybucket`(
  `eventversion` string COMMENT 'from deserializer', 
  `useridentity` struct<type:string,principalid:string,arn:string,accountid:string,invokedby:string,accesskeyid:string,username:string,sessioncontext:struct<attributes:struct<mfaauthenticated:string,creationdate:string>,sessionissuer:struct<type:string,principalid:string,arn:string,accountid:string,username:string>>> COMMENT 'from deserializer', 
  `eventtime` string COMMENT 'from deserializer', 
  `eventsource` string COMMENT 'from deserializer', 
  `eventname` string COMMENT 'from deserializer', 
  `awsregion` string COMMENT 'from deserializer', 
  `sourceipaddress` string COMMENT 'from deserializer', 
  `useragent` string COMMENT 'from deserializer', 
  `errorcode` string COMMENT 'from deserializer', 
  `errormessage` string COMMENT 'from deserializer', 
  `requestparameters` string COMMENT 'from deserializer', 
  `responseelements` string COMMENT 'from deserializer', 
  `additionaleventdata` string COMMENT 'from deserializer', 
  `requestid` string COMMENT 'from deserializer', 
  `eventid` string COMMENT 'from deserializer', 
  `resources` array<struct<arn:string,accountid:string,type:string>> COMMENT 'from deserializer', 
  `eventtype` string COMMENT 'from deserializer', 
  `apiversion` string COMMENT 'from deserializer', 
  `readonly` string COMMENT 'from deserializer', 
  `recipientaccountid` string COMMENT 'from deserializer', 
  `serviceeventdetails` string COMMENT 'from deserializer', 
  `sharedeventid` string COMMENT 'from deserializer', 
  `vpcendpointid` string COMMENT 'from deserializer')
COMMENT 'CloudTrail table for mybucket'
ROW FORMAT SERDE 
'com.amazon.emr.hive.serde.CloudTrailSerde' 
STORED AS INPUTFORMAT 
'com.amazon.emr.cloudtrail.CloudTrailInputFormat' 
OUTPUTFORMAT 
'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
's3://access_logs_bucket_path/AWSLogs/account_id/CloudTrail'
TBLPROPERTIES (
'classification'='cloudtrail', 
'transient_lastDdlTime'='1605028457')
```

From what i gathered cloudtrail costs for data events storage is very high. Just the storage of data events trails for the entire bucket will cost us around ~150$ a day.
It will make the cloudtrail solution unusable. 
cost estimate taken from the following calculation: 
`307446651 / 100000 * 0.1`. cloudtrail pricing taken from [here](https://aws.amazon.com/cloudtrail/pricing/) (paid tier)

Even after reducing the cloudtrail logs only to the athena folder, storage costs alone are at around $30 a day for our scale of data.

## Fourth approach - using S3 inventory
After trying a lot of different solutions we settled on using S3 inventory to monitor our tables.
[S3 inventory](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html) is a service that helps you manage your storage. It gives you a daily report on the 
state of every file in your bucket. Also, you can use athena to query your inventory files. First you need to enable it, select the source bucket and the destination folder for the 
inventory file. After you enable the service you can create an Athena table to query the data.
Another great thing about S3 inventory is that it does not cost a lot of money. Storage costs for the destination folder are around ~$3 dollars for daily data.
Only con for this solution is that there is no way to use it for real time monitoring since the report is sent once a day.
here is the query to generate the table:

```sql
CREATE EXTERNAL TABLE `s3-inventory-athena-table`(
  `bucket` string, 
  `key` string, 
  `version_id` string, 
  `is_latest` boolean, 
  `is_delete_marker` boolean, 
  `size` bigint, 
  `last_modified_date` bigint, 
  `e_tag` string, 
  `storage_class` string, 
  `is_multipart_uploaded` boolean, 
  `replication_status` string, 
  `encryption_status` string, 
  `object_lock_retain_until_date` bigint, 
  `object_lock_mode` string, 
  `object_lock_legal_hold_status` string, 
  `intelligent_tiering_access_tier` string, 
  `bucket_key_status` string, 
  `checksum_algorithm` string)
PARTITIONED BY ( 
  `dt` string)
ROW FORMAT SERDE 
  'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.SymlinkTextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://your_bucket_path/AWSLogs/account_id/CloudTrail'
TBLPROPERTIES (
  'projection.dt.format'='yyyy-MM-dd-HH-mm', 
  'projection.dt.interval'='1', 
  'projection.dt.interval.unit'='DAYS', 
  'projection.dt.range'='2022-01-01-00-00,NOW', 
  'projection.dt.type'='date', 
  'projection.enabled'='true')
```

For more information follow [this guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory-athena-query.html)

After creating the table we set out to creating our DAG. 
We use airflow for our ETL workloads so we created a DAG that will run every 12 hours and send a metric to our Graylog metric service.

### monitor tables size
To monitor the sizes of our athena tables we used the following query:
```python
def monitor_athena_tables_size(self):
    query = f"""
    select 
        split_part(key, '/', 5) as bucket_path, 
        sum(size) bucket_count
    from default.s3-inventory-athena-table
    where dt = '{datetime.datetime.now() - datetime.timedelta(days=1)}'
        and is_latest
        and not is_delete_marker
        and split_part(key, '/', 4) = 'athena'
        and split_part(key, '/', 2) = 'production'
        and split_part(key, '/', 7) = 'output' -- only check output folder
    group by 1
    """
    self._logger.info(f"query: {query}")
    result = pd.read_sql(query, self.connection)
    for _, row in result.iterrows():
        self._metrics.value(
            measurement_name="athena_delete_monitoring_table_size",
            value=row[1],
            bucket_path_id=row[0],
        )
```

This code snippets queries the s3 inventory data from 1 day ago, splits it by each table id, and uses `sum(size)` to aggregate the total size of each table.
After we execute the query we send a gauge metric to Graylog (using statsd) with the id of the table and the sum of its size. 
Here is an example of how we use that metric to create a grafana dashboard with an alert in case the size of the table drops:
![purchase events table size](https://drive.google.com/uc?id=11_82kE-rGXqrT9k6JoGJb0vJp7m-rvyz)

### monitor delete events
We want to be aware if we see an unusual increase in delete events. Our S3 bucket is set up using versioning so initially files are not really deleted, they 
simply mark the file as deleted using a delete marker 
(Notice that in the previous query to calculate the total size of each table we filtered files which are not marked as latest and also files who are marked as deleted).
So to get a count of delete markers per table we can use the following code snippet:

```python
def monitor_athena_tables_delete_markers(self):
    query = f"""
    select 
        split_part(key, '/', 5) as bucket_path, 
        count(*) bucket_count 
    from default.s3-inventory-athena-table
    where dt = '{datetime.datetime.now() - datetime.timedelta(days=1)}'
        and is_latest
        and is_delete_marker
        and split_part(key, '/', 4) = 'athena'
        and split_part(key, '/', 2) = 'production'
        and split_part(key, '/', 7) = 'output' -- only check output folder
    group by 1
    """
    self._logger.info(f"query: {query}")
    result = pd.read_sql(query, self.connection)
    for _, row in result.iterrows():
        self._metrics.value(
            measurement_name="athena_delete_monitoring_delete_markers_count",
            value=row[1],
            bucket_path_id=row[0],
        )
```

This code snippets queries the s3 inventory data from 1 day ago, splits it by each table id, and uses `count(*)` to aggregate the count of delete markers per table.
After we execute the query we send a gauge metric to Graylog (using statsd) with the id of the table and the count of delete events. 
Here is an example of how we use that metric to create a grafana dashboard with an alert in case we see a high increase in delete events:
![purchase_events delete markers count](https://drive.google.com/uc?id=11bZ1iVDD6H-EwmFUR1sIwDpzIkVeerpQ)
(It's normal to see a high number of delete events regularly because the table has a lifecycle policy to delete old partitions)

## final words
So this turned out to be quite a long post. In the next post we will continue to review how we use S3 inventory to monitor deletions of data on specific partitions
