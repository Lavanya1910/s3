# Data Lake Architecture and Organization

## 1. Your data lake stores raw data in CSV, and Athena queries are slow/expensive. How would you redesign storage format and structure to improve performance and reduce cost?

I would change two things: how the data is stored (columnar format, compression, right-sized  © Shubham Wadekar
files) and how it is laid out in S3 (clean partitions and atomic writes). 
First, convert CSV to a columnar format like Parquet. Columnar files let Athena read only the  columns used in a query, which cuts scanned bytes a lot. I would also compress with snappy or  zstd. This typically reduces both cost and runtime dramatically compared to plain CSV.  
Second, partition by the fields most commonly used in filters, usually a date column like  
dt=YYYY-MM-DD, and maybe one low-cardinality dimension such as region or source. I avoid  over-partitioning; date, and optionally one more dimension, is usually enough.  
Third, avoid thousands of tiny files. I aim for 128–512 MB per Parquet file. I batch writes (via  Glue/EMR/Firehose) and schedule a compaction job that merges small files inside each  
partition. Fewer, larger files make planning and reading faster.  
Fourth, make writes atomic. Each job writes to a run-scoped temp path, then promotes to the  final partition and drops a _SUCCESS marker or manifest. Readers only scan partitions with the  marker, so they never see half-written data.  
Fifth, register clear table metadata in Glue Data Catalog with proper data types. For large date  ranges, I enable partition projection so Athena computes partitions instead of listing millions of  folders. I keep table and column names lowercase and consistent.  
Sixth, create analyst-friendly presentation tables or views that expose only needed columns  and hide raw noise. That reduces scanned data and avoids select *. If late data is common, I  materialize a small, recent summary table to speed daily dashboards. 
In simple terms: convert CSV to Parquet with compression, use sensible partitions, keep files  big, write atomically, and register clean schemas. This combination cuts scan cost and speeds  up Athena immediately.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 2. A new JSON log source needs onboarding into your S3 data lake. How would you organize raw, processed, and curated layers for efficient downstream use?

I would land JSON safely in a raw area, standardize it in a processed area, and publish  
business-ready Parquet tables in a curated area. Each layer has a clear purpose.  
© Shubham Wadekar
Raw layer  
I store exactly what arrives, unmodified and immutable, for audit and replay. I use newline 
delimited JSON, optionally gzipped, under a simple layout like s3://lake/raw/app_logs/dt=YYYY MM-DD/hr=HH/. Every ingestion writes to a temp path, then promotes and adds a _SUCCESS  marker. I also write bad records to a separate rejects prefix with the error reason.  
Processed layer  
I enforce schema, types, and basic quality rules. I parse the JSON into structured columns,  flatten only the fields we actually use, flatten nested objects in a single column if they are  rarely queried. I add standard columns like event_time, dt, source, and an event_id for dedup. I  mask or tokenize PII here so sensitive values never reach downstream in plain text. Output is  Parquet, partitioned by dt (and maybe hour or region), with 128–512 MB files and compaction. I  register the table in Glue, and allow only additive schema changes.  
Curated layer  
I publish business-oriented tables that analysts and BI use directly. These are clean, well 
named Parquet tables, often arranged as a star schema (facts and dimensions) or purpose built aggregates. I keep only the columns needed for analytics, set clear data types, and add  surrogate keys if useful. If slowly changing dimensions are required, I maintain them here. I  
expose stable views that hide internal technical columns.  
Cross-cutting practices  
I manage metadata in Glue Data Catalog for all layers and use Lake Formation for permissions.  I use partition projection for very large partition counts. I keep idempotency by writing to run scoped paths and promoting only on success. I schedule validation checks (row counts, null  rates, duplicate event_ids) and capture simple lineage: raw path → processed table → curated  table. If the source adds fields, the pipeline accepts new nullable columns in processed, and I  update curated on my schedule so downstream tools are not surprised.  
In simple terms: land JSON as-is in raw, standardize and secure it as Parquet in processed, and  publish tidy, business-ready Parquet tables in curated. Clear partitions, big files, atomic writes,  and a single Glue catalog make the data easy and cheap to query.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 3. Daily ETL jobs are creating too many small files in S3, slowing Spark and Athena queries. How would you fix this in the data lake design?

