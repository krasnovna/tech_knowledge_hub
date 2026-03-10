## Module 1
#### Intro
ClickHouse: SQL Database 
OLAP (Online Analytical Processing ) 
Column-oriented 

#### Use cases 
1. Real-time Analytics 
2. Observability 
3. Data Warehousing ( Dashboarding )
4. ML & Gen AI 

CloudFlare case 
2018: 6M rows/sec -> 2025: 700M rows/sec 

#### Running ClickHouse
Cloud | Self-managed 
Cloud features:
* allowing certaing IPs to connect 
* it's possible to connect/select from S3 bucket data 
Using basic concole http://localhost:8123/play 
without using any JDBC drivers 

s3 VS s3Cluster ( allowed to parallel job across replicas)
any table has an ENGINE ( like MergeTree )

Playing with this dataset
https://clickhouse.com/docs/getting-started/example-datasets/uk-price-paid

in s3 there is a format 'CSVWithNames', that helps to assign headers as a first row 
otherwise it works wrong in Dbeaver with Docker version 

```sql
CREATE OR REPLACE TABLE uk_prices_temp 
ENGINE = Memory
AS 
    SELECT * 
      FROM s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/uk_property_prices/uk_prices.csv.zst',
      'CSVWithNames'
      )
     LIMIT 100;
```

Probably, in the cloud configs
this parameter 'input_format_with_names_use_header' is True 
that switch automatically to 'CSVWithNames'


#### Other Connection 
* Using jdbc via Dbeaver 
* Connected via client from docker to cloud CH 

## Module 2

#### Handling Data 
SHOW DATABASES;
* Each table has an ENGINE 
Clickhouse can be used as a query engine for get Postgres Data for example 

### Data Storage 
SharedMergeTree ( Clickhouse cloud )
ReplicatedMergeTree ( on prem )

### Column-oriented architecture 
```sql 
select avg(price) from phones 
``` 
benefits fro compression as you store each columns separately 

### MergeTree 

makes data and insert in table 
use async_insert feature to perform in batches 
INSERT -> part (immutable folder ) 
part -> merged part ( you can't control it, it happens in the background )
(!) you should have enough resource for background merging (how to assess it ? )
max_bytes_to_merge_at_max_space_in_pool ( 150 GB default )
As long at it is 150 GB it stops to merge 
table system.parts  for each table has field 
name = 'all_i_j_k' where (i,j) show which parts where merged, k - number of merges 
or using partitions 

### Data Parts 
Order BY can extend primary KEY 
best practices for PK:
0. Choose those which are frequently queried 
1. low cardinality columns go first

### Working with indexes 
https://clickhouse.com/docs/guides/best-practices/sparse-primary-indexes
A little magic or prewhere statement, can't get reproducible results 
Playing with parameters 
SETTINGS optimize_move_to_prewhere = 0,
	use_query_cache = 0,
    use_query_condition_cache = 1;

* index_granularity = 8192 = pow(2,13)
* ClickHouse allows inserting multiple rows with identical primary key column values. I
* for each column there is a file *.bin storing on the disk
* Clickhouse always reading data by granules (not by rows )
* Primary index has one value per granule 
* Indexes are stored in uncompress primary.idx 

Copying index so we can select it 
( in the manual it was primary-hits_UserID_URL.idx -> primary.idx actually )
docker exec -u root ch-sandbox bash -c "cp /var/lib/clickhouse/store/156/15659656-1d46-44f1-8d28-c1fd407c2433/all_1_9_2/primary.idx /var/lib/clickhouse/user_files/primary_debug.idx"

* The primary index is used for selecting granules
* Clickhouse use binary search to find granules corresponding to primary key value in filter 
* first stage = granules selection 

**Mark files are used for locating granules**
for each column there is a mark file .mrk with stores locations of granules 
each .mrk file contains block_offset, granule_offset 
bloc_offset used to get addres og compressed_data -> uncompressed them and then 
used granule_offset to get final data for further manipulation 

* When filtering on second index -> it looks like full scan 
generic exclusion algo 

**Generic exclusion algo** 
algo work better when the preceeding index has lower cardinality 

**Options for creating additional primary indexes**

1. second table with other primary key
cons: inserting into 2 tables; two different objects to work later on
in this case for different filtering we use effective binary search to perfrom filtering  
2. materialized view with other primary key
3. projection - most transparent way 

**Ordering key columns efficiently**
the more similar the data is, the better the compression ratio is

**Identifying single rows efficiently**
* high cardinality leads to poor column compression 
Example: plaintext paste service https://pastila.nl 
fingerprint and hash as a possible solution for both compression rate and quick selection

## Module 3
### Options for Inserting Data 
* Upload file
* ClickPipes - Kafka, S3, GCS ... 
* Messaging services 
* Client application 

table funcions, table engines 

### ClickPipes 
We can use SETTINGS schema_inference_make_columns_nullable = 0  
to not define nullable as default 
* LowCardinality(String)
* S3Queue Engine  
* Kafka Integration 

ClickPipes is an easy way to get data just by using UI tool

### Lab 3.1
* Temperatures dataset 
This setting now by default set to 3 
it is more complex logic than simple binary 0/1 
3 means that CH automatically determine whether Nullable is needed for any column based on first 1000 rows and metadata 

### Lab 3.2 
* Issues with data: parsing strings, casting data types ... 

working with learn-clickhouse bucket 
https://us-east-2.console.aws.amazon.com/s3/buckets/learn-clickhouse?region=us-east-2&tab=objects

's3://learn-clickhouse/operating_budget.csv'
VS 'https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv'

second is better 
1. Explicitly defined bucket's region 
2. Alternative s3 storages GCS ot Minio works fine with https 

approved_amount is by default string -> so any aggregated operations are not working 

* Using funciotns toUInt32OrZero and formatReadableQuantity to find results 

```sql
SELECT toUInt32(toFloat64OrZero('132 '));
```
Very strict function event with spaces will return 0

inserting with additional fix 
```sql
insert into default.operating_budget
select c1 as fiscal_year,
       c2 as service,
       c3 as department,
       trim(splitByChar('(', c4::String)[1]) as pogram,
       c5 as description, 
       c6 as item_category, 
       toUInt32OrZero(c7) as approved_amount,
       toUInt32OrZero(c8) as recommended_amount,
       c9::Decimal(12,2) as actual_amount,
       c10 as fund,
       c11 as fund_type, 
       trimBoth(splitByChar('(', c4::String)[2],'\s)') as pogram_code

  from s3('https://learn-clickhouse.s3.us-east-2.amazonaws.com/operating_budget.csv', format = 'CSV')
  SETTINGS  format_csv_delimiter = '~',
  input_format_csv_skip_first_lines = 1 ;
  ```

  S3Queue format only streams data from s3 bucket 
  