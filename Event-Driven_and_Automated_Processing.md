# Event-Driven and Automated Processing

## 22. You want Glue ETL to run automatically when new files arrive in S3. How would you design S3 event notifications to avoid duplicates/misses?

I design the flow knowing S3 events are “at-least-once” (duplicates can happen) and deliveries  can be delayed. So I never start Glue directly from S3. I put a reliable queue in the middle, make  © Shubham Wadekar
the job idempotent, and add a daily reconciliation.  
How I wire it  
• S3 sends only the events I care about (ObjectCreated:Put and  
ObjectCreated:CompleteMultipartUpload) into SQS. I use prefix/suffix filters so only the  
right folder and file types trigger it, for example raw/appX/dt=/ and *.json.gz.  
• A small Lambda reads from SQS in batches and starts or bookmarks the Glue job with  
the object key, ETag, and versionId.  
• If I expect many files per day, I do micro-batches: Lambda groups keys by partition (for  
example, same dt) and triggers Glue once per partition, passing a manifest list to Glue  
to process together.  
How I avoid duplicates  
• I make Glue idempotent: before writing processed data, it checks a DynamoDB table (or  a _SUCCESS marker) to see if this object/version was already processed. The  
idempotency key is bucket + key + versionId (or ETag).  
• Glue writes to a run-scoped temp path and promotes only on success. If  
the same file comes again, the job does nothing.  
• In Lambda, I also use a short-lived de-dup cache (for example, DynamoDB TTL of a few  
hours) to skip obvious repeats before starting Glue.  
How I avoid misses  
• I enable an SQS dead-letter queue (DLQ). If the Lambda can’t process a message after  
retries, it lands in DLQ for manual review.  
• I set SQS visibility timeout > Lambda’s max runtime so messages don’t reappear mid 
processing.  
• I add a daily “reconciliation” Glue job that reads the S3 listing (or S3 Inventory) for  
yesterday’s prefix and compares against the processed manifest. Any missing keys are  
queued again.  
Operational details  
• Increase Lambda memory to speed up S3 HEAD/manifest work, and set reserved  
concurrency so it scales but doesn’t start too many Glue runs at once. 
• Use EventBridge rules for additional routing if later I need to send a copy of the event to  
monitoring or a second pipeline.  
In simple terms: send S3 events to SQS, trigger Glue through Lambda, make the job  
idempotent, and run a daily catch-up so duplicates don’t hurt and misses are re-queued  
automatically.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 23. S3 → Lambda integration misses some large files. How would you troubleshoot and redesign it for both small and large files?

I check whether the event fired, whether Lambda received it, and whether processing failed due  to time or memory. Then I redesign to use SQS between S3 and Lambda, tune timeouts, and  © Shubham Wadekar
handle multi-part uploads properly.  
Troubleshooting steps  
• Verify event configuration: are we listening to ObjectCreated:CompleteMultipartUpload  as well as Put? Large files often use multi-part, so only “Put” may miss them. 
• Check filters: wrong prefix/suffix filters can exclude large files if they land in a different  
folder or extension.  
• Look at S3 object metadata: confirm the upload actually completed and isn’t a partial or  aborted upload.  
• Check CloudWatch logs and Lambda metrics: look for timeouts, out-of-memory, or  
throttling. Also check concurrent executions; throttling can drop events if not using a  
queue.  
• Confirm permissions: ensure Lambda’s role can GetObject on the source, and KMS  
decrypt if objects are SSE-KMS.  
Redesign for reliability and size  
• Put SQS in the middle: S3 → SQS (notifications) → Lambda (polls SQS). This gives retries,  buffering, DLQ, and removes the risk of direct-event throttling.  
• Include both ObjectCreated:Put and ObjectCreated:CompleteMultipartUpload in the  
S3 notification so large multi-part uploads trigger only after they fully complete.  
• Set SQS visibility timeout longer than Lambda’s max runtime. Add a DLQ for poison  
messages.  
• Increase Lambda timeout and memory so it can handle larger headers/manifests. If  
Lambda copies data, consider moving that work to Glue or Step Functions instead of  
doing heavy work inside Lambda.  
• If objects are extremely large, avoid reading them in Lambda. Have Lambda write a  
manifest entry, and let a Glue/Spark job process in parallel from S3.  
Performance and cost tips  
• Batch SQS messages (for example, 5–10 keys per batch) so Lambda fewer cold  
starts.  
• For very hot buckets, use Standard SQS (not FIFO) for higher throughput, and add  
idempotency in processing.  
• Add a periodic reconciliation using S3 Inventory to catch any stragglers. 
In simple terms: include the right event types for big files, put SQS between S3 and Lambda,  tune timeouts/memory, and move heavy processing to Glue so both small and large files are  handled reliably.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 24. You need to push S3 file metadata into SQS for real-time processing. How would you configure S3 event notifications for scale and reliability?

I configure S3 to send only the necessary events into SQS, design message shape clearly, and  add buffering, retries, and DLQ so it scales without losing messages.  
© Shubham Wadekar
Event configuration  
• Turn on notifications for ObjectCreated:Put and  
ObjectCreated:CompleteMultipartUpload. I usually skip Copy/Post unless needed.  
• Use prefix/suffix filters so only relevant folders/extensions generate messages, for  
example raw/appY/ and *.parquet.  
• If many teams need the events, I fan out via SNS (S3 → SNS → multiple SQS queues), or  
use EventBridge rules to route to multiple targets.  
SQS queue setup  
• Choose Standard SQS for very high throughput. If strict ordering and de-dup are  
required, use FIFO with content-based deduplication (throughput will be lower).  
• Set a visibility timeout comfortably larger than the consumer’s processing time. 
• Configure a DLQ with an appropriate maxReceiveCount (for example, 5). Alert on DLQ  
messages.  
Message content and idempotency  
• The S3 event already includes bucket, key, size, ETag, and (if versioning) versionId. I  
keep these as the idempotency key in my consumer so reprocessing the same object  
does nothing.  
• If I need extra metadata (tags, custom headers), the consumer can call HeadObject to  
enrich the message before downstream processing.  
Consumer design  
• Use an autoscaling consumer (Lambda with reserved concurrency, or a container  
service polling SQS) so it keeps up with spikes.  
• Process messages in small batches (for example, 5–10). For each message, validate the  object exists and is in a “ready” state. 
• On success, delete the message from SQS. On transient failure, let SQS retry; on  
repeated failure, the message lands in DLQ for manual action.  
Reliability and observability  
• Enable CloudWatch metrics on SQS (ApproximateNumberOfMessagesVisible,  
NotVisible, DLQ depth) and add alarms.  
• Add a daily reconciliation job using S3 Inventory: list objects created yesterday under  
the prefixes and compare with what was processed. Re-queue any gaps.  
• If encrypted with SSE-KMS, ensure S3 can publish to SQS (SQS policy) and the  
consumer has kms:Decrypt for the data it needs to read.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Cost and noise control  
• Keep filters tight so you don’t flood SQS with unneeded events. 
• For extremely chatty workloads, consider aggregating events: have S3 send to  
EventBridge, then a small rule batches keys into a manifest and pushes fewer, larger  
messages to SQS.  
© Shubham Wadekar
In simple terms: S3 sends only the right “object created” events to SQS, the consumer scales  and is idempotent, there’s a DLQ for failures, and a daily reconciliation catches anything that  slips through. This gives real-time, reliable metadata flow at scale.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar