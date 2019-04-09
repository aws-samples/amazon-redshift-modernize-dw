# ADB305 - Modernize your data warehouse with Amazon Redshift

Can you set up a data warehouse and create a dashboard in under 60 minutes? In this workshop, we show you how with Amazon Redshift, a fully managed cloud data warehouse that provides first-rate performance at the lowest cost for queries across your data warehouse and data lake. Learn the steps and best practices for deploying your data warehouse in your organization. Also, learn how to query petabytes of data in your data warehouse and exabytes of data, without loading or moving, in your Amazon S3 data lake. Finally, learn how to easily migrate from traditional or on-premises data warehouses. Please be sure to bring your laptop.


```
Matt Scaer
Principal DW Specialist SA
AWS

```


## License Summary

This sample code is made available under a modified MIT license. See the LICENSE file.

## Agenda
* Introductions
* Account Logins and Cluster Spin-up
* Refresher on Amazon Redshift
* Workshop time

## Why this session
* Data typically grows at 10x every 5 years.
* Average lifetime for an Analytics Platform is 15yrs. 
* Not just price and performance but also complexity. 

## Benefits of this session

* To learn how simple it is:
	* To query 2.87 billion rows (200Gb’s of data) in <5 seconds.
	* Query historical data residing on S3.
	* Plan for the future.

## Account Login and Redshift Cluster Spin-up 
* You have the option to use workshops credits with your account or get a temporary account (slip of paper).
* Log into AWS using the provided placeholder credentials, then switch the **us-west-2**.
* We are here to help, please don’t hesitate to ask for assistance!
* Create an IAM role to query S3 data, giving the role read-only access to all Amazon S3 buckets. Make sure to choose **AmazonS3ReadOnlyAccess** and **AWSGlueConsoleFullAccess**

````
https://docs.aws.amazon.com/redshift/latest/dg/c-getting-started-using-spectrum-create-role.html 
````

* Use Redshift’s ‘Quick Create’ functionality (or “Classic”) to create a cluster and associate the IAM role with it.
	* Please use **2 compute nodes of DC2.Large**, using a cluster identifier, and master user of your choice.
	* **Do not pick an AZ**
	* Double-check you are in the **us-west-2** region


<details><summary>How-to Screenshot</summary>
<p>
![GitHub Logo](/images/redshift_launch.png) 
</p>
</details>

## Refresher on Amazon Redshift

![GitHub Logo](/images/redshift_arch.png)

* Massively parallel, shared nothing columnar architecture
* Leader Node:
	* SQL Endpoint
	* Stores Metadata
	* Coordinates parallel SQL Processing
* Compute Node:
	* Local, columnar storage
	* Executes queries in parallel
	* Load, unload, backup, restore
* Amazon Redshift Spectrum nodes
	* Execute queries directly against 
	* Amazon Simple Storage Service (Amazon S3)

### Two Complimentary Usage Patterns

Amazon Redshift combines two usage patterns under a single, seamless service:

* Redshift (using direct-attached storage objects):
	* Billed hourly for the number and type of compute nodes in the cluster.
	* An “all you can eat” model.
* Redshift Spectrum (table data resides on S3):
	* Billed at $5 per TB of data scanned.
	* Both performance and cost-savings incent reducing the amount of data scanned through:
	* Compressing the data on S3.
	* Storing the data on S3 in a columnar format (eg. Parquet or ORC).
	* Partitioning the data on S3

**Amazon Redshift can be leveraged using the patterns either solely, or in combination.**

### Connecting to the Cluster

* Existing tools like SQL Workbench can be used. 
* Amazon Redshift has a built-in Query Editor via the AWS Management console. 

	![GitHub Logo](/images/query_editor.png)

	and you can type in the following query to get the ball-rolling

	````
	SELECT 'Hello Redshift Tiered-Storage workshop attendee!' AS welcome;
	````
* PgWeb -> Pgweb is a web-based database browser for PostgreSQL, written in Go and works on OSX, Linux and Windows machines. Feel free to download this runtime-client from here:

	````
	https://github.com/sosedoff/pgweb/releases
	````
	
	* PgWeb defaults to port 5432, whereas Redshift defaults to 5439. If you want to change the port in PGWeb to 5439, invoke with:
	
	```
	pgweb --url postgres://{username}:{password}@{cluster_endpoint}:5439/{database_name}?sslmode=require
	```
	
