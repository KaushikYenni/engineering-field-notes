# Moving a global data estate from Elasticsearch to Kusto

I owned the Elasticsearch-to-ADLS Download Job: the part of a larger
Elasticsearch (ES)-to-Kusto migration that extracted existing data and wrote
it to durable files for the downstream stages. The program moved about
200 TB and 500 billion records spanning up to five years. The source and
destination remained live during the work; the migration completed without
customer downtime or data loss and was associated with about $5 million in
annual savings.

This was not a one-time database export. Live ingestion continued while the
new Kusto path was being built and validated. The Download Job copied the
historical data already in ES, and later delta runs closed the gap as new
records continued to arrive. Kusto became the serving path only after the
data and customer-visible behavior had been checked.

## Why we moved

The existing ES dependency had become a security, lifecycle, cost, and
operational concern. Kusto was already used by other activity workloads, so
moving onto that platform reduced the need to maintain a separate search and
data-lifecycle stack.

The migration touched more than storage. It involved live ingestion,
schema mapping, query behavior, historical data, active-case retention,
tenant deletion, deployment, monitoring, validation, and eventually retiring
the old path. My direct ownership was the Download Job and its production
operation; the broader migration and cutover were shared work across the
teams responsible for those other pieces.

## The data path

During rollout, the existing and new ingestion and query paths ran side by
side. That let us compare results before moving customer traffic fully to
Kusto.

For existing data, the path was:

```text
Elasticsearch
    -> Download Job
    -> ADLS Parquet files
    -> Spark transformation
    -> Kusto ingestion
    -> Kusto queries
```

ADLS was a deliberate boundary between extraction and the downstream work.
It meant Spark or Kusto processing could be retried from the files without
reading the same historical data from ES again. It also made failures easier
to locate: a completed Download Job did not necessarily mean that the data
had made it through transformation, ingestion, and query validation.

The source was organized into monthly and yearly index families. The job
traversed those physical indices in order. Monthly data was split across ten
physical user-group indices; yearly data was processed as a yearly index.
The physical index was also the unit of durable progress: the job recorded a
completed index before advancing to the next one. For large deployments, we
divided the work into separate time ranges so several instances could
process independently.

The wider migration reused shared Kusto ingestion infrastructure. A
transformation mapped the flexible ES records into Kusto's typed schema,
while the existing and new query paths stayed available side by side for
comparison. Normal activity data followed its existing retention window;
records associated with active cases needed a separate path to longer
retention, and tenant deletion had to cover both. I describe that lifecycle
work in [the data-retention project](data-lifecycle-management.md).

## Building the Download Job

The first version established the basic job model: which source and time
range to process, how to walk monthly or yearly indices, how to retry
transient failures, how to generate Parquet, and where to record job state.
The early read path used a shared ES provider. It proved the complete route
from ES to ADLS, but only one producer advanced through the source pages.
Parallel ADLS writers helped on the output side, but the ES read itself was
too slow for the largest deployments.

I moved the reader to a lower-level ES client and added sliced scrolls so
multiple producers could fetch parts of an index concurrently. I separated
ES reads from ADLS writes with a bounded queue. In a typical instance, four
reader threads fed twelve storage writers. The queue let the two sides
overlap without allowing readers to accumulate unbounded data in memory;
when the writers fell behind, backpressure slowed the readers. A producer or
consumer failure failed the job rather than letting a partial index look
complete.

I added operational controls alongside the data path: durable job state in
an Azure Storage Table, pause and resume, automatic reclaim of abandoned
work, configurable concurrency and retry settings, and the ability to target
one index or a narrower time range. Per-index records captured
copied-record counts, duration, status, and errors. The job-level checkpoint
decided where a resume should continue; the per-index records gave me a way
to inspect and compare what had happened.

The durable checkpoint advanced only after an entire index completed. That
protected finished indices from being repeated automatically, but an
interrupted index could be replayed. I chose that simple, visible recovery
boundary rather than persisting a page cursor in durable state.

### Keeping pagination alive

Sliced scroll improved concurrency, but its cursor depended on a
server-side scroll context held by an ES node. In October 2025, large-index
runs began failing with errors such as `search_context_missing`, shard
failures, and HTTP 500 responses. The logs and restart timings pointed to ES
node restarts: retries could not recover a scroll context that had
disappeared.

Some large indices held 300–400 million records and took one or two days.
Because the checkpoint moved only after an index finished, losing a scroll
context late in that run could mean repeating much of the work. We tried
different slice counts, batch sizes, delays, retries, and diagnostics. Those
helped with pressure and investigation but could not reconstruct lost
server-side state.

I proposed and implemented a pluggable `search_after` reader. Each query
sorted on creation time and record ID; after processing a page, the next
query used the last record's sort values to continue. I split the requested
time range into non-overlapping windows and ran them in parallel. Each page
was a normal ES request, no longer dependent on a long-lived scroll context
or the node that created it.

The new path removed that particular failure mode, not every restart
problem. The page cursor still lived in memory, so a restart of the
application could replay the in-flight index from its last completed-index
checkpoint. The downstream ADLS-to-Kusto path already handled deduplication
for replayed records. `search_after` also did not create a snapshot of a
changing index; correctness depended on extracting bounded windows that
were stable enough for the duration of a run.

### Batching and safe ADLS writes

The reader adjusted batch sizes in response to success and pressure. It
started from a configured size, increased by 25 percent after three
consecutive successful requests, and halved after a size-related error or
repeated failures such as timeouts and payload limits. It stayed within
configured minimum and maximum bounds. This let us tune throughput without
assuming that a single batch size would work equally well across every ES
deployment.

The Download Job grouped records by creation-time hour and index type, then
wrote Snappy-compressed Parquet files to ADLS. Each file was validated before
the job treated the write as successful.
That mattered because a file that a .NET writer can produce is not
necessarily a file the downstream Spark/Java reader can consume; I checked
the actual boundary rather than stopping at “the upload succeeded.”

Large-scale testing exposed a separate write race. Concurrent jobs could
target the same output partition, and two writers could choose the same next
file sequence. The write path used non-overwriting uploads; when it detected
a collision, it refreshed the existing file sequence and retried instead
of overwriting another writer's output.

## Running it in production

The Download Job was isolated in its own Service Fabric deployment so a
resource-intensive migration would not compete with the existing migrator
work. Large deployments usually ran three instances over separate time
ranges; six were used in a few cases. At the throughput I observed after
tuning, an instance handled roughly 12–18 million records per hour. Those
rates varied with index size, source pressure, batch size, parallelism, and
replay, so I paid more attention to completed and validated output than to a
single peak rate.

I watched production progress in dashboards: completed indices, records
read and saved, files written, transfer rate, and errors. Breaking down time
spent fetching, parsing, partitioning, and uploading helped distinguish a
slow but active job from one that had stopped or was blocked at a particular
stage. I also used the job's per-index state and the migration tracker to
decide what to investigate or rerun. [I wrote more about watching those
runs](../stories/watching-the-migration.md).

Long-running work also changed how we handled deployments. A routine
Service Fabric upgrade could interrupt a job expected to run for days or
weeks, so the Download Job was kept out of the normal upgrade path while
migration was active. Elasticsearch restarts and application restarts had
different effects: `search_after` addressed the lost ES scroll context, but
an application restart could still replay the current index from its last
durable checkpoint.

The day-to-day loop was to review forest and index progress, locate the
failing stage, patch or deploy a fix, trigger a scoped index or time-window
rerun, compare counts and downstream results, then update the tracker. Some
deployments required a separate access and escort process, particularly in
government environments. There were also platform transitions and capacity
issues to work through; increasing ES concurrency was not the answer when
the bottleneck was Kusto, Spark, ADF, or another downstream dependency.

One useful distinction was between watching a live production rollout and
debugging a lower environment. Production progress came from dashboards. In
lower environments, centralized logs could take around ten minutes to show
an error, so I added an error-message field to an existing Azure Storage
Table job record and used it to get a faster explanation when an instance
failed. [That small debugging change has its own story](../stories/migration-error-visibility.md).

## Parity, delta, and cutover

Copying the bytes was only one part of the migration. We checked column
availability, record counts, field values, missing records, and query
behavior. Matching row counts was not enough: mapping and representation
differences could still change what a customer saw.

When a comparison found a mismatch, we first worked out where it came from:

- If extraction from ES was incomplete, rerun the affected index or time
  window through the Download Job.
- If ADLS was correct but the transformation needed a fix, replay from ADLS
  rather than rereading ES.
- If the issue was in Kusto ingestion, a query, or a mapping, fix that stage
  and validate it again.

Meanwhile, live ingestion continued. The initial historical copy covered a
bounded window; delta jobs picked up records written after that window.
Each deployment moved toward cutover only after historical data, delta
coverage, live ingestion, and query parity lined up. The tracker and
dashboards were operational tools for deciding what was ready, what needed
another pass, and what still required verification.

## Timeline and outcome

- **July–September 2025:** I built the Download Job foundation, including
  job state, index traversal, retries, metrics, Parquet output, and
  operational controls. We isolated it in a dedicated deployment and added
  per-index progress.
- **October 2025:** I scaled the ES read path with a low-level client,
  sliced scrolls, and a bounded producer/consumer pipeline. After tracing
  large-index failures to node-local scroll contexts, I implemented the
  `search_after` path and adaptive batching.
- **Late October–November 2025:** We began the broader production rollout,
  ran multiple deployments in parallel, and worked through the different
  deployment and access requirements for government environments.
- **November 2025–January 2026:** We continued delta runs, parity work,
  scoped retries, and downstream validation while resolving deployment,
  capacity, and cross-system issues.
- **January–February 2026:** Activity Explorer was reported migrated to
  Kusto; the old ES path was deprecated and cleanup followed in stages.

The program completed worldwide, including government environments, without
customer downtime or data loss. For my part, the lasting work was not only
getting data out of ES faster. It was making the Download Job resumable at a
useful boundary, observable while it ran, safe to replay, and practical to
operate across a long rollout.

## Related stories

- [When Elasticsearch forgot my scroll context](../stories/elasticsearch-scroll-context.md)
- [Making migration errors visible sooner](../stories/migration-error-visibility.md)
- [Watching the migration while it ran](../stories/watching-the-migration.md)
- [A cluster problem in Europe](../stories/european-cluster.md)
- [Keeping investigation data for as long as it matters](data-lifecycle-management.md)
