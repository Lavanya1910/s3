# Storage Classes and Cost Optimization

## 4. You have 200 TB of logs in S3 Standard, but only the last 3 months are queried. How would you cut costs using storage classes?

I would keep the most recent logs in a fast class for analytics and shift older data to colder,  
cheaper classes automatically using lifecycle policies. I will also compress and organize the  © Shubham Wadekar
data so we pay less for both storage and retrieval.  
Classify access patterns first  
• Hot data: last 0–90 days, queried by analysts and jobs. Keep this quickly accessible.  
• Warm/cold data: 3–12 months, rarely queried. Keep it cheaper but still retrievable  
without long waits.  
• Archive: older than a year, kept mainly for compliance or audits.  
Choose storage classes by age  
• 0–30 days: S3 Standard for best performance (or S3 Intelligent-Tiering if access is  
unpredictable; it automatically moves objects between frequent and infrequent tiers).  
• 31–90 days: S3 Standard-IA to drop storage cost while still keeping milliseconds access.  Note the 30-day minimum storage charge, which fits this window.  
• 91–365 days: S3 Glacier Instant Retrieval if we occasionally need to query these logs  
with low latency; if access is extremely rare, S3 Glacier Flexible Retrieval is cheaper but  
has minutes-to-hours retrieval.  
• 365 days: S3 Glacier Deep Archive for the lowest cost, accepting hours-level retrieval  
time for rare audits.  
Implement with lifecycle rules  
• Create prefix-based rules per dataset, or use object tags like data_class=logs to target  
policies precisely.  
• Example rule: transition to Standard-IA at 30 days, Glacier Instant Retrieval at 90 days,  
Deep Archive at 365 days, and optionally delete after your retention period (for example,  
730 or 1825 days).  
• Apply rules only after confirming minimum storage duration for each class to avoid  
early-deletion fees.  
Keep retrieval costs predictable  
• For teams that occasionally need older data quickly, I prefer Glacier Instant Retrieval for  the 3–12 month window to avoid surprise restore delays.  
• For bulk historical pulls, plan ahead: use bulk retrieval options and schedule restores  
during off-hours.  
Reduce the footprint before moving  
• Compress the logs. If they are text/JSON, store as gzip or, better, convert to Parquet in  
the processed layer to massively reduce size and speed up any future analytics.  
• Avoid billions of tiny objects. If the raw layer has many small files, compact them in the  
processed layer so Intelligent-Tiering monitoring overhead and lifecycle actions remain  
efficient.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Operational guardrails  
• Bucket encryption with KMS and bucket-level policies remain unchanged across  
storage classes.  
• Enable S3 Storage Lens or Cost Explorer to verify savings and adjust thresholds based  
on real access patterns.  
© Shubham Wadekar
• Test restores from each cold class once, document runbooks, and set expectations for  retrieval times and costs.  
In simple terms: keep the last three months hot, move months 3–12 to a cheaper retrieval 
friendly class, and push anything older into deep archive. Do this automatically with lifecycle  policies, and shrink the data with compression/Parquet so you pay less from day one.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 5. ML datasets are used heavily for the first few weeks, then rarely. Which S3 storage class would you pick, and how would you automate transitions?

I would start the dataset in a fast, general-purpose class while it is actively used, and then let  S3 automatically move it to cheaper tiers as access drops. If usage is unpredictable across  © Shubham Wadekar
teams, I prefer S3 Intelligent-Tiering so S3 handles the movement without me guessing  
thresholds. If the pattern is always the same, I use clear lifecycle steps to the cheapest safe  classes.  
What I store and how I lay it out  
• I keep each dataset version under a clean prefix like  
s3://ml/datasets/<project>/<version>/, and I tag objects with project, version,  
data_class=ml_dataset, created_utc, and retention_days.  
• If the data is tabular, I write Parquet with Snappy. For image or text corpora, I compress  archives or keep reasonably large objects to avoid many tiny files.  
Option 1: unpredictable access, use S3 Intelligent-Tiering  
• I upload to the dataset prefix with Intelligent-Tiering enabled. Objects start in the  
frequent access tier during the first weeks while training and experimentation are heavy.  
• When objects are not accessed, S3 shifts them to infrequent access automatically,  
reducing cost with no changes to my code. If needed, I enable the archive tiers within  
Intelligent-Tiering so long-unused data moves even cheaper, with restore time similar to  
Glacier classes.  
• There are no retrieval fees between frequent and infrequent tiers, so surprise reads are  
safe. I account for the small per-object monitoring cost; I keep objects reasonably large  
to keep that overhead tiny.  
Option 2: predictable access, use lifecycle transitions  
• Hot phase: 0–30 days in S3 Standard for best performance during training.  
• Warm phase: 31–90 days in S3 Standard-IA for lower storage cost but instant access if  
we need to retrain quickly. I avoid moving earlier than 30 days to respect the minimum  
storage duration.  
• Cold phase: 91–365 days in S3 Glacier Instant Retrieval or Glacier Flexible Retrieval,  
depending on how quickly we might need it back. Instant Retrieval keeps millisecond  
access with much lower storage price; Flexible Retrieval is cheaper but has minutes-to 
hours restore.  
• Archive: older than a year in S3 Glacier Deep Archive when it is kept only for audit or  
reproducibility. I document restore times and costs so no one is surprised.  
How I automate it safely  
• I attach lifecycle rules to the dataset prefix or to objects with data_class=ml_dataset, so  I don’t affect other data in the bucket. 
• I keep versioning on for safety, then add noncurrent version expiration (for example,  
delete noncurrent versions after 30–60 days) to avoid cost creep.  
• I set abort-incomplete-multipart-uploads after 7 days to stop paying for abandoned  
parts.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• I use S3 Storage Lens or Cost Explorer to verify that transitions actually happen and that  cost drops as expected. If I see repeated re-access right after a transition, I shift the  
thresholds or switch to Intelligent-Tiering.  
• I store a manifest file per dataset version (list of keys, sizes, checksums) so restores or  
moves are easy.  
© Shubham Wadekar
In simple terms: I keep new ML datasets hot for a few weeks, then either let Intelligent-Tiering  auto-optimize cost, or I step them down with lifecycle rules from Standard to Standard-IA and  finally to Glacier tiers. I tag, version, and monitor so transitions are automatic and safe.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 6. Old intermediate ETL files in S3 are driving up costs. How would you apply lifecycle rules and storage classes to clean this up safely?

I treat intermediate files as disposable once the final outputs are validated. I separate them  clearly, tag them on write, move them quickly to cheaper classes, and then expire them on a  short timer. I also clean up failed uploads and old versions so the bucket does not accumulate  hidden costs.  
Structure and tagging so cleanup is easy  
• I keep clear prefixes for each stage, for example:  
• s3://lake/raw/ for immutable source of truth 
• s3://lake/processed/ for parquet outputs used by downstream  
• s3://lake/stage/tmp/ and s3://lake/stage/work/ for intermediate and scratch 
files  
• I tag intermediate objects with stage=intermediate, pipeline=name, run_id,  and created_utc. Final tables get stage=curated or stage=processed.  
Make deletion safe and idempotent  
• Every job writes to a run-scoped temp prefix, then promotes committed outputs to the  
final processed prefix only after all checks pass. That way, if a run fails, its  
intermediates are already isolated.  
• I drop a _SUCCESS marker with the final outputs and record expected row counts or  
checksums. The lifecycle policy for intermediates starts only after the corresponding  
final partition shows _SUCCESS.  
Lifecycle rules I apply  
• Transition quickly: move stage=intermediate objects to a cheaper class after a short  
delay. Common pattern is transition to S3 Standard-IA at 7 days because we rarely need  
intermediates beyond the first week. If the data is almost never re-read, I go straight to a  
Glacier tier at 7–14 days.  
• Expire aggressively: delete stage=intermediate after 14–30 days. For massive pipelines I  often choose 14 days; for mission-critical jobs I keep 30–60 days to allow reprocessing  
investigations.  
• Version cleanup: enable versioning for safety but add a noncurrent version expiration,  
for example keep only the last 1 noncurrent version and expire older noncurrent  
versions after 7–14 days on intermediate prefixes.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• Abort incomplete uploads: set abort-incomplete-multipart-uploads after 7 days to stop  paying for abandoned parts from failed runs.  
• Scope rules precisely: target the tmp/work prefixes or tag filters stage=intermediate so I  never touch raw or curated data by accident.  
Extra cost and performance hygiene  
© Shubham Wadekar
• Compact intermediates during the job so there are fewer, larger files even before  
deletion. This reduces listing costs and lifecycle evaluation overhead. 
• If intermediates must be kept briefly for debugging, I still compress them (gzip or  
Parquet) to reduce storage immediately.  
• For datasets with a legal hold or audit needs, I exclude those prefixes via tags like  
legal_hold=true so lifecycle never deletes them.  
• I monitor with S3 Storage Lens to confirm the object count and total bytes in  
stage=intermediate are trending down. If not, I adjust rules or fix pipelines that forgot to  
tag outputs.  
Runbook for exceptions  
• If a downstream job fails and we need to re-run, we read from raw or processed, not  
from intermediates. This lets us keep intermediate retention short.  
• If a team occasionally needs intermediates for root-cause analysis, I extend the  
transition window slightly or place them in Intelligent-Tiering for the first week, then let  
deletion kick in.  
In simple terms: I keep intermediates in their own prefix with tags, move them to a cheaper  
class within a few days, and delete them soon after final outputs are confirmed. I also clean up  failed uploads and old versions. This keeps storage bills low without risking important data.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 7. IoT raw data must be kept for 1 year, but analysts only query 30 days. How would you design lifecycle rules for compliance and cost efficiency?