I would attack the root causes (over-partitioning and uncoordinated writers) and then add  
compaction so we always end up with a small number of large Parquet files per partition. My  © Shubham Wadekar
goal is to land 128–512 MB Parquet files, partitioned sensibly, written atomically, and  
compacted on a schedule.  
Diagnose the issue quickly  
• Check a few partitions and count files. If I see hundreds of files that are 1–10 MB, that  
explains the slow scans in Athena and the small-file overhead in Spark.  
• Confirm the partition scheme. If we partition by too many keys (like dt=YYYY-MM 
DD/hr=HH/min=MM/app=user_id), each partition becomes tiny and creates small files.  
• Look at writer concurrency. Many small Spark tasks writing independently will each  
produce separate files.  
Design changes to reduce small files at the source  
• Right-size partitions. I keep dt at day, add hour only if volume is high. I avoid minute 
level partitions and avoid putting high-cardinality fields (like user_id) in the partition  
path.  
• Batch the write window. Instead of writing every 5 minutes, I buffer and write hourly  
where feasible. If upstream must be more frequent, I still compact within the hour.  
• Control the number of output files. In Spark I set the number of shuffle partitions and  
use repartition or coalesce before the final write so each partition ends up with only a  
few large files.  
• Target file size. I tune Spark to aim for ~256 MB files using options like  
spark.sql.files.maxRecordsPerFile (indirectly controls size) and by adjusting the number  
of output partitions to match the data volume.  
Concrete Spark/Glue techniques  
• Before writing: compute the target number of files per partition as roughly  
data_size_in_partition / 256MB, then use df.repartition(n, "dt") or  
df.repartitionByRange("dt") so Spark writes that many files.  
• After write: if I still get small files, run a quick compaction job that reads a partition and  
writes it back with a lower number of output partitions (coalesce) to merge files.  
• Use columnar format. Always write Snappy Parquet (or ORC). It reduces file count and  
speeds Athena.  
• Avoid too many small tasks. Set spark.sql.shuffle.partitions to a sane number based on  cluster size and daily input volume.  
• Atomic, idempotent writes. Write to a temporary run path, then promote to the final  
prefix only on success. That prevents partial partitions with a mix of small files from  
failed runs.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Automated compaction and housekeeping  
• Add a daily compaction job per table/partition. It reads yesterday’s partitions and  
rewrites them into 128–512 MB files, then replaces the old files in a single commit.  
• For streaming or micro-batch pipelines, I compact hourly and then run a final daily  
compaction to hit ideal sizes.  
© Shubham Wadekar
• Optional table formats. If the lake uses Apache Hudi, Delta Lake, or Apache Iceberg, I  
enable their built-in compaction/clustering so the table stays healthy automatically  
over time.  
Improvements for Athena and downstream  
• Keep partitions predictable and not overly granular so partition pruning actually helps.  
• Use Glue Data Catalog with correct parquet statistics and enable partition projection  
for very high partition counts to avoid expensive MSCK REPAIR calls.  
• Educate producers. If a producer pushes thousands of tiny JSON files, I either batch  
them upstream (for example, larger flush intervals) or stage them and merge during the  
processed step.  
In simple terms: I reduce partition granularity, control how many files Spark writes per partition,  always write Parquet, and schedule compaction so each partition ends up with a handful of  large files. That fixes query speed and lowers compute cost.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 25. IoT data in S3 makes Athena queries slow because of wide scans. How would you redesign the partitioning strategy to improve performance?

I would make partitions match how people actually filter the data, keep them simple and  
predictable, and avoid very high-cardinality columns. My main goal is that every Athena query  touches only a small set of folders, not the whole table.  
What I change in the layout  
• I organize data by date first, because almost all queries filter by time. For example:  
s3://lake/iot/device_events/dt=YYYY-MM-DD/hr=HH/.  
• If we also filter by geography or site, I add a second low-cardinality partition such as  
region or site. For example: dt=YYYY-MM-DD/region=APAC/ or region first if most  
queries filter by region.  
• I never partition by device_id because it creates millions of tiny folders. Instead I keep  
device_id as a normal column and let the engine filter it inside the files.  
How this speeds Athena  
• When queries include predicates like dt between and region in, Athena prunes all other  partitions and scans only the needed folders.  
• Fewer partitions per query means fewer files to open, lower scanned bytes, and much  
faster results at lower cost.  
Practical settings and guardrails  
• I store data as Parquet with Snappy so Athena can read just the needed columns.  
• I size files to about 128–512 MB per file so each partition has a handful of files, not  
thousands.  
• I register the table in Glue with the same partition columns (order matters). I enable  
partition projection for large date ranges so Athena does not need to load millions of  
partitions into the catalog.  
• I use clear naming and keep partition columns stable. Changing partition columns later  is painful and slows queries.  
In simple words: partition mainly by time (and maybe region/site), not by device_id. Keep files  columnar and reasonably large. Then Athena only reads the few folders that match the filter and  runs much faster.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 26. You’re worried about read-after-write consistency for new files in S3. How would you design the ingestion pipeline to ensure consistent downstream reads?

