## Announcements

- We'll post solutions to HW5, but don't rely on them to do the peer review, because there are many possible valid solutions.
- I have a new SQLite database built for you.
    I'm in the process of zipping and transferring it.
    I plan to post it later today.
- Regarding grading- Homework 5 was harder, but worth less points on Canvas than the others.
    Homeworks 1-5 will all be worth the same amount, and Homework 6 will be worth 1/4 of that amount.
    We'll drop the lowest of homeworks 1-6.


## Project notes

- It can be frustrating and overwhelming to work with large data sets.
    They're rarely in the form you want, they're incomplete or have duplicates, and we often don't know exactly what things mean.
- You'll probably get going faster by starting small.
    Start by filtering the rows and columns to something more manageable than the entire data set.
- Remember to address specific, focused questions

Do the best with what you have.
When in doubt, use common sense and state your assumptions.
For example:

> We noted that the annual totals for `total_funding` in the `awards` table are less than the aggregate figures provided by the Congressional Budget Office (CBO).
> However, since 2008, the numbers consistently aggregate to about 80% of those of the CBO, so we've restricted our analysis to this time period.


## Reading

- [Documentation: MapReduce tutorial](https://hadoop.apache.org/docs/r1.2.1/mapred_tutorial.html)
- [Wikipedia: MapReduce](https://en.wikipedia.org/wiki/MapReduce)
- [2014 news: Google going beyond Map Reduce](https://www.datacenterknowledge.com/archives/2014/06/25/google-dumps-mapreduce-favor-new-hyper-scale-analytics-system)



## Review

A good analogy for performance improvement:
I filled up the disk on my local hard drive while doing this class.
To make more space I went through and deleted some of the largest files I had.
Tuning a program for performance is the same way- we want to eliminate the largest bottlenecks.


Here's some industrial grade SQL.
This is how the `universal_transaction_matview` table gets made, which is the same as 
I didn't write this query- rather, it's how the Postgres database was designed.
It's the materialized view of the data that powers most of the usaspending website.

A few interesting notes:
- `to_tsvector` are used to create a text searching vector
- `COALESCE` is a SQL function that returns the first non `NULL` argument
- `CASE WHEN` offers a simple form of conditional expression
- `obligation_to_enum` is a user defined function that does binning
- `transaction_normalized LEFT JOIN ...` does a left join on a whopping __19__ different tables.
    This essentially means taking the `transaction_normalized` table and adding much more information to it.

```{sql}
  SELECT to_tsvector(concat_ws(' '::text, COALESCE(recipient_lookup.recipient_name, transaction_fpds.awardee_or_recipient_legal, transaction_fabs.awardee_or_recipient_legal), transaction_fpds.naics, naics.description, psc.description, transaction_normalized.description)) AS keyword_ts_vector,
     to_tsvector(concat_ws(' '::text, awards.piid, awards.fain, awards.uri)) AS award_ts_vector,
     to_tsvector(COALESCE(recipient_lookup.recipient_name, transaction_fpds.awardee_or_recipient_legal, transaction_fabs.awardee_or_recipient_legal)) AS recipient_name_ts_vector,
     transaction_normalized.id AS transaction_id,
     transaction_normalized.action_date,
     transaction_normalized.last_modified_date,
     transaction_normalized.fiscal_year,
     transaction_normalized.type,
     transaction_normalized.action_type,
     transaction_normalized.award_id,
     awards.category AS award_category,
     (COALESCE(
         CASE
             WHEN (awards.category = 'loans'::text) THEN awards.total_subsidy_cost
             ELSE transaction_normalized.federal_action_obligation
         END, (0)::numeric))::numeric(23,2) AS generated_pragmatic_obligation,
     awards.total_obligation,
     awards.total_subsidy_cost,
     awards.total_loan_value,
     obligation_to_enum(awards.total_obligation) AS total_obl_bin,
     awards.fain,
     awards.uri,
     awards.piid,
     (COALESCE(transaction_normalized.federal_action_obligation, (0)::numeric))::numeric(20,2) AS federal_action_obligation,
     (COALESCE(transaction_normalized.original_loan_subsidy_cost, (0)::numeric))::numeric(20,2) AS original_loan_subsidy_cost,
     (COALESCE(transaction_normalized.face_value_loan_guarantee, (0)::numeric))::numeric(23,2) AS face_value_loan_guarantee,
     transaction_normalized.description AS transaction_description,
     transaction_normalized.modification_number,
     place_of_performance.location_country_code AS pop_country_code,
     place_of_performance.country_name AS pop_country_name,
     place_of_performance.state_code AS pop_state_code,
     place_of_performance.county_code AS pop_county_code,
     place_of_performance.county_name AS pop_county_name,
     place_of_performance.zip5 AS pop_zip5,
     place_of_performance.congressional_code AS pop_congressional_code,
         CASE
             WHEN (COALESCE(transaction_fpds.legal_entity_country_code, transaction_fabs.legal_entity_country_code) = 'UNITED STATES'::text) THEN 'USA'::text
             ELSE COALESCE(transaction_fpds.legal_entity_country_code, transaction_fabs.legal_entity_country_code)
         END AS recipient_location_country_code,
     COALESCE(transaction_fpds.legal_entity_country_name, transaction_fabs.legal_entity_country_name) AS recipient_location_country_name,
     COALESCE(transaction_fpds.legal_entity_state_code, transaction_fabs.legal_entity_state_code) AS recipient_location_state_code,
     COALESCE(transaction_fpds.legal_entity_county_code, transaction_fabs.legal_entity_county_code) AS recipient_location_county_code,
     COALESCE(transaction_fpds.legal_entity_county_name, transaction_fabs.legal_entity_county_name) AS recipient_location_county_name,
     COALESCE(transaction_fpds.legal_entity_congressional, transaction_fabs.legal_entity_congressional) AS recipient_location_congressional_code,
     COALESCE(transaction_fpds.legal_entity_zip5, transaction_fabs.legal_entity_zip5) AS recipient_location_zip5,
     transaction_fpds.naics AS naics_code,
     naics.description AS naics_description,
     transaction_fpds.product_or_service_code,
     psc.description AS product_or_service_description,
     transaction_fpds.pulled_from,
     transaction_fpds.type_of_contract_pricing,
     transaction_fpds.type_set_aside,
     transaction_fpds.extent_competed,
     transaction_fabs.cfda_number,
     references_cfda.program_title AS cfda_title,
     transaction_normalized.recipient_id,
     COALESCE(recipient_lookup.recipient_hash, (md5(upper(COALESCE(transaction_fpds.awardee_or_recipient_legal, transaction_fabs.awardee_or_recipient_legal))))::uuid) AS recipient_hash,
     upper(COALESCE(recipient_lookup.recipient_name, transaction_fpds.awardee_or_recipient_legal, transaction_fabs.awardee_or_recipient_legal)) AS recipient_name,
     COALESCE(transaction_fpds.awardee_or_recipient_uniqu, transaction_fabs.awardee_or_recipient_uniqu) AS recipient_unique_id,
     COALESCE(transaction_fpds.ultimate_parent_unique_ide, transaction_fabs.ultimate_parent_unique_ide) AS parent_recipient_unique_id,
     legal_entity.business_categories,
     transaction_normalized.awarding_agency_id,
     transaction_normalized.funding_agency_id,
     taa.name AS awarding_toptier_agency_name,
     tfa.name AS funding_toptier_agency_name,
     saa.name AS awarding_subtier_agency_name,
     sfa.name AS funding_subtier_agency_name,
     taa.abbreviation AS awarding_toptier_agency_abbreviation,
     tfa.abbreviation AS funding_toptier_agency_abbreviation,
     saa.abbreviation AS awarding_subtier_agency_abbreviation,
     sfa.abbreviation AS funding_subtier_agency_abbreviation
    FROM (((((((((((((((transaction_normalized
      LEFT JOIN transaction_fabs ON (((transaction_normalized.id = transaction_fabs.transaction_id) AND (transaction_normalized.is_fpds = false))))
      LEFT JOIN transaction_fpds ON (((transaction_normalized.id = transaction_fpds.transaction_id) AND (transaction_normalized.is_fpds = true))))
      LEFT JOIN references_cfda ON ((transaction_fabs.cfda_number = references_cfda.program_number)))
      LEFT JOIN legal_entity ON ((transaction_normalized.recipient_id = legal_entity.legal_entity_id)))
      LEFT JOIN ( SELECT rlv.recipient_hash,
             rlv.legal_business_name AS recipient_name,
             rlv.duns
            FROM recipient_lookup rlv) recipient_lookup ON (((recipient_lookup.duns = COALESCE(transaction_fpds.awardee_or_recipient_uniqu, transaction_fabs.awardee_or_recipient_uniqu)) AND (COALESCE(transaction_fpds.awardee_or_recipient_uniqu, transaction_fabs.awardee_or_recipient_uniqu) IS NOT NULL))))+
      LEFT JOIN awards ON ((transaction_normalized.award_id = awards.id)))
      LEFT JOIN references_location place_of_performance ON ((transaction_normalized.place_of_performance_id = place_of_performance.location_id)))
      LEFT JOIN agency aa ON ((transaction_normalized.awarding_agency_id = aa.id)))
      LEFT JOIN toptier_agency taa ON ((aa.toptier_agency_id = taa.toptier_agency_id)))
      LEFT JOIN subtier_agency saa ON ((aa.subtier_agency_id = saa.subtier_agency_id)))
      LEFT JOIN agency fa ON ((transaction_normalized.funding_agency_id = fa.id)))
      LEFT JOIN toptier_agency tfa ON ((fa.toptier_agency_id = tfa.toptier_agency_id)))
      LEFT JOIN subtier_agency sfa ON ((fa.subtier_agency_id = sfa.subtier_agency_id)))
      LEFT JOIN naics ON ((transaction_fpds.naics = naics.code)))
      LEFT JOIN psc ON ((transaction_fpds.product_or_service_code = (psc.code)::text)))
   WHERE (transaction_normalized.action_date >= '2000-10-01'::date)
   ORDER BY transaction_normalized.action_date DESC;
```


## Hadoop

Hadoop is a framework for distributed computing.
The idea is that we have many (1000's) of compute node working on the same problem.
The usaspending data we are working with now could possibly benefit from this.

The key features of Hadoop are

- __distributed file system__ HDFS stands for hadoop distributed file system.
    This acts as one conceptual file system, but the data is distributed across many nodes.
- __fault tolerant__ There is redundancy, so that if one node or hard drive fails then the computation will still proceed.
- __distributed sort__ This is central to Hadoop's model, see below.

Let's look at some distributed files:

```{bash}
clarkf@hadoop ~ $ hdfs dfs -ls
Found 11 items
drwx------   - clarkf hdfs          0 2017-11-21 16:00 .Trash
drwxr-xr-x   - clarkf hdfs          0 2017-06-21 16:25 .hiveJars
drwxr-xr-x   - clarkf hdfs          0 2017-10-27 15:30 fundamental_diagram
drwxr-xr-x   - clarkf hdfs          0 2017-06-22 17:17 pems
drwxr-xr-x   - clarkf hdfs          0 2017-07-20 11:37 pems_parquet
...
```

These are directories containing conceptual "chunks" of files.

The `pems` directories contains some large data files:

```{bash}
$ hdfs dfs -du -h pems
69.7 M   pems/d04_text_station_raw_2016_01_01.txt.gz
77.9 M   pems/d04_text_station_raw_2016_01_02.txt.gz
67.5 M   pems/d04_text_station_raw_2016_01_03.txt.gz
78.7 M   pems/d04_text_station_raw_2016_01_04.txt.gz
83.6 M   pems/d04_text_station_raw_2016_01_05.txt.gz
...
```

I can move files to and from HDFS.
For example, to extract one file from HDFS to the Hadoop head node:

```{bash}
hdfs dfs -get pems/d04_text_station_raw_2016_01_01.txt.gz
```

Now I see it locally.

```{bash}
$ du -h d04_text_station_raw_2016_01_01.txt.gz
70M     d04_text_station_raw_2016_01_01.txt.gz
```

I can peek inside it.
It's stored in a gzip archive.

```{bash}
$ gunzip -c d04_text_station_raw_2016_01_01.txt.gz | head
01/01/2016 00:00:05,402174,0,0,0,0,0,0,0,0,0,,,,,,,,,,,,,,,
01/01/2016 00:00:05,402175,0,0,0,0,0,0,0,0,0,,,,,,,,,,,,,,,
01/01/2016 00:00:05,402176,0,0,0,0,0,0,0,0,0,,,,,,,,,,,,,,,
01/01/2016 00:00:05,402177,0,0,0,0,0,0,0,0,0,,,,,,,,,,,,,,,
```

In the end it's just a text file, a CSV.
It started compressed and stored in HDFS.


## Hive

Often our data looks more like tables than key value pairs.
In this case we can use one of the many SQL over Hadoop offerings, for example Apache Hive.


The file `d04_text_station_raw_2016_01_01.txt.gz` is part of a bigger table called `pems` that I've stored in Hive.
We can write SQL on this table.
Hive will transform the SQL into a MapReduce job and show us the progress.

```{bash}

$ hive

Logging initialized using configuration in file:/etc/hive/2.5.0.0-1245/0/hive-log4j.properties
hive> SELECT COUNT(*) FROM pems;
Query ID = clarkf_20190312111038_22937094-d8f1-4496-893d-a127e149f88f
Total jobs = 1
Launching Job 1 out of 1


Status: Running (Executing on YARN cluster with App id application_1551998252961_0003)

--------------------------------------------------------------------------------
        VERTICES      STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED
--------------------------------------------------------------------------------
Map 1 ..........   SUCCEEDED    127        127        0        0       0       0
Reducer 2 ......   SUCCEEDED      1          1        0        0       0       0
--------------------------------------------------------------------------------
VERTICES: 02/02  [==========================>>] 100%  ELAPSED TIME: 47.01 s
--------------------------------------------------------------------------------
OK
2598111899
Time taken: 50.342 seconds, Fetched: 1 row(s)
```

There are 2.6 billion rows in this table, which is more than one order of magnitude larger than the usaspending data.



## Map Reduce

At the most basic level, Hadoop works on (key, value) pairs.

Hadoop does the following.

1. __Map__ a function over existing data.
    This is the same idea as `lapply, Map` in R and Python.
    The map step produces (key, value) pairs.
2. __Shuffle__ (distributed sort) puts all the same keys on the same node.
    This step can be quite time consuming.
3. __Reduce__ Once each node has all the values for a given key, it processes them with the reduce function.

It's up to the user to write the map and reduce steps, so they can be as simple or as complex as you like.

Canonical Example: wordcount program.


## Spark

Hadoop writes all intermediate results to disk.
This makes it slow.
Spark tries to keep things in memory, which makes it much faster.


## R functions

Here are a couple features of the R language that I haven't yet had a chance to show you.
This comes from the [database copying script](https://github.com/clarkfitzg/phd_research/blob/master/experiments/usaspending/postgres_to_sqlite.R#L11) I showed you last week.

- __lazy evaluation__ allows us to write flexible function interfaces, which means we can call them in many different ways.
- __ellipses__ The `...` is used to pass arguments through from a wrapper function without having to name them.
    Question: Why would we want to do this?
    Because then we can pass any arguments at all, without having to specify them.
- __`on.exit`__ From the documentation: _`on.exit` records the expression given as its argument as needing to be executed when the current function exits (either naturally or as the result of an error).
This is useful for resetting graphical parameters or performing other cleanup actions._


```{r}
# Copy a table between databases in chunks.
chunk_copy_table = function(from_name
    , to_name = from_name
    , qstring = paste0("SELECT * FROM ", from_name)
    , .from_db = from_db
    , .to_db = to_db
    , chunk_size = 1e5L
    , ...
    )
{
    result = dbSendQuery(.from_db, qstring, ...)
    on.exit(dbClearResult(result))

    # Remaining code omitted
}
```