I would separate what we query from what we only keep for compliance, use clear prefixes, and  let lifecycle rules move and delete data automatically.  
© Shubham Wadekar
What I store where  
• I keep the last 30 days in a “hot” prefix that Athena reads, for example:  
s3://iot/hot/device_events/dt=YYYY-MM-DD/  
• I keep day 31 to day 365 in an “archive” prefix for compliance, for example:  
s3://iot/archive/device_events/dt=YYYY-MM-DD/  
Storage classes I use  
• Hot data (0–30 days): S3 Standard or S3 Intelligent-Tiering so queries stay fast  
• Archive data (31–365 days): if we might need quick access sometimes, S3 Glacier  
Instant Retrieval; if access is almost never, S3 Glacier Flexible Retrieval or Deep Archive  
to save more  
Lifecycle rules I set  
• Rule for hot prefix: after 30 days, either delete the hot copy or move it to the archive  
prefix via a small job so we do not keep two copies  
• Rule for archive prefix: at 365 days, expire the object to meet the one-year retention  
• I add “abort incomplete multipart uploads after 7 days” to avoid paying for failed  
uploads  
• If compliance requires write-once, I enable Object Lock (governance or compliance  
mode) on the archive bucket so no one can delete early  
How I keep Athena safe  
• Athena tables only point to the hot prefix  
• I publish a default view that always filters to the last 30 days, so dashboards never scan  old data by mistake  
• If someone needs older data, we restore specific partitions from the archive into a  
temporary “restore” prefix and query from there 
Basic security and housekeeping  
• I encrypt with SSE-KMS and restrict archive access to a small group  
• I tag objects with data_class=iot_raw and retention_until=YYYY-MM-DD so rules target  
the right data  
• I watch S3 Storage Lens and Cost Explorer to check that hot data stays small and  
archive grows as expected  
In simple words: keep 30 days in a fast area for Athena, push older days to a cheap archive for  the rest of the year, and delete on day 366. Queries never touch the archive, so they do not  
break or get slow.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 8. Athena queries fail because some files moved to Glacier. How would you redesign lifecycle policies to prevent this issue?

I would stop moving anything that Athena needs into classes it cannot read, and keep archived  copies separate. The key is to split queryable data and archived data, and aim lifecycle rules  © Shubham Wadekar
only at the archive.  
Why it broke  
• Athena can read Standard, Standard-IA, One Zone-IA, Intelligent-Tiering, and Glacier  
Instant Retrieval  
• Athena cannot read objects that are in Glacier Flexible Retrieval or Deep Archive unless  they are restored first  
• Lifecycle rules pushed table data into an archive class that Athena cannot read, so  
queries failed  
How I redesign the layout  
• I create two clear locations:  
s3://lake/analytics/<table>/dt=YYYY-MM-DD/ for data that Athena queries  
s3://lake/archive/<table>/dt=YYYY-MM-DD/ for long-term storage that Athena  
does not touch  
• Athena tables and views only point to the analytics location  
Storage classes I allow for analytics  
• Keep analytics data in Standard or Intelligent-Tiering  
• If we need to cut cost more but still be queryable, I can use Glacier Instant Retrieval  
because Athena can read it without a restore  
• I never let analytics data transition to Glacier Flexible Retrieval or Deep Archive  
Lifecycle policies I apply  
• On analytics prefixes:  
Either do nothing, or at most move to Intelligent-Tiering (no retrieval surprises)  
Never transition to Glacier Flexible Retrieval or Deep Archive  
• On archive prefixes:  
Move older data to Glacier Flexible Retrieval or Deep Archive on a schedule  
Expire after the retention period  
• I scope rules by prefix or by object tags like stage=analytics or stage=archive so the right  data follows the right policy  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Safe migration from the current broken state  
• For partitions already in Deep Archive or Flexible Retrieval that we need, I restore only  
the required dates to a temporary restore prefix, then copy them back into the analytics  
prefix in a supported class  
• I update the Athena table or MSCK REPAIR to include the fixed partitions  
© Shubham Wadekar
• I add a guardrail check in the pipeline: before publishing a partition, we assert the  
storage class is allowed for analytics  
Operational guardrails  
• I tag analytics objects with stage=analytics and enforce a policy that blocks transitions  
to forbidden classes for that tag  
• I enable an S3 EventBridge rule or a daily check that lists any analytics objects not in an  allowed class and alerts the team  
• I document a short runbook: where to restore from, which classes Athena supports, and  how to re-point partitions if needed  
In simple words: keep Athena’s data in classes it can read, put true archives in a different place  with their own lifecycle, and add small checks so nothing from analytics ever gets pushed into a  non-readable class again.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar