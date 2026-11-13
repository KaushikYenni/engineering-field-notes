# When Elasticsearch forgot my scroll context

Large-index jobs in the migration would sometimes fail with HTTP 500 and
`search_context_missing`. I tried retries and tuning the batch size, but the
same failure kept coming back.

The logs and deployment timings were the clue. I saw the issue after
deployments or restarts on both sides: the application hosting the migration
and Elasticsearch itself. A restart could invalidate Elasticsearch's
server-side scroll context, leaving the scroll ID I had unusable. At this
scale, expecting a scroll to stay alive indefinitely wasn't realistic.

I proposed and implemented a pluggable `search_after` path. Instead of asking
Elasticsearch to remember a long-lived session, I could continue from the
last document's sort values: a combination of record ID and timestamp. That
let the application keep its place when something interrupted a scroll and
start again without depending on the same live server-side context.

I parallelized extraction across non-overlapping time ranges and used
adaptive batch sizes to handle service pressure. Within an application
instance, four threads read from Elasticsearch and twelve threads wrote to
storage, with a bounded queue between them. We usually ran three instances
per deployment, and six in a few less common ones.

Throughput was around 12–18 million records per hour per instance. Large
indices often held 300–400 million records and took one or two days to
process. For example, I remember splitting a month's work into roughly four
slices. The approach let the migration keep moving through restarts. Some
duplicates could still happen in edge cases; the downstream storage-to-Kusto
load path already had deduplication. That was existing pipeline behavior,
not something I built for this migration.
