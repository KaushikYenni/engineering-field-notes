# A cluster problem in Europe

One of the harder stretches of the migration came when a cluster in Europe
became unhealthy and our application stopped making reliable progress.

The first clear signal was the cluster-health dashboard: the cluster was
red, not just the migration job. That changed where I looked. Rather than
keep treating it as an extraction or application failure, I brought the
platform team into the investigation and followed the cluster's health with
them.

The application could not make progress until the underlying infrastructure
was healthy again. The useful part of the dashboard was not just that it
showed red; it gave us a shared signal to separate a platform problem from a
job-level one and coordinate the response.
