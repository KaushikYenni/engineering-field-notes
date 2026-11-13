# Keeping investigation data for as long as it matters

I designed, built, and operated a distributed service for the data lifecycle
work that came with moving activity data to Kusto. One retention policy
couldn't meet two different needs: most activity should age out on its normal
schedule, while activity associated with an active investigation might need
to remain available for years. Separately, when a tenant was offboarded or
hard-deleted, its activity had to be removed from both retention paths.

The service handled two long-running workflows: copying eligible activity
into longer retention, and processing tenant-deletion requests. Both depended
on durable state because Kusto operations can outlive a service process,
run, or deployment. The service had to know what work was submitted, what
had completed, what needed another attempt, and whether an operation was
still running on the primary or paired cluster.

## Two lifecycles, one service

Normal Activity Explorer data remained in its ordinary retention path.
Activity for people associated with active cases had a longer lifecycle—up
to five years—without extending the retention of everybody's data. An
upstream active-case signal, refreshed by a recurring Spark job, identified
which users were eligible for that extension.

The service also handled tenant deletion. A purge needed to cover both the
normal-retention data and the long-retention copy. Hiding rows from queries
wasn't enough for a hard-deletion requirement; Kusto's purge process was
needed to remove stored data artifacts.

Those operations had different triggers and failure modes, but shared the
same engineering concerns: durable progress, retry behavior, observability,
and data placement across clusters.

## Reindexing into long retention

The reindex workflow ran daily. It read the active-case user list and the
existing per-user state, selected work that needed to be done, submitted
Kusto copy operations, and tracked each operation until it reached a
terminal state. Successful users did not need to be recopied on every run;
failed, throttled, or incomplete work could resume later.

I kept state at the level of a user and operation because the workflow
couldn't be treated as one all-or-nothing batch. A service upgrade or
shutdown could happen while a Kusto operation was still running. In that
case, host cancellation was not the same as a failed copy: the service left
the work resumable instead of recording a false failure. A later run could
reconcile the operation and continue.

The daily run also had a time budget. That bounded how long the service
worked in one execution, but meant it had to choose fairly among new
backfills, incremental users, and operations that needed monitoring. At one
point, processing a large set of backfills first could use the budget before
incremental work was even scheduled. I changed the scheduling to interleave
those categories and use bounded parallelism, with metrics for work skipped
when the budget ran out.

As the feature matured, I added support for paired clusters, a Kusto
function as the source for eligible users, heartbeats for long-running
operations, bounded polling retries, and better handling of throttled
requests. Batch copy and controlled concurrency reduced the overhead of
submitting work user by user, while durable per-user state kept that
parallelism explainable and resumable.

## Purging offboarded tenants

Deletion requests arrived through Event Hub. Event delivery is at least
once, so the intake path could not assume every message would arrive
exactly once. A reader turned the request into durable state, deduplicating
against requests already recorded. The service then processed requests on a
daily cadence, grouping tenant IDs into bounded batches rather than issuing
an unbounded number of individual purge commands.

The Event Hub reader used managed identity and checkpointing so it could
resume after a process restart without losing its place. When its connection
needed rotation, the reader started a replacement before swapping over and
stopping the old one. That avoided a gap in which a tenant-deletion event
could be missed during credential or connection refresh.

For each tenant batch, the purge coordinator submitted work for both the
normal and long-retention data, recorded the Kusto operation IDs, and
monitored the results. Submitting a purge command was not the success
boundary: Kusto purge is asynchronous. A request was complete only when all
the required operations reached terminal success. Failed, timed-out, and
still-running work remained visible in state for retry or follow-up, with
metrics and alerts for repeated failures.

I also handled the primary/pair topology. The same tenant could need work
on both clusters, but the operation IDs, state, retries, and completion
signals had to remain separate. Success on one cluster could not hide a
failure or unfinished purge on the other. The Event Hub intake could create
the corresponding requests for both sides, while the cluster-specific
instances executed and monitored their own work.

## Production behavior changed the design

Following Kusto operation IDs was important when the service's own state
appeared to say a task had timed out. In one investigation, the Kusto
operations had actually succeeded; the daily budget had expired before the
service reconciled their results, leaving stale timeout state. I traced the
operation IDs and separated a successful-but-unreconciled operation from a
real Kusto failure or a true budget timeout. That distinction led to clearer
state handling rather than blindly resubmitting successful work.

Throttling provided a different lesson. Some reindex operations were
repeatedly consuming the daily budget under Kusto capacity pressure. The
operation status showed that the problem was throttling, not a bad query.
I used the timeout metrics to identify affected deployments, patched
instances where normal rollout was lagging, and validated the newer
behavior. Bounded submission, backoff, batch sizing, and durable
reconciliation were more useful than retrying the same work at full speed.

These were not just monitoring refinements. They changed whether the service
could distinguish work that failed from work that was still progressing,
and whether a retry would help or make the capacity problem worse.

## Placement and scope

The lifecycle service followed the data's placement: primary and paired
clusters had to receive consistent reindex and purge work. Tenant relocation
was a related but distinct responsibility. The wider platform decided when
a tenant moved between forests; my service needed to keep its lifecycle
operations and cluster selection consistent with that placement. It did not
own the entire relocation workflow.

I also avoided extending long retention to every activity record. Only
activity associated with active cases needed the longer window; applying it
to all data would increase storage and operational cost. Similarly, purge
requests were batched rather than submitted immediately one by one because
Kusto purge is expensive and serialized at the cluster level.

## How the service evolved

I created the service during the ES-to-Kusto migration, when Kusto needed
explicit handling for active-case retention and tenant deletion. It first
lived alongside the older data-platform code. I built the initial reindex
and purge flows, then added paired-cluster support, Event Hub intake,
metrics, heartbeat and retry behavior as production needs became clearer.

I later moved the service into the common data-platform codebase, with its
own .NET service, tests, Service Fabric package, and deployment
configuration. The later move toward ARM-managed deployment reduced the
legacy Service Fabric resources. Afterward, I helped remove the old
applications and stale migration resources rather than leaving temporary
infrastructure behind.

The work continued beyond code delivery: I investigated production
operations, added metrics for reindex and purge behavior, documented the
architecture and operating procedures, and handed off the service's
lifecycle and monitoring model. The main changes came in stages:

- **September 2025:** I helped shape the lifecycle design alongside the
  ES-to-Kusto work, including daily purge requests, durable request state,
  active-case eligibility, and per-tenant grouping.
- **December 2025–January 2026:** I built the initial service and then
  designed and implemented separate primary and paired-cluster execution
  with cluster-aware state.
- **February 2026:** I completed Event Hub request intake, traced stale
  timeout state back to successful Kusto operations, addressed throttling,
  and added clearer purge and intake metrics.
- **March–April 2026:** I moved the service into the common data-platform
  codebase and changed scheduling so backfills, incremental users, and
  operation monitoring shared the daily runtime budget more fairly.
- **Spring–July 2026:** I improved copy throughput with batching and
  controlled parallelism, moved toward ARM-managed deployment, distinguished
  host cancellation from real timeouts, removed stale resources, and
  completed the architecture and operations handoff.

## What I learned

Retention is not just a setting on a table when different users have
different needs. A long-running lifecycle process needs durable state beyond
a command ID; cancellation must not be mistaken for failure; and the copy
and deletion paths have to follow the same cluster topology as the data.
Most importantly, a submitted operation is not a completed operation. The
service had to keep checking, reconciling, and recording the outcome before
it could say the data had been reindexed or purged.

This project grew out of the larger
[Elasticsearch-to-Kusto migration](elasticsearch-to-kusto-migration.md),
but the lifecycle service remained responsible for retention and deletion
after the historical migration itself was complete.
