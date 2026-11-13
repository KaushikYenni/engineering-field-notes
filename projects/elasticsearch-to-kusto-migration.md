# Moving a global data estate from Elasticsearch to Kusto

I worked on the migration path for a global move of historical and active
Insider Risk data from Elasticsearch to Kusto. The program migrated about
200 TB and approximately 500 billion records spanning up to five years. It
ran active-to-active, with no customer downtime or data loss, and was
associated with about $5 million in annual savings. I owned the
Elasticsearch-to-ADLS download flow and supported its rollout.

The download job read Elasticsearch in pages and wrote Parquet files to
ADLS. A Spark stage transformed those files before they were ingested into
Kusto. Keeping those stages separate made it possible to diagnose and rerun
the stage that had failed, rather than send every correction back to
Elasticsearch. The source data was organized into time-based indices, and
the job checkpointed completed indices so a long migration could move
forward in manageable units.

My part was the extraction and its operation: making the reader resilient to
node restarts, tuning parallel reads and storage writes, and following
progress across production deployments. A few stories from that work:

- [When Elasticsearch forgot my scroll context](../stories/elasticsearch-scroll-context.md)
- [Making migration errors visible sooner](../stories/migration-error-visibility.md)
- [Watching the migration while it ran](../stories/watching-the-migration.md)
- [A cluster problem in Europe](../stories/european-cluster.md)
