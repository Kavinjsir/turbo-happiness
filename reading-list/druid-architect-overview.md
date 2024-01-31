
## External Deps
### 1. Deep storage

Druid uses deep storage to store any data that has been ingested into the system.
Deep storage is shared file storage accessible by every Druid server.
In a clustered deployment, this is typically a distributed object store like S3 or HDFS, or a network mounted filsystem.
In a single-server deployment, this is typically local disk.

Druid uses deep storage for the following purposes:
- To store all the data you ingest.
  Segments that get loaded onto Historical processes for low latency queries are also kept in deep storage for backup purposes.
  Additionally, segments that are only in deep storage can be used for queries from deep storage.
- As a way to transfer data in the background between Druid processes.
  Druid stores data in files called segments.

Historical processes cache data segments on local disk and serve queries from that cache as well as from an in-memory cache.
Segments on disk for Historical processes provide the low latency querying performance Druid is known for.

You can also query directly from deep storage.
When you query segments that exist only in deep storage, you trade some performance for the ability to query more of your data without necessarily having to scale your Historical processes.

When determing sizing for your storage, keep the following in mind:
- Deep storage needs to be able to hold all the data that you ingest into Druid.
- On disk storage for Historical processes need to be able to accommodate the data you want to load ontot them to run queries.
  The data on Historical processes should be data you access frequently and need to run low latency queries for.

Deep storage is an important part of Druid's elastic, fault-tolerant design.
Druid boostraps from deep storage even if every single data server is lost and re-provisioned.

#### Deep storage Details

Deep storage is where segmentsjkjk are stored.
It is a storage mechanism that Apache Druid does not provide.
This deep storage infrastructure defines the level of durability of your data.
As long as Druid processes can see this storage infrastructure and get at the segments stored on it, you will not lose data no matter how many Druid nodes you lose.
If segments disappear from this storage layer, then you will lose whatever data those segments represented.

In addition to being the backing store for segments, you can use query from deep storage and run queries against segments stored primarily in deep storage.
The load rules you configure determine whether segments exist primarily in deep storage or in a combination of deep storage and Historical processes.

Deep supports multiple options for deep storage, including blob storage from major cloud providers.

### 2. Metadata storage

The metadata storage holds various shared system metadata such as segment usage information and task information.
In a clustered deployment, this is typically a traditional RDBMS like PostgreSQL or MySQL.
In a single-server deployment, it is typically a locally-stored Apache Derby database.

#### Metadata storage Details

Apache Druid relies on an external dependency for metadata storage.
Druid uses the metadata store to house various metadata about the system, but not to store the actual data.
The metadata store retains all metadata essential for a Druid cluster to work.

The metadata store includes the following:
- Segments records
- Rule records
- Configuration records
- Task-related tables
- Audit records

Derby is the default metadata store for Druid, however, it is not suitable for production.
MySQL and PostgreSQL are more production suitable metadata stores.

`We also recommend you setup a high availability environment because there is no way to restore lost metadata.`

To avoid issues with upgrades that require schema changes to a large metadata table, consider a metadata store version that supports instant ADD COLUMN semantics.

**Segments table**

This is dictated by the `druid.metadata.storage.tables.segments` property.

This table stores metadata about the segments that should be available in the system.
(This set of segments is called "used segments" elsewhere in the documentation and throughout the project.)
The table is polled by the `Coordinator` to determin the set of segments that should be available for querying in the system.
The table has two main functional columns, the other columns are for indexing purposes.


Value 1 in the used column means that the segment should be "used" by the cluster (i.e., it should be loaded and available for requests).
Value 0 means that the segment should not be loaded into the cluster.
We do this as a means of unloading segments from the cluster without actually removing their metadata (which allows for simpler rolling back if that is ever an issue).
The used column has a corresponding used_status_last_updated column which denotes the time when the used status of the segment was last updated.
This information can be used by the Coordinator to determine if a segment is a candidate for deletion (if automated segment killing is enabled).

The payload column stores a JSON blob that has all of the metadata for the segment.
Some of the data in the payload column intentionally duplicates data from other columns in the segments table.
As an example, the payload column may take the following form:

