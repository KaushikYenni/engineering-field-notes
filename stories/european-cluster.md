# A cluster problem in Europe

One of the hardest stretches of the migration was when a cluster in Europe
became unhealthy and affected our application.

The first clear signal was the cluster health dashboard: the deployment was
red and marked unhealthy. That made it clear the issue wasn't just our
migration process.

The problem wasn't in the migration code itself. The application couldn't
make reliable progress while its underlying cluster was unhealthy. I worked
with the platform team to investigate and get the infrastructure healthy
again. I think they may have restarted the cluster or increased its
resources, but I don't remember which.

I don't remember what made the cluster unhealthy or how we confirmed it was
ready afterward. What I do remember is having to distinguish an application
issue from a platform issue and keep working across teams while the migration
was blocked.
