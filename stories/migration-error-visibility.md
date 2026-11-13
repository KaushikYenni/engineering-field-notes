# Making migration errors visible sooner

Kusto logs could take around ten minutes to show an application failure.
When I was trying to get a migration instance running again, waiting for the
logs made an already slow problem slower.

We already wrote job status, progress, and hourly heartbeats to Azure Storage
Tables. While debugging in lower environments, I added an error-message
column. When a job failed, I could look at the table and see the error almost
immediately. Some of the failures were .NET runtime issues, like a missing
DLL or dependency.

I used this approach hundreds of times in lower environments to find
failures, troubleshoot deployments, and get migration instances running
again. It was a small change, but it shortened the distance between "the job
stopped" and "I know why."

I don't know how much time it saved overall. I also don't remember exactly
which fields connected an error to a particular job.
