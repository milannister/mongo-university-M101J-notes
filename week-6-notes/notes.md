# Creating a Replica Set
create_replica_set.sh:
```bash
#!/usr/bin/env bash

mkdir -p /data/rs1 /data/rs2 /data/rs3
mongod --replSet m101 --logpath "1.log" --dbpath /data/rs1 --port 27017 --oplogSize 64 --fork --smallfiles
mongod --replSet m101 --logpath "2.log" --dbpath /data/rs2 --port 27018 --oplogSize 64 --smallfiles --fork
mongod --replSet m101 --logpath "3.log" --dbpath /data/rs3 --port 27019 --oplogSize 64 --smallfiles --fork
```

and to let them know about each other
from mongo shell
init_replica.js

```javascript
config = { _id: "m101", members:[
          { _id : 0, host : "localhost:27017", priority:0, slaveDelay:5},
          { _id : 1, host : "localhost:27018"},
          { _id : 2, host : "localhost:27019"} ]
};

rs.initiate(config);
rs.status();
```

```javascript
rs.slaveOk()
```
allows you to read from secondary node

` rs.status() ` - prints status of the replica set


# Replica Set Internals
* The `local` Database - Every mongod instance has its own local database, which stores data used in the replication process, and other instance-specific data.
* oplog - kept in sync (seconaries are constantly reading the oplog of the primary)
* write done on the primary -> insert statement is written down in the oplog of the primary
After write occurs, this is written down in oplog.rs
```javascript
{
	"ts" : Timestamp(1448718938, 1),
	"h" : NumberLong("-6979816546038143653"),
	"v" : 2,
	"op" : "i",
	"ns" : "test.people",
	"o" : {
		"_id" : ObjectId("5659b25ac9c4b3304464d00a"),
		"name" : "milan"
	}
}
```
` "op" : "i" ` - operation : insert

# Failover and Rollback


# Connecting to a Replica Set from the Java Driver
* just initialize MongoClient with the list of ServerAddresses with the addresses of the members of replica sets
```java
new MongoClient(asList(new ServerAddress("localhost", 27017),
                      (new ServerAddress("localhost", 27018),
                      (new ServerAddress("localhost", 27019)));
```
* one more level of protection is to specify replica set's name when initialize mongo client

# When Bad Things Happen to Good Nodes
* handle MongoException and continue with business as usual

# Write Concern Revisited

* write
* w, j
* wtimeout


# Read Preference

* Primary (default)
* PrimaryPrefered
* Secondary
* SecondaryPrefered
* Nearest (by ping time)

# Review of Implications of Replication

* Seed Lists
* Write Concern - w, j, wtimeout
* Read Preference
* Errors can happen

# Sharding

* shards - splits up the data from particular collection (shards are usually replica sets itself)

* mongos - router that takes care of routing the queries
* range based approach
* from 2.4 hash-based approach
  * more even distribution of data
  * worse performance for range-based queries
* shard key (part of the document that is used for deciding where the document will be stored/queried)

* If the shard key is not included in a find operation and there are 4 shards mongos has to send the query to all 4 of the shards.

# Building a Sharded Environment (create_scores.js, init_sharded_env.sh)

* config servers - holds info about the way the data is distributed accross the shards
* chunks - are mapped to
* shard key - must be mandatory property, index must be present on shard key

`sh.status()` - status of the sharded system

# Implications of Sharding

* every document must include the shard key
* shard key is immutable
* index that starts with the shard key
* shard key has to be specified or multi?
* no shard key => search all nodes
* no unique key - unless part of the shard key (or starts with the shard key), because document in a collection can be scathered accross several shards and there's no way to enforce uniquenes

# Sharding + Replication

* almost always done together (shards are replica sets)
* write concern (w,j, wtimeout) - is passed through mongos to shards and replica sets

# Choosing a Shard Key




