* (optional) Command Line Interface (CLI) for Amazon Redshift.
	
	````
	https://docs.aws.amazon.com/redshift/latest/mgmt/setting-up-rs-cli.html
	````
## Workshop - Scenario #1: What happened in 2016?

* Assemble your toolset:
	* Choosing a SQL editor (SQL Workbench, PGWeb, psql, query from Console, etc.) 

* Load the Green company data for January 2016 into Redshift direct-attached storage (DAS) with COPY.
* Collect supporting/refuting evidence for the impact of the January, 2016 blizzard on taxi usage.
* The CSV data is by month on Amazon S3. Here's a quick screenshot via the CLI: 
	````
	$ aws s3 ls s3://us-west-2.serverless-analytics/NYC-Pub/green/ | grep 2016
	````
	
	![GitHub Logo](/images/s3_ls.png)

* Here's Sample data from the January File:

	````
	head -20 green_tripdata_2016-01.csv 
VendorID,lpep_pickup_datetime,Lpep_dropoff_datetime,Store_and_fwd_flag,RateCodeID,Pickup_longitude,Pickup_latitude,Dropoff_longitude,Dropoff_latitude,Passenger_count,Trip_distance,Fare_amount,Extra,MTA_tax,Tip_amount,Tolls_amount,Ehail_fee,improvement_surcharge,Total_amount,Payment_type,Trip_type 
	````

	![GitHub Logo](/images/jan_file_head.png)
	
	
### Build you DDL 
- Create a schema `workshop_das` and table `workshop_das.green_201601_csv` for tables that will reside on the Redshift compute nodes, AKA the Redshift direct-attached storage (DAS) tables.

	<details><summary>Hint</summary>
	<p>
	
	````
	CREATE SCHEMA workshop_das;

	CREATE TABLE workshop_das.green_201601_csv 
	(
	  vendorid                VARCHAR(4),
	  pickup_datetime         TIMESTAMP,
	  dropoff_datetime        TIMESTAMP,
	  store_and_fwd_flag      VARCHAR(1),
	  ratecode                INT,
	  pickup_longitude        FLOAT4,
	  pickup_latitude         FLOAT4,
	  dropoff_longitude       FLOAT4,
	  dropoff_latitude        FLOAT4,
	  passenger_count         INT,
	  trip_distance           FLOAT4,
	  fare_amount             FLOAT4,
	  extra                   FLOAT4,
	  mta_tax                 FLOAT4,
	  tip_amount              FLOAT4,
	  tolls_amount            FLOAT4,
	  ehail_fee               FLOAT4,
	  improvement_surcharge   FLOAT4,
	  total_amount            FLOAT4,
	  payment_type            VARCHAR(4),
	  trip_type               VARCHAR(4)
	)
	DISTSTYLE EVEN SORTKEY (passenger_count,pickup_datetime);

	````
	</p>
	</details>

### Build your Copy Command 

Build your copy command to copy the data from Amazon S3. This dataset has the number of taxi rides in the month of January 2016.

````
s3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv
````

<details><summary>Hint</summary>
<p>

````
COPY workshop_das.green_201601_csv
FROM 's3://us-west-2.serverless-analytics/NYC-Pub/green/green_tripdata_2016-01.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole'
DATEFORMAT 'auto'
IGNOREHEADER 1
DELIMITER ','
IGNOREBLANKLINES
;
````
**HINT HINT: The `XXXXXXXXXXXX` in the above command should be your AWS account number and Role information.**

</p>
</details>	

### Pin-point the Blizzard and Quickly Graph the Data

In this month, there is a date which had the lowest number of taxi rides due to a blizzard. Can you find that date?

<details><summary>SQL-Based Hint</summary>
<p>

````
SELECT TO_CHAR(pickup_datetime, 'YYYY-MM-DD'),
COUNT(*)
FROM workshop_das.green_201601_csv
GROUP BY 1
ORDER BY 1;
````
</p>
</details>

<details><summary>Graphical Hint</summary>
<p>
<ul>
<li>Navigate to Amazon Quicksight</li>
<li>Create a Data Source</li>
<li>Choose a Table Schema</li>
<li>Choose Table(s)</li>
<li>Optionally import to Spice. The record size is about 1.44 million rows</li>
<li>Select the table workshopdas.green201601csv</li>
<li>In the Fields List, click to the right of pickusdatetime to change the data format to a Date.</li>
<li>The visualization should update with the count of rides by date, pinpointing the lower ride count date.</li>
<ul>
</p>
</details>

## Workshop - Scenario #2: Go Back in Time

* Query historical data residing on S3:
	* Create external DB for Redshift Spectrum.
	* Create the external table on Spectrum.
	* Write a script or SQL statement to add partitions.
	* Create and populate a small number of dimension tables on Redshift DAS.
* Introspect the historical data, perhaps rolling-up the data in novel ways to see trends over time, or other dimensions.
* Enforce reasonable use of the cluster with Redshift Spectrum-specific Query Monitoring Rules (QMR).
	* Test the QMR setup by writing an excessive-use query.
* For the dimension table(s), feel free to leverage multi-row insert in Redshift:
	`https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-multi-row-inserts.html`

**Note the partitioning scheme is Year, Month, Type (where Type is a taxi company). Here's a quick Screenshot:**

````
$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/
                           PRE year=2009/
                           PRE year=2010/
                           PRE year=2011/
                           PRE year=2012/
                           PRE year=2013/
                           PRE year=2014/
                           PRE year=2015/
                           PRE year=2016/
$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/
                           PRE type=fhv/
                           PRE type=green/
                           PRE type=yellow/
$ aws s3 ls s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/type=green/
2017-05-18 19:43:22   18910771 part-r-00000-4c01b1ef-3419-40ba-908e-5b36b3556fa7.gz.parquet
````

### Create external schema (and DB) for Redshift Spectrum

* Create an external schema **adb305** from your database **spectrumdb**

	<details><summary>Hint</summary>
	<p>
	
	```python
	CREATE external SCHEMA adb305
	FROM data catalog DATABASE 'spectrumdb' 
	IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' 
	CREATE external DATABASE if not exists;
	```
	
	</p>
	</details>

### Create your Spectrum table DDL (or use this)

* Create your external table **adb305.NYTaxiRides** for
	`vendorid, pickup_datetime, dropoff_datetime, ratecode, passenger_count, trip_distance, fare_amount, total_amount, payment_type`
	stored in parquet format under location `s3://us-west-2.serverless-analytics/canonical/NY-Pub/`
	<details><summary>Hint</summary>
	<p>
	
	
	```python
	CREATE EXTERNAL TABLE adb305.NYTaxiRides (
    vendorid VARCHAR(6),
    pickup_datetime TIMESTAMP,
    dropoff_datetime TIMESTAMP,
    ratecode INT,
    passenger_count INT,
    trip_distance FLOAT8,
    fare_amount FLOAT8,
    total_amount FLOAT8,
    payment_type INT
    )
  PARTITIONED BY (YEAR INT, MONTH INT, "TYPE" CHAR(6))
  STORED AS PARQUET
  LOCATION 's3://us-west-2.serverless-analytics/canonical/NY-Pub/'
	;

	```
	</p>
	</details>
	
### Add the Partitions

**Note for the Redshift Editor users:** Due to current limitations, the embedded seimicolon will cause an error. Similarly, this version of the editor only allows a single statement to be executed at a time. So, try two of the vendors (including Green) and the months in 2016 to save the time of running statement-by-statement. Users of the other editors should be able to run the partition creations script-as and add the partitions in a single command.

```
WITH generate_smallint_series AS (select row_number() over () as n from workshop_das.green_201601_csv limit 65536)
, part_years AS (select n AS year_num from generate_smallint_series where n between 2009 and 2016)
, part_months AS (select n AS month_num from generate_smallint_series where n between 1 and 12)
, taxi_companies AS (SELECT 'fhv' taxi_vendor UNION ALL SELECT 'green' UNION ALL SELECT 'yellow')

SELECT 'ALTER TABLE adb305.NYTaxiRides ADD PARTITION(year=' || year_num || ', month=' || month_num || ', type=\'' || taxi_vendor || '\') ' ||
'LOCATION \'s3://us-west-2.serverless-analytics/canonical/NY-Pub/year=' || year_num || '/month=' || month_num || '/type=' || taxi_vendor || '/\';'
FROM part_years, part_months, taxi_companies order by 1;

```

### Update the number of rows table property

1. Determine the number of rows in the table.
2. Save a copy of the explain plan for #1 above.
3. Set the (approximate or specific) number of rows using the TABLE PROPERTIES under ALTER EXTERNAL TABLE.
4. Rerun the explain plan for #1, noting the difference. When might this be impactful?

### Add a Redshift Spectrum Query Monitoring Rule to ensure reasonable use

In Amazon Redshift workload management (WLM), query monitoring rules define metrics-based performance boundaries for WLM queues and specify what action to take when a query goes beyond those boundaries. Setup a Query Monitoring Rule to ensure reasonable use.

```
https://docs.aws.amazon.com/redshift/latest/dg/cm-c-wlm-query-monitoring-rules.html
```
Take a look at SVL_QUERY_METRICS_SUMMARY view shows the maximum values of metrics for completed queries. This view is derived from the STL_QUERY_METRICS system table. Use the values in this view as an aid to determine threshold values for defining query monitoring rules.

```
https://docs.aws.amazon.com/redshift/latest/dg/r_SVL_QUERY_METRICS_SUMMARY.html
```

Quick Note on QLM: The WLM configuration properties are either dynamic or static. Dynamic properties can be applied to the database without a cluster reboot, but static properties require a cluster reboot for changes to take effect. Additional info here:

```
https://docs.aws.amazon.com/redshift/latest/mgmt/workload-mgmt-config.html
```

### [Advanced Topic] Debug a Parquet/Redshift Spectrum datatype mismatch

1. Create a new Redshift Spectrum table, changing the datatype of column ‘trip_distance’ from FLOAT8 to FLOAT4. 
	* Add a single partition for testing.
2. Counts still work, but what about other operations (SELECT MIN(trip_distance) FROM, SELECT * FROM, CTAS)?
3. Instead of considering Apache Drill or other tool to help resolve the issue, consider Redshift system view SVL_S3LOG  
 
<details><summary>Hint</summary>
<p>


```python
https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-troubleshooting.html#spectrum-troubleshooting-incompatible-data-format
```

</p>
</details>


## Workshop - Scenario #3: Create a Single Version of Truth

### Create a view 

Create a view that covers both the January, 2016 Green company DAS table with the historical data residing on S3 to make a single table exclusively for the Green data scientists. Use CTAS to create a table with data from January, 2016 for the Green company. Compare the runtime to populate this with the COPY runtime earlier.


<details><summary>Hint</summary>
<p>


```
CREATE TABLE workshop_das.taxi_201601 AS SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month = 1 AND type = 'green';
```

</p>
</details>

Note: What about column compression/encoding? Remember that on a CTAS, Amazon Redshift automatically assigns compression encoding as follows:

* Columns that are defined as sort keys are assigned RAW compression.
* Columns that are defined as BOOLEAN, REAL, or DOUBLE PRECISION data types are assigned RAW compression.
* All other columns are assigned LZO compression.

```
https://docs.aws.amazon.com/redshift/latest/dg/r_CTAS_usage_notes.html 

```
Here's the ANALYZE COMPRESSION output in case you want to use it:

![GitHub Logo](/images/analyze_compression.png)


### Complete populating the table 

Add to the January, 2016 table with an INSERT/SELECT statement for the other taxi companies.

<details><summary>Hint</summary>
<p>

```
INSERT INTO workshop_das.taxi_201601 (SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month = 1 AND type != 'green');

```

</p>
</details>

### Create a new Spectrum table 
Create a new Spectrum table **adb305.NYTaxiRides** (or simply drop the January, 2016 partitions).

<details><summary>Hint</summary>
<p>

```python
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='fhv'); 
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='green'); 
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='yellow'); 

```
</p>
</details>


### Create a view with no Schema Binding
Create a view **adb305_view_NYTaxiRides** from **workshop_das.taxi_201601** that allows seamless querying of the DAS and Spectrum data.

<details><summary>Hint</summary>
<p>

```python
CREATE VIEW adb305_view_NYTaxiRides AS
SELECT * FROM workshop_das.taxi_201601
UNION ALL 
SELECT * FROM adb305.NYTaxiRides
WITH NO SCHEMA BINDING
;

```

</p>
</details>

### Is it Surprising this is valid SQL?

- Note the use of the partition columns in the SELECT and WHERE clauses. Where were those columns in your Spectrum table definition?
- Note the filters being applied either at the partition or file levels in the Spectrum portion of the query (versus the Redshift DAS section).
- If you actually run the query (and not just generate the explain plan), does the runtime surprise you? Why or why not?

<pre><code>
EXPLAIN SELECT year, month, type, COUNT(*) FROM adb305_view_NYTaxiRides WHERE year = 2016 AND month IN (1) AND passenger_count = 4 GROUP BY 1,2,3 ORDER BY 1,2,3;
</code></pre>

<pre><code>
QUERY PLAN 
XN Merge  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090025653.20..1000090025653.21 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90025653.19..90025653.19 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=25608.12..90025653.17 rows=2 width=48)
                          ->  XN Append  (cost=25608.12..90025653.15 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=25608.12..25608.13 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=25608.12..25608.12 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..25292.49 rows=31563 width=18)
                                                  <b>Filter: ((passenger_count = 4) AND ("month" = 1) AND ("year" = 2016))</b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000045.00..90000045.02 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000045.00..90000045.01 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000035.00 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..15.00 rows=1 width=30)
                                                       <b> Filter: (("month" = 1) AND ("year" = 2016))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
</code></pre>

- Now include Spectrum data by adding a month whose data is in Spectrum

```
EXPLAIN SELECT year, month, type, COUNT(*) FROM adb305_view_NYTaxiRides WHERE year = 2016 AND month IN (1,2) AND passenger_count = 4 GROUP BY 1,2,3 ORDER BY 1,2,3;

```

<pre><code>
QUERY PLAN
XN Merge  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
  Merge Key: derived_col1, derived_col2, derived_col3
  ->  XN Network  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
        Send to leader
        ->  XN Sort  (cost=1000090029268.92..1000090029268.92 rows=2 width=48)
              Sort Key: derived_col1, derived_col2, derived_col3
              ->  XN HashAggregate  (cost=90029268.90..90029268.90 rows=2 width=48)
                    ->  XN Subquery Scan adb305_view_nytaxirides  (cost=29221.33..90029268.88 rows=2 width=48)
                          ->  XN Append  (cost=29221.33..90029268.86 rows=2 width=38)
                                ->  XN Subquery Scan "*SELECT* 1"  (cost=29221.33..29221.34 rows=1 width=18)
                                      ->  XN HashAggregate  (cost=29221.33..29221.33 rows=1 width=18)
                                            ->  XN Seq Scan on t201601_pqt  (cost=0.00..28905.70 rows=31563 width=18)
                                                 <b> Filter: ((passenger_count = 4) AND ("year" = 2016) AND (("month" = 1) OR ("month" = 2))) </b>
                                ->  XN Subquery Scan "*SELECT* 2"  (cost=90000047.50..90000047.52 rows=1 width=38)
                                      ->  XN HashAggregate  (cost=90000047.50..90000047.51 rows=1 width=38)
                                            ->  XN Partition Loop  (cost=90000000.00..90000037.50 rows=1000 width=38)
                                                  ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=30)
                                                       <b> Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))</b>
                                                  ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45000010.00 rows=1000 width=8)
                                                        ->  S3 Aggregate  (cost=45000000.00..45000000.00 rows=1000 width=0)
                                                              ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..37500000.00 rows=3000000000 width=0)
                                                                  <b> Filter: (passenger_count = 4)</b>
</code></pre>

<pre><code>
EXPLAIN SELECT <b>passenger_count</b>, COUNT(*) FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (1,2) GROUP BY 1 ORDER BY 1;
</code></pre>

<pre><code>
QUERY PLAN
XN Merge  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
  <b>Merge Key: nytaxirides.derived_col1</b>
  ->  XN Network  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
        Send to leader
        ->  XN Sort  (cost=1000090005026.64..1000090005027.14 rows=200 width=12)
              <b>Sort Key: nytaxirides.derived_col1</b>
              ->  XN HashAggregate  (cost=90005018.50..90005019.00 rows=200 width=12)
                    ->  XN Partition Loop  (cost=90000000.00..90004018.50 rows=200000 width=12)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=0)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=45000000.00..45002000.50 rows=200000 width=12)
                                <b> ->  S3 HashAggregate  (cost=45000000.00..45000000.50 rows=200000 width=4)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=4)
</code></pre>

<pre><code>
EXPLAIN SELECT <b>type</b>, COUNT(*) FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (1,2) GROUP BY 1 ORDER BY 1 ;
</code></pre>

<pre><code>
QUERY PLAN
XN Merge  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
  <b>Merge Key: nytaxirides."type"</b>
  ->  XN Network  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
        Send to leader
        ->  XN Sort  (cost=1000075000042.52..1000075000042.52 rows=1 width=30)
              <b>Sort Key: nytaxirides."type"</b>
              ->  XN HashAggregate  (cost=75000042.50..75000042.51 rows=1 width=30)
                    ->  XN Partition Loop  (cost=75000000.00..75000037.50 rows=1000 width=30)
                          ->  XN Seq Scan PartitionInfo of adb305.nytaxirides  (cost=0.00..17.50 rows=1 width=22)
                               Filter: (("year" = 2016) AND (("month" = 1) OR ("month" = 2)))
                          ->  XN S3 Query Scan nytaxirides  (cost=37500000.00..37500010.00 rows=1000 width=8)
                              <b>  ->  S3 Aggregate  (cost=37500000.00..37500000.00 rows=1000 width=0)</b>
                                      ->  S3 Seq Scan adb305.nytaxirides location:"s3://us-west-2.serverless-analytics/canonical/NY-Pub" format:PARQUET  (cost=0.00..30000000.00 rows=3000000000 width=0)
</code></pre>

## Workshop - Scenario #4: Plan for the Future

* Allow for trailing 5 quarters reporting by adding the Q4 2015 data to Redshift DAS:
	* Anticipating the we’ll want to ”age-off” the oldest quarter on a 3 month basis, architect your DAS table to make this easy to maintain and query.
	* Adjust your Redshift Spectrum table to exclude the Q4 2015 data.
* Develop and execute a plan to move the Q4 2015 data to S3.
	* What are the discrete steps to be performed?
	* What extra-Redshift functionality must be leverage as of Monday, November 27, 2018?
	* Simulating the extra-Redshift steps with the existing Parquet data, age-off the Q4 2015 data from Redshift DAS 	and perform any needed steps to maintain a single version of the truth.

* Several options to accomplish the goal

![GitHub Logo](/images/table_pop_strat.png)


* Anticipating that we’ll want to ”age-off” the oldest quarter on a 3 month basis, architect your DAS table to make this easy to maintain and query.

* How about something like this:

	<pre><code>
	CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
   SELECT * FROM workshop_das.taxi_201504 <b>Note how these are business quarters</b>
UNION ALL 
  SELECT * FROM workshop_das.taxi_201601
UNION ALL 
  SELECT * FROM workshop_das.taxi_201602
UNION ALL 
  SELECT * FROM workshop_das.taxi_201603
UNION ALL 
  SELECT * FROM workshop_das.taxi_201604
UNION ALL 
  SELECT * FROM adb305.NYTaxiRides
WITH NO SCHEMA BINDING;
	</code></pre>
	
* Or something like this? Bulk DELETE-s in Redshift are actually quite fast (with one-time single-digit minute time to VACUUM), so this is also a valid configuration as well:

	<pre><code>
	CREATE OR REPLACE VIEW adb305_view_NYTaxiRides AS
   SELECT * FROM workshop_das.taxi_current
UNION ALL 
  SELECT * FROM adb305.NYTaxiRides
WITH NO SCHEMA BINDING;
	</code></pre>
	
* Don’t forget a quick ANALYZE and VACUUM after completing either version.

* If needed, the Redshift DAS tables can also be populated from the Parquet data with COPY. Note: This will highlight a data design when we created the Parquet data

**COPY with Parquet doesn’t currently include a way to specify the partition columns as sources to populate the target Redshift DAS table. The current expectation is that since there’s no overhead (performance-wise) and little cost in also storing the partition data as actual columns on S3, customers will store the partition column data as well.**

* We’re going to show how to work with the scenario where this pattern wasn’t followed. Use the single table option for this example
	
	````
	CREATE TABLE workshop_das.taxi_current DISTSTYLE EVEN SORTKEY(year, month, type) AS SELECT * FROM adb305.NYTaxiRides WHERE 1 = 0;

	````

* And, create a helper table that doesn't include the partition columns from the Redshift Spectrum table.

	````
	CREATE TABLE workshop_das.taxi_loader AS SELECT vendorid, pickup_datetime, dropoff_datetime, ratecode, passenger_count, trip_distance, fare_amount, total_amount, payment_type FROM workshop_das.taxi_current WHERE 1 = 0;

	````

### Parquet copy continued

* The population could be scripted easily; there are also a few different patterns that could be followed, (this isn't the only one):
	- Start Green loop.
	- Q4 2015.

	<pre><code>	
	COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=10/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=11/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2015/month=12/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
-- All 2016:
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=1/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=2/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=3/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=4/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=5/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=6/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=7/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=8/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=9/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=10/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=11/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
COPY workshop_das.taxi_loader FROM 's3://us-west-2.serverless-analytics/canonical/NY-Pub/year=2016/month=12/type=green' IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/mySpectrumRole' FORMAT AS PARQUET;
	</code></pre>

	<pre><code>
	INSERT INTO workshop_das.taxi_current SELECT *, DATE_PART(year,pickup_datetime), DATE_PART(month,pickup_datetime), 'green' FROM workshop_das.taxi_loader;
	
	TRUNCATE workshop_das.taxi_loader;
	</code></pre>

- Similarly, start Yellow loop.

### Redshift Spectrum can, of course, also be used to populate the table(s).

<pre><code>
INSERT INTO  workshop_das.taxi_201601    SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (2,3); /* Need to complete the first quarter of 2016.*/
CREATE TABLE workshop_das.taxi_201602 AS SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (4,5,6);
CREATE TABLE workshop_das.taxi_201603 AS SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (7,8,9);
CREATE TABLE workshop_das.taxi_201604 AS SELECT * FROM adb305.NYTaxiRides WHERE year = 2016 AND month IN (10,11,12);
</code></pre>

### Adjust your Redshift Spectrum table to exclude the Q4 2015 data

**Note for the Redshift Editor users:** Adjust accordingly based on how many of the partitions you added above.

<pre><code>
WITH generate_smallint_series AS (select row_number() over () as n from workshop_das.green_201601_csv limit 65536)
, part_years AS (select n AS year_num from generate_smallint_series where n between 2015 and 2016)
, part_months AS (select n AS month_num from generate_smallint_series where n between 1 and 12)
, taxi_companies AS (SELECT 'fhv' taxi_vendor UNION ALL SELECT 'green' UNION ALL SELECT 'yellow')

SELECT 'ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=' || year_num || ', month=' || month_num || ', type=\'' || taxi_vendor || '\');'
FROM part_years, part_months, taxi_companies WHERE year_num = 2016 or (year_num = 2015 and month_num IN (10,11,12)) ORDER BY year_num, month_num;

Or

ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=10, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=10, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=10, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=11, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=11, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=11, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=12, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=12, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2015, month=12, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=1, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=2, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=2, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=2, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=3, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=3, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=3, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=4, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=4, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=4, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=5, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=5, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=5, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=6, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=6, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=6, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=7, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=7, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=7, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=8, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=8, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=8, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=9, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=9, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=9, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=10, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=10, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=10, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=11, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=11, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=11, type='green');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=12, type='yellow');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=12, type='fhv');
ALTER TABLE adb305.NYTaxiRides DROP PARTITION(year=2016, month=12, type='green');
</code></pre>

* Now, regardless of method, there’s a view covering the trailing 5 quarters in Redshift DAS, and all of time on Redshift Spectrum, completely transparent to users of the view. What would be the steps to “age-off” the Q4 2015 data?

* Put a copy of the data from Redshift DAS table to S3. Listen closely this week for a possible announcement around this step! What would be the command(s)?
	* UNLOAD to Parquet.
* Extend the Redshift Spectrum table to cover the Q4 2015 data with Redshift Spectrum.
	* ADD Partition.
* Remove the data from the Redshift DAS table:
	* Either DELETE or DROP TABLE (depending on the implementation).

**You have already done all of the steps in previous scenarios for this workshop. You have the toolset in your mind to do this!
**