I design the pipeline so readers only see a partition after it is fully written and validated. I use a  temporary path, a success marker or manifest, and an atomic “promote” step. This makes  
© Shubham Wadekar
readers consistent even if infrastructure is slow.  
Write pattern  
• The writer lands data in a run-scoped temporary folder like  
s3://lake/tmp/run_id=UUID/… . 
• After writing all files, it does validation checks (row counts, schema, basic quality).  
• Only then it moves or copies the files into the final partition path like  
s3://lake/processed/table/dt=YYYY-MM-DD/hr=HH/.  
• Finally it writes a small _SUCCESS file or a manifest file listing the exact objects for that  partition.  
Read pattern  
• Downstream jobs and Athena read only partitions that have the _SUCCESS file or a  
manifest present.  
• If a downstream job runs while a partition is still being written, it simply skips it because  the marker is not there yet.  
Idempotency and retries  
• The writer includes a run_id and an idempotency key (bucket + key + version or ETag) in  
a small tracking table. If a retry happens, it does not publish the same data twice.  
• Glue writes to a run-scoped temp path and promotes only on success. If  
the same file comes again, the job does nothing.  
Operational tips  
• For event-driven loads, I queue S3 events into SQS and trigger processing from there to  
avoid race conditions.  
• I compact small files before the promote step to keep final partitions healthy.  
• I keep schema and partition columns consistent so readers can rely on predictable  
locations.  
In simple words: write to a temp area, validate, then publish and drop a _SUCCESS or manifest.  Readers only look at published partitions, so they always see complete, consistent data.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 27. Spark ETL produces thousands of tiny Parquet files per partition in S3, slowing Athena. How would you optimize partitioning and file sizes?

I would reduce partition granularity, control the number of output files, and add compaction so  each partition ends up with only a few large Parquet files. This makes both Spark and Athena  © Shubham Wadekar
much faster.  
Fix partitioning  
• Keep partitions simple, usually by dt=YYYY-MM-DD, and add hr=HH only if the daily  
volume is very high.  
• Do not partition by high-cardinality columns like device_id or user_id. That is the main  
cause of tiny files.  
Control file counts at write time  
• Before the final write, repartition the DataFrame by the partition column and set the  
number of output partitions to target 128–512 MB per file.  
• I coalesce output partitions if the data is small to avoid creating dozens of tiny files.  
• I set reasonable Spark shuffle partitions so I do not create thousands of tiny tasks that  
each write a file.  
Use the right format and compression  
• I write Snappy Parquet for processed and curated data. Parquet is columnar and  
compresses well, which reduces both storage and scan cost.  
• I avoid writing CSV/JSON in analytics layers; I keep those only in raw.  
Add compaction  
• I schedule a small daily job that reads yesterday’s partitions and rewrites them into  
large files if they are fragmented. After compaction, I replace the old files in one atomic  
step.  
• If I use Delta Lake, Apache Hudi, or Apache Iceberg, I enable their optimize/compaction  features so tables stay healthy automatically.  
Safe publish  
• I write to a run-scoped temp path, compact there, and then promote to the final  
partition with a _SUCCESS marker. This prevents half-written partitions.  
Athena-friendly practices  
• I keep file counts per partition low (for example, 5–20 files). That reduces open/close  
overhead.  
• I register partitions in Glue and consider partition projection for very large date ranges.  
• I keep schemas clean and typed correctly so predicate pushdown works well.  
In simple words: stop over-partitioning, write Parquet, control the number and size of output  files, and compact regularly. Each partition should have a handful of big files, not thousands of  tiny ones. This makes Athena scan much less data and run much faster.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar