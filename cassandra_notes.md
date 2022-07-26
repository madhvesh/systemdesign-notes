# Cassandra Notes

Cassandra is a columear database which can be configured not have a single point of Failure as it replicates data on multiple nodes. 
It is also Elastically scalable.

Referencing CAP theorem, Cassandra is AP database. Cassandra can be made to improve consistency by tunable consistency. 
In tunable consistency can decide how many nodes need to agree for data consistency and this concept is called Quorum
 
Differences:

![image](https://user-images.githubusercontent.com/3052872/180290819-66b0d2ac-ef18-4c01-82fd-f9aac9ed9bb8.png)
 

* Reference from DataStax Docs
 
### Partition Key 
      Defines how data is stored in the node. Eg: Partition by Year
### Clustering Column 
      Defines the ordering of data in the partition. Eg: Order by Movies Names
### Keys and Partitions
- Primary Key: 
  - Usually Partition Key is a combination pf Partition Key and Clustering Colum for Unquiness
  - Has to be either Natural Keys and Surrogate Keys 
    - Natural keys: One of the attributes in the dataset
    - Surrogate Keys: keys for sole purpose of uniqueness, say UUID. Has no relationship with data
      - Conflict Free
      - Immutable, Uniform, Compactness, Performance

- Partitions
  - Single Row Partition - Unique value like UUID
  - Multiple Row Partition - Multiple columns share the partition Eg: PRIMARY KEY((venue, year), artifact)
 
Number of partitions does not decrease performance what does is the number of rows in these partitions and the amount of data. Some of the strategies for splitting partitions if it gets too large
   - Add another column this can be Natural or Surrogate the goal is have fewer rows per partition
   - Split using bucket, this enables the application to decide how many rows can be stored in the partition
   - Split the colums into smaller tables
 
 To the contrary, we can merge partitions if its too fragmented.
  - Introduce new partition key and nest objects into new partition
 
##### Important Concepts:
   - Data Storage
        Primary Keys define data uniqueness
        Partition Keys define data distribution
        Partition Key affect partition sizes(too big or too small)
        Clustering Keys define row ordering
 
   - Query:
        Primary keys define how data is retrieved
        Partition keys allow equality predicates(key == values)
        Clustering keys allow inequality predicates and ordering (key != values)
        Only one table per query, no joins
 
 
Cassandra orders columns by partition key, clustering columns (shown later), and then alphabetical order of the remaining columns
 
ALTER Table cannot alter PRIMARY KEYS
SOURCE - Execute a set of CQL commands
 
 
### Collections :
    Store Multivalued Data, Used for Small data
 
- Set 
    Stores only unique values, stored unsorted, sorted while retrieving 
Ex: 
     Create table users(
              name text,
        emails set<text> )
Insert into users (name,emails) values('xyz',{ 'cxaa123@gmail.com','123@xyz.com'});
 
- List: 
   Stores any value including duplicates
      Freq_dest = ['SFO','LA','DEL']

- Map: Key Value pairs
       Todo={'1':'abc', '2':'def'}
 
Use of FROZEN keyword in collection
  1. Nest datatypes in collection
  2. Will serialize multiple components in a single value
  3. FROZEN collection are treated as BLOBs
  4. Non-frozen allow updates to individual fields
 
User Defined Types(UDTs)
  Used to group related information
      1. Can have multiple data fields, each named and Type to a singlr column
      2. Can be any datatype including other UDTs
      3. Allows more complex embedding of data with a single column
 Example UDTs
 - create TYPE full_name (first_name text, last_name text);
 - create TYPE address (street text, city text, zip number);

 create TABLE users(id uuid, name frozen <full_name>, direct_reports set<frozen <full_name>>, addresses map<text,frozen <address> >,
                    PRIMARY KEY(id)); 

### Static columns
It is a special column that is shared by all the rows of a partition. the static column is very useful when we want to share a column with a single value. 
 
 ## Counters
 - 64 bit signed integer, can be incremented or decremented
 - use UPDATE to change values, no-  using timestamp or update ttl
 - Need special tables with Primary Key and counter columns
 - cannot assign default values
 - counter operations internally need read before write
 
### SQLs: 
1. create TABLE PIZZA_DISPATCHED(store_name text, pizza_count, counter, PRIMARY KEY(store_name));
2. update PIZZA_DISPATCHED set pizza_count = pizza_count +1 where store_name = 'Dominos'
 
#### Data Consistency with Batches
 - Log Batches aid with Data Consistency during multiple INSERTs, UPDATES and DELETEs
 - Log Batches combines all the writes (operations) in the batch and sends them at once to coordinator node rather than exectuting each operations.Once
 batch is accepted DSC manages the update. The coordinator saves the batch to a special log on corodinator and replica. If the execution fails it will be 
 replayed. The Batch succeeds once all the writes succeeds
 - Syntax:
     BEGIN BATCH
      INSERT into table1 ...
      INSERT into table2 ...
     APPLY BATCH
 
 - Not a good way to bulk load data
 - All writes have the same timestamp. There is no concept of ordering.
 
 ## Lightweight Transactions
    - Compare and Set (CAS) operation with ACID properties
    - Esentially an ACID operation at Partition Level
    - More expensive than regular reads/writes
 Example: 
 INSERT into users(user_id...) values ('maddy'...) if not exists
 
 Use Case:
   Reset token for password without CAS
 
   update users set
     reset_token = '12345' where user_id='maddy'
    
 Set the token to null with the last updated token and set the new password provided by user
 
   update users set
     reset_token = 'null',password='dull' 
     where user_id='maddy'
     IF reset_token = '12345'
    
### Secondary Index
 1. For column lookups which is not part of the Primary Key
 2. Cannot be used for counters and static columns
 3. Creates additional data structure in nodes
    - Each local node will have its own Index of the data it stores. So a query on secondary Index will need to go to all the nodes and pickup data from
      local Index
    - Better design is to provide partition key and then the Index so that search is efficient.
 
 #### When to use secondary Index
  1. Low cardinality columns (unique values for the columns)*
  2. Small Datasets or when in Prototype
  3. Search on both partition key and indexed column in a large partition 
 
  #### When not to use secondary Index
  1. High cardinality columns*
  2. Columns that use counter
  3. Frequently updated or deleted columns
 
 *If the cardinality is low then number of unique columns returned is small so its worth doing an Index as large data is retreived. If its fewer rows
  then we need to search all the nodes for less data
 
 ## Materialized View
 Materialized view is a separate table built from another table  data but a different primary Key. Materialized view are suitable for  
 high cardinality data.
  - The columns from the Primary key from the base table should be part of the primary key of Materialized view
  - Additional column which is usually the key to search on (also can be partition key) will be added.
  - Static columns are not allowed
 
Syntax:
 CREATE MATERIALIZED VIEW user_by_email AS SELECT first_name, last_name, email FROM users WHERE email is not null and user_id is not null
 PRIMARY KEY(email, user_id);
 
 *User_id is the primary key in Users table
 
AS SELECT  - Specifies Base table column
FROM - Name of BaseTable
 
- Cannot write to MatViews
- Mat Views are updated async when base table is updated. Since the partition col are different updates might get delayed
- Read repair to Base table will also result in repair to Mat View. If a read repair is performed to Mat View its not done for base table
 
### DATA AGGREGATION FUNCTIONS
 1. SUM - Total value from the column within the selected row
 2. AVG - Mean value of column within selected row
 3. COUNT - Count of all rows
 4. MIN - Min value of column within slected rows
 5. MAX - Max value of column within selected rows
 
### Cassandra Anti-Patterns:
 #### Query Specific:
     1. Queries that do a entire tablescan
     2. Layering IN clauses to get a particular result
     3. Doing a read before write excessively, as these are expensive. Use lightweight transactions as last resort (atleast 4 times slow than normal   
        write)
     4: Allow FILTERING to retreive data
 
 #### Table Specific
     1. Excessive creation of secondary index
     2. Use of non-frozen collection is a huge performance hit. If non-frozen is defined in UDTs make it frozen
     3. Use of string to represent dates and timestamps instead of time, timestamp or data
 
 #### Keyspace Level
     1. Not using TTLs or deletes can cause tombstones to build up
     2. If your read queries are timing out then consider modifying table structure
    
 
