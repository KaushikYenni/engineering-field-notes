# Watching the migration while it ran

I was accountable for the Elasticsearch-to-ADLS download job and owned that
flow. That included keeping it moving across large production deployments
and watching how many Elasticsearch indices had been processed, records per
index and per deployment, transfer speed, throughput, and application
errors.

There were long-running deployments where I stayed close to the progress.
Sometimes I chose the office gym instead of the home gym so I could check the
dashboard between sets and investigate an error as soon as it appeared. It's
a small detail, but I remember it because owning the flow meant staying
connected to how it was running.

Azure Storage Table heartbeats and job status helped me see whether work was
still moving. When the centralized logs were slow, the
[error-message field I added to the table](migration-error-visibility.md)
gave me another way to find out what had happened.
