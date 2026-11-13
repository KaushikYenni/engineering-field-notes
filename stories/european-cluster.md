# A cluster problem in Europe

One of the hardest stretches of the migration was when a cluster in Europe
became unhealthy and affected our application.

The first clear signal was the cluster health dashboard: the deployment was
red and marked unhealthy. That made it clear the issue wasn't just our
migration process.

The application couldn't make reliable progress while its underlying cluster
was unhealthy. I worked with the platform team to investigate and get the
infrastructure healthy again. I think they may have restarted the cluster or
increased its resources; I don't remember which.

The experience reinforced how important it is to distinguish an application
issue from a platform issue and keep working across teams while a migration
is blocked.