```json
{
 "dataSource":"wikipedia",
 "interval":"2012-05-23T00:00:00.000Z/2012-05-24T00:00:00.000Z",
 "version":"2012-05-24T00:10:00.046Z",
 "loadSpec":{
    "type":"s3_zip",
    "bucket":"bucket_for_segment",
    "key":"path/to/segment/on/s3"
 },
 "dimensions":"comma-delimited-list-of-dimension-names",
 "metrics":"comma-delimited-list-of-metric-names",
 "shardSpec":{"type":"none"},
 "binaryVersion":9,
 "size":size_of_segment,
 "identifier":"wikipedia_2012-05-23T00:00:00.000Z_2012-05-24T00:00:00.000Z_2012-05-23T00:10:00.046Z"
}
```

**Rule table**
The rule table stores the various rules about where segments should land.
These rules are used by the `Coordinator` when making segment(re-)allocation decision about the cluster.

**Config table**
The config table stores runtime configuration objects.
We do not have many of these yet and we are not sure if we will keep this mechanism going foward,
but it is the beginnings of a method of changing some configuiration parameters across the cluster at runtime.

**Task-related tables**
Task-related tables are created and used by the `Overload` and `MiddleManager` when managing tasks.

**Audit table**
Theu audit table stores the audit history for configuration changes such as rule changes done by `Coordinator` and other config changes.

#### Metadata storage access

Only the following proccesses access the metadata storage:
1. indexing service processes(if any)
2. Realtime processes(if any)
3. Coordinator processes

Thus you need to give permissions(e.g. in AWS security groups) for only these machines to access the metadata storage.

### ZooKeeper

Used for internal service discovery, coordiantion, and leader election.
The operations that happen over ZK are:
1. `Coordinator` leader election
2. Segment "publishing" protocol from `Historical`
3. Segment load/drop protocol between `Coordinator` and `Historical`
4. `Overload` leader election
5. `Overload` and `MiddleManager` task management

**Coordinator Leader Election**: We use the Curator `LeaderLatch` recipe to perform leader election at path:
```
${druid.zk.paths.coordinatorPath}/_COORDINATOR
```

## Storage Design

### Datasources and segments

Druid data is stored in datasources, which are similar to tables in a traditional RDBMS.
Each datasource is partitioned by time and, optionally, further partitioned by other attributes.
Each time range is called a chunk (for example, a single day, if your datasource is partitioned by day).
Within a chunk, data is partitioned into one or more segments.
Each segment is a single file, typically comprising up to a few million rows of data.
Since segments are organized into time chunks, it's sometimes helpful to think of segments as living on a timeline like the following:

---------------            ---------------  ---------------                   ---------------
| 2000-01-01  |            | 2000-01-02  |  | 2000-01-02  |                   | 2000-01-03  |
| Partition 0 |            | Partition 0 |  | Partition 0 |                   | Partition 0 |
---------------            ---------------  ---------------                   ---------------
-----------------------|---------------------------------------------------|---------------------->
   Chunk 2000-01-01                      Chunk 2000-01-02                     Chunk 2000-01-03


A datasource may have anywhere from just a few segments, up to hundreds of thousands and even millions of segments.
Each segment is created by a MiddleManager as mutable and uncommitted.
Data is queryable as soon as it is added to an uncommitted segment.
The segment building process accelerates later queries by producing a data file that is compact and indexed:

- Conversion to columnar format
- Indexing with bitmap indexes
- Compression
    - Dictionary encoding with id storage minimization for String columns
    - Bitmap compression for bitmap indexes
    - Type-aware compression for all columns

Periodically, segments are committed and published to deep storage, become immutable, and move from MiddleManagers to the Historical processes.
An entry about the segment is also written to the metadata store.
This entry is a self-describing bit of metadata about the segment, including things like the schema of the segment, its size, and its location on deep storage.
These entries tell the Coordinator what data is available on the cluster.

### Indexing and handoff

Indexing is the mechanism by which new segments are created.
And handoff is the mechanism by which they are published and begin being served by Historical processes.
On the indexing side:
1. An indexing task starts running and building a new segment.
   It must determine the identifier of the segment before it starts building it.
   For a task that is appending (like a Kafka task, or an index task in append mode) this is done by calling an "allocate" API on the Overlord to potentially add a new partition to an existing set of segments.
   For a task that is overwriting (like a Hadoop task, or an index task not in append mode) this is done by locking an interval and creating a new version number and new set of segments.
2. If the indexing task is a realtime task (like a Kafka task) then the segment is immediately queryable at this point.
   It's available, but unpublished.
