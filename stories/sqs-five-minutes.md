# The SQS message that came back five minutes later

At Expedia, I debugged a production bug where a message was processed and
then showed up again exactly five minutes later.

I found the problem through an error in Splunk. I followed that clue into the
code and configuration and found that the consumer was polling the right SQS
queue, but a misplaced environment variable sent its delete request to a
different queue. The request returned an error, so the original queue never
got the acknowledgment. It held the message for its five-minute visibility
timeout, then made it available to another worker. With the receive limit
set to 2, it eventually went to the dead-letter queue.

I fixed the queue configuration so the consumer acknowledged the message on
the same queue it had polled. After the fix, the message cleared on the first
successful attempt.
