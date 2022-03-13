# What Does It Mean for a Column to Be Indexed?

## Goal
The goal of this project is to understand behavior of scan types with and without a column is Indexed.

## Scan Types
All database operations are one of Create Read Update Delete(CRUD) operations. These operations require the database engine to read/write data from the disk, accessing one or many blocks/pages of the data from the disk requires scanning of the data to get the exact block(s). A scan is the operation we aim to optimize with an index. We will see 4 main types of scan

### Seq Scan
The database engine scans through all the data sequentially

### Parllel Seq Scan
The database scans through all the data in parallel with n number of workers (can be gotten using explain). This is not used always because the overhead of starting, managing and reading data from multiple workers is a lot, but used to improve the performance of queries that require scanning a large amount of data but only few of them satisfy the selection criteria.

### Index Scan
If your query has a filter on an indexed column, the database engine uses the B-tree index to get the data location and just reads that page (this is the fastest). Depending on the database, if the database engine determines your query will return > 5-10% of the entire data Index Scan will be skipped for a Seq Scan, this is because the database engine decides that overhead of getting the page location from B-tree index, then reading the page for multiple values of the indexed column is not worth it compared to a simple sequential scan of the entire data.

### Bitmap Index Scan
This is a combination of Index Scan and Seq Scan, this scan is used when the number of rows to be selected are too large for Index Scan and too low for Seq Scan.

## Setup
What We Need?
  1. docker to run postgres
  2. pgcli to connect to postgres instance
  3. user.csv file

Setup Docker with folder (with user.csv) as a volume and start postgres container

    docker run -d --name postgres -p 5432:5432 -e POSTGRES_USER=data_engineer \
    -e POSTGRES_PASSWORD=data_password -e POSTGRES_DB=indextest \
    -v <your-data-folder>:/data postgres

Start pgcli and connect to postgres instance

    pgcli -h localhost -p 5432 -U data_engineer indextest

    Note: type password when it prompts

Create user table

    CREATE SCHEMA bank;
    SET search_path TO bank,public;
    CREATE TABLE bank.user (
        user_id int,
        user_name varchar(50),
        score int,
        user_account_type char(1)
      );

Load given user.csv file into user table

    COPY bank.user FROM '/data/user.csv' DELIMITER ',' CSV HEADER;

## Explain
Explain command is used to explain how the database engine executes the given query


## Experiments
E1. Select query with filter on score without index

    explain select * from bank.user where score = 900001;

    Scan type: Parallel Seq Scan is used since no index and with very few rows for the given score
    Execution time: 0.055s


E2. Select query with range filter on score without index

    explain select * from bank.user where score > 900000;

    Scan type: Seq Scan is used since no index and need to query entire data SET
    Execution time: 0.354s


E3. Update query with filter on score without index

    explain update bank.user set score = 1000000 where score > 1000;

    Scan type: Seq Scan is used since no index and range filter on score to scan entire dataset
    Execution time: 2.447s


E4. Select query with filter on user_account_type without indexed

    explain select * from bank.user where user_account_type='S';

    Scan type: Seq Scan is used to scan entire dataset since no index
    Execution time: 0.830s


E5. Select query with filter on score with index

    CREATE INDEX score_idx ON bank.user (score);

    explain select * from bank.user where score =900001;

    Scan type: Index Scan which means it uses the b-tree index to get the reference to the page the required data lives in
    Execution time: 0.019s


E6. Select query with range filter on score with index

    explain select * from bank.user where score > 900000;

    Scan type: Bitmap Index Scan which means the no.of records that satisfy the condition is estimated to be too large
    to benefit from Index scan and is too small to warrant a full Seq Scan
    Execution time: 0.411s

    explain select * from bank.user where score > 600000;

    Scan type: Seq Scan is used even with index on score column as database engine estimated that cost of checking the
    index, then grabbing the data page and repeating this for each score > 600000 is more than a simple Seq Scan
    Execution time: 0.585s


E7. Update query on score with filter on score with index

    explain update bank.user set score = 100000 where score > 1000;

    Scan type: Bitmap Index Scan as the no.of records that satisfy the condition is estimated to be too large
    to benefit from Index scan and is too small to warrant a full Seq Scan
    Execution time: 4.239s


E8. Select query with filter on user_account_type with index

    CRATE INDEX user_actype_idx ON bank.user(user_account_type);

    explain select * from bank.user where user_account_type='S';

    Scan type: Bitmap Index Scan as the no.of records that satisfy the condition is estimated to be too large
    to benefit from Index scan and is too small to warrant a full Seq Scan
    Execution time: 0.660s


## Cardinality

Cardinality defines the uniqueness of the data field.
  1. ***High Cardinality*** field that has many unique values e.g. score
  2. ***Low Cardinality*** field that has few unique values e.g. user_account_type


## Summary on using an Index

**Pros**
  1. Query with filter on an indexed column is much faster than one without. See Exp#1 vs Exp#5

**Cons**
  1. Performance drops for insert/update/delete operations, since the B-tree also has to be managed
     Compare Exp#7 and Exp#3, udpate time 8.627s with index vs 2.447s without index
  2. Depending on flavor of SQL, create/update/delete operations can be locked while the column is being indexed
  3. An index may not be used always, depending on estimated result size
  4. The increase of performance of index for low cardinality column will be lower than that of a high cardinality one.
     In the above experiments, the reduction in latency by adding index on a low cardinality field user_account_type
     was 20% whereas the same for high cardinality field score was 60%
