# Watching the migration while it ran

Most days, I was the person watching the migration move through large
production deployments. I kept an eye on how many Elasticsearch indices
had been processed, records per index and per deployment, transfer speed,
throughput, and application errors.

There were long-running deployments where I needed to stay close to the
progress. Sometimes I chose the office gym instead of the home gym so I
could check the dashboard between sets and investigate an error as soon as
it appeared. It's a small detail, but I remember it because monitoring the
migration wasn't something I did only during a scheduled check.

Azure Storage Table heartbeats and job status helped me see whether work was
still moving. When the centralized logs were slow, the
[error-message field I added to the table](migration-error-visibility.md)
gave me another way to find out what had happened.

I remember the progress signals better than the exact rules for calling a
job stale or the full restart and recovery process.
