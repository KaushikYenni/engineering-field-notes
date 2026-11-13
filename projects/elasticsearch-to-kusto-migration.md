# Moving a global data estate from Elasticsearch to Kusto

I worked on the migration path for a global move of historical and active
Insider Risk data from Elasticsearch to Kusto. We migrated about 200 TB and
approximately 500 billion records spanning up to five years. The migration
ran active-to-active, with no customer downtime or data loss, and the program
was associated with about $5 million in annual savings. Those are program
outcomes; my part was engineering the migration path and supporting its
rollout.

Some of the stories that stick with me are about losing pagination state when
a source node restarted, making failures visible before logs arrived, and
keeping an eye on the rollout:

- [When Elasticsearch forgot my scroll context](../stories/elasticsearch-scroll-context.md)
- [Making migration errors visible sooner](../stories/migration-error-visibility.md)
- [Watching the migration while it ran](../stories/watching-the-migration.md)
- [A cluster problem in Europe](../stories/european-cluster.md)
