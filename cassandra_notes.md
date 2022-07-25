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
- Primary Key - Usually Partition Key is a combination pf Partition Key and Clustering Colum for Unquiness
- Partitions 
     Single Row Partition - Unique value like UUID
     Multiple Row Partition - Multiple columns share the partition Eg: PRIMARY KEY((venue, year), artifact)
 
Number of partitions does not decrease performance what does is the number of rows in these partitions and the amount of data
 
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
 
 
## Collections :
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
 
Use of FROZEN in collection
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
 
 ## Counters
 - 64 bit signed integer, can be incremented or decremented
 - use UPDATE to change values
 - Need special tables with Primary Key and counter columns
 
### SQLs: 
1. create TABLE PIZZA_DISPATCHED(store_name text, pizza_count, counter, PRIMARY KEY(store_name));
2. update PIZZA_DISPATCHED set pizza_count = pizza_count +1 where store_name = 'Dominos'
 