3. When the indexing task has finished reading data for the segment, it pushes it to deep storage and then publishes it by writing a record into the metadata store.
4. If the indexing task is a realtime task, then to ensure data is continuously available for queries, it waits for a Historical process to load the segment.
   If the indexing task is not a realtime task, it exits immediately.

On the Coordinator / Historical side:
1. The Coordinator polls the metadata store periodically (by default, every 1 minute) for newly published segments.
2. When the Coordinator finds a segment that is published and used, but unavailable, it chooses a Historical process to load that segment and instructs that Historical to do so.
3. The Historical loads the segment and begins serving it.
4. At this point, if the indexing task was waiting for handoff, it will exit.


### Segment identifiers

Segments all have a four-part identifier with the following components:

- Datasource name.
- Time interval (for the time chunk containing the segment; this corresponds to the segmentGranularity specified at ingestion time).
- Version number (generally an ISO8601 timestamp corresponding to when the segment set was first started).
- Partition number (an integer, unique within a datasource+interval+version; may not necessarily be contiguous).

For example, this is the identifier for a segment in
- datasource `clarity-cloud0`
- time chunk `2018-05-21T16:00:00.000Z/2018-05-21T17:00:00.000Z`
- version `2018-05-21T15:56:09.909Z`
- partition number 1:
```
clarity-cloud0_2018-05-21T16:00:00.000Z_2018-05-21T17:00:00.000Z_2018-05-21T15:56:09.909Z_1
```

Segments with partition number 0 (the first partition in a chunk) omit the partition number, like the following example, which is a segment in the same time chunk as the previous one, but with partition number 0 instead of 1:
```
clarity-cloud0_2018-05-21T16:00:00.000Z_2018-05-21T17:00:00.000Z_2018-05-21T15:56:09.909Z
```

### Segment versioning

You may be wondering what the "version number" described in the previous section is for.
Or, you might not be, in which case good for you and you can skip this section!

The version number provides a form of multi-version concurrency control (MVCC) to support batch-mode overwriting.
If all you ever do is append data, then there will be just a single version for each time chunk.
But when you overwrite data, Druid will seamlessly switch from querying the old version to instead query the new, updated versions.
Specifically, a new set of segments is created with the same datasource, same time interval, but a higher version number.
This is a signal to the rest of the Druid system that the older version should be removed from the cluster, and the new version should replace it.

The switch appears to happen instantaneously to a user,
because Druid handles this by first loading the new data (but not allowing it to be queried),
and then, as soon as the new data is all loaded, switching all new queries to use those new segments.
Then it drops the old segments a few minutes later.


### Segment lifecycle

Each segment has a lifecycle that involves the following three major areas:

1. `Metadata store`: Segment metadata (a small JSON payload generally no more than a few KB) is stored in the metadata store once a segment is done being constructed.
   The act of inserting a record for a segment into the metadata store is called **publishing**.
   These metadata records have a boolean flag named used, which controls whether the segment is intended to be queryable or not.
   Segments created by realtime tasks will be available before they are published, since they are only published when the segment is complete and will not accept any additional rows of data.
2. `Deep storage`: Segment data files are pushed to deep storage once a segment is done being constructed.
   This happens immediately before publishing metadata to the metadata store.
3. `Availability for querying`: Segments are available for querying on some Druid data server, like a realtime task, directly from deep storage, or a Historical process.

You can inspect the state of currently active segments using the Druid SQL sys.segments table. It includes the following flags:

- `is_published`: True if segment metadata has been published to the metadata store and `used` is true.
- `is_available`: True if the segment is currently available for querying, either on a realtime task or Historical process.
- `is_realtime`: True if the segment is only available on realtime tasks.
  For datasources that use realtime ingestion, this will generally start off true and then become `false` as the segment is published and handed off.
- `is_overshadowed`: True if the segment is published (with `used` set to true) and is fully overshadowed by some other published segments.
  Generally this is a transient state, and segments in this state will soon have their `used` flag automatically set to false.


### Availability and consistency

Druid has an architectural separation between ingestion and querying, as described above in Indexing and handoff.
This means that when understanding Druid's availability and consistency properties, we must look at each function separately.

On the ingestion side, Druid's primary ingestion methods are all pull-based and offer transactional guarantees.
This means that you are guaranteed that ingestion using these methods will publish in an all-or-nothing manner:
- Supervised "seekable-stream" ingestion methods like Kafka and Kinesis.
  With these methods, Druid commits stream offsets to its metadata store alongside segment metadata, in the same transaction.
  Note that ingestion of data that has not yet been published can be rolled back if ingestion tasks fail.
  In this case, partially-ingested data is discarded, and Druid will resume ingestion from the last committed set of stream offsets.
  This ensures exactly-once publishing behavior.
- Hadoop-based batch ingestion.
  Each task publishes all segment metadata in a single transaction.
- Native batch ingestion.
  In parallel mode, the supervisor task publishes all segment metadata in a single transaction after the subtasks are finished.
  In simple (single-task) mode, the single task publishes all segment metadata in a single transaction after it is complete.

Additionally, some ingestion methods offer an idempotency guarantee.
This means that repeated executions of the same ingestion will not cause duplicate data to be ingested:
- Supervised "seekable-stream" ingestion methods like Kafka and Kinesis are idempotent due to the fact that stream offsets and segment metadata are stored together and updated in lock-step.
- Hadoop-based batch ingestion is idempotent unless one of your input sources is the same Druid datasource that you are ingesting into.
  In this case, running the same task twice is non-idempotent, because you are adding to existing data instead of overwriting it.
- Native batch ingestion is idempotent unless appendToExisting is true, or one of your input sources is the same Druid datasource that you are ingesting into.
  In either of these two cases, running the same task twice is non-idempotent, because you are adding to existing data instead of overwriting it.


On the query side, the Druid Broker is responsible for ensuring that a consistent set of segments is involved in a given query.
It selects the appropriate set of segment versions to use when the query starts based on what is currently available.
This is supported by atomic replacement, a feature that ensures that from a user's perspective, queries flip instantaneously from an older version of data to a newer set of data, with no consistency or performance impact.
This is used for Hadoop-based batch ingestion, native batch ingestion when appendToExisting is false, and compaction.


Note that atomic replacement happens for each time chunk individually.
If a batch ingestion task or compaction involves multiple time chunks, then each time chunk will undergo atomic replacement soon after the task finishes, but the replacements will not all happen simultaneously.


Typically, atomic replacement in Druid is based on a core set concept that works in conjunction with segment versions.
When a time chunk is overwritten, a new core set of segments is created with a higher version number.
The core set must all be available before the Broker will use them instead of the older set.
There can also only be one core set per version per time chunk. Druid will also only use a single version at a time per time chunk.
Together, these properties provide Druid's atomic replacement guarantees.


Druid also supports an experimental segment locking mode that is activated by setting forceTimeChunkLock to false in the context of an ingestion task.
In this case, Druid creates an atomic update group using the existing version for the time chunk, instead of creating a new core set with a new version number.
There can be multiple atomic update groups with the same version number per time chunk.
Each one replaces a specific set of earlier segments in the same time chunk and with the same version number.
Druid will query the latest one that is fully available.
This is a more powerful version of the core set concept, because it enables atomically replacing a subset of data for a time chunk, as well as doing atomic replacement and appending simultaneously.


If segments become unavailable due to multiple Historicals going offline simultaneously (beyond your replication factor), then Druid queries will include only the segments that are still available.
In the background, Druid will reload these unavailable segments on other Historicals as quickly as possible, at which point they will be included in queries again.



### Query processing

Queries are distributed across the Druid cluster, and managed by a Broker.
Queries first enter the Broker, which identifies the segments with data that may pertain to that query.
The list of segments is always pruned by time, and may also be pruned by other attributes depending on how your datasource is partitioned.
The Broker will then identify which Historicals and MiddleManagers are serving those segments and distributes a rewritten subquery to each of those processes.
The Historical/MiddleManager processes execute each subquery and return results to the Broker.
The Broker merges the partial results to get the final answer, which it returns to the original caller.

Time and attribute pruning is an important way that Druid limits the amount of data that must be scanned for each query, but it is not the only way.
For filters at a more granular level than what the Broker can use for pruning, indexing structures inside each segment allow Historicals to figure out which (if any) rows match the filter set before looking at any row of data.
Once a Historical knows which rows match a particular query, it only accesses the specific rows and columns it needs for that query.

So Druid uses three different techniques to maximize query performance:

- Pruning the set of segments accessed for a query.
- Within each segment, using indexes to identify which rows must be accessed.
- Within each segment, only reading the specific rows and columns that are relevant to a particular query.

