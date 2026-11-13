# Making migration errors visible sooner

This was a lower-environment debugging trick, separate from the production
dashboard I used to watch migration progress. In test and development
environments, centralized logs could take around ten minutes to show why an
instance had failed. That delay made an already slow deployment or restart
cycle slower.

The job already recorded its status, progress, and periodic heartbeat in
Azure Storage Tables. I added an error-message field to that record, so when
a job failed I could check the table instead of waiting for the full log
trail. The message often made the next step obvious; some failures, for
example, were .NET runtime problems such as a missing DLL or dependency.

I used this approach hundreds of times in lower environments to find
failures, troubleshoot deployments, and get instances running again. It was
a small change, but it shortened the distance between "the job stopped" and
"I know why."
