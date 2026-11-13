# The SQS message that came back five minutes later

I remember a production bug where a message was processed and then showed up
again exactly five minutes later. I think this was at Expedia, but I want to
confirm the product area before I state that as fact.

I found the problem through an error in Splunk. I followed that clue into the
code and configuration and found that the consumer was polling the right SQS
queue, but a misplaced environment variable sent its delete request to a
different queue. The request returned an error. The original queue never got
the acknowledgment, so it held the message for its five-minute visibility
timeout and then made it available to another worker. With the receive limit
set to 2, it eventually went to the dead-letter queue.

I fixed the queue configuration so the consumer acknowledged the message on
the same queue it had polled. I don't remember how the code handled the
failed delete request or the deployment check afterward. I do remember the
message clearing on the first successful attempt after the configuration fix.

I didn't build the event-driven workflow from scratch. The part that was
mine was finding why a consumer that seemed to be working kept processing
the same message.
