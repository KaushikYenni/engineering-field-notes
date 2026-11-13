# Keeping investigation data for as long as it matters

I designed and built a distributed service to keep activity data available
for people under active investigation. Normal-retention data lived in
time-based partitions; as it approached expiration, the service copied
eligible activity into a separate long-retention table. Data associated with
an active case could be kept for up to five years, without extending the
retention of everyone's activity.

The service used an upstream active-case signal populated by a recurring
Spark job to decide whose data needed that longer retention. The challenge
was making sure data for a long-running case didn't expire while still
letting ordinary data age out on schedule.
