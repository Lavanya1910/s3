# Query and Performance Optimization

## 19. Athena queries large CSV files but only need a few columns. How would you use S3 Select to cut cost and improve performance?

I would use S3 Select to read only the columns and rows I need from each CSV object instead of  downloading the whole file. This reduces data scanned and speeds up downstream processing.  © Shubham Wadekar
I typically use it in a small pre-filter step that writes a lighter dataset for Athena to query.  
How I apply it in practice  
• I add a lightweight Lambda or Glue job that reads each CSV object with S3 Select and  
runs a simple SQL like “SELECT col_a, col_b FROM S3Object WHERE event_date  
BETWEEN …”. 
• The job writes the filtered result as Parquet into a processed prefix. Athena then queries  the Parquet table, which is much faster and cheaper.  
• If a tool or notebook needs to read directly from S3, I call S3 Select from the SDK and  
pull only the required columns, not the whole file.  
Key setup choices  
• Keep CSV newline-delimited and clean (consistent headers, types).  
• Push as many filters into S3 Select as possible (date range, status flags).  
• Convert the S3 Select output to Parquet with Snappy so future queries are columnar  
and small.  
• Use parallelism by running one S3 Select call per object in parallel, then merge outputs.  
In simple words: I let S3 Select do the first cut on each CSV object (pick columns and filter  
rows), then I store that reduced data in Parquet for Athena. This lowers scanned bytes and  
speeds everything up.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 20. A data science team queries JSON logs but only needs some fields. How would you use S3 Select with Parquet/ORC to optimize efficiency?

For raw JSON, I use S3 Select to project only needed fields and to filter rows before I write  
Parquet. For Parquet, S3 Select can also read specific columns inside each object, which is  © Shubham Wadekar
useful for lightweight extraction tools. For ORC, I would not rely on S3 Select; instead I would  use normal engines (Athena/Spark) and let them prune columns.  
How I design the flow  
• Raw JSON → use S3 Select to pull only the fields needed (for example, event_time,  
user_id, country) and only the rows I care about (date range, event_type). Then I write  
Parquet.  
• Parquet → if a small script or Lambda needs data, I can use S3 Select on Parquet to  
fetch just a few columns from each object. For big analytics, Athena or Spark already  
prune Parquet columns efficiently, so I just query the Parquet table directly.  
• ORC → I keep using Athena/Spark for column pruning. If the team wants S3 Select style  
reads, I prefer converting to Parquet.  
Quality and performance tips  
• Normalize timestamps and types before writing Parquet so downstream tools don’t  
spend time fixing schemas.  
• Partition by date so both S3 Select and analytics engines read fewer objects.  
• Keep Parquet files in the 128–512 MB range for good scan efficiency.  
• Mask or drop PII during the JSON→Parquet step so sensitive fields never travel further  
than needed.  
In simple words: use S3 Select to trim raw JSON to only the fields and rows you need, store the  result as Parquet, and then let Athena or Spark query that Parquet efficiently. If you must read  Parquet from a small script, S3 Select can return just the columns you need.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 21. Spark ETL jobs on S3 are slow due to small files and poor partitions. How would you redesign file format, partitioning, and compression for performance and cost?

I would switch to a columnar format, fix the partition strategy, and control how many files Spark  writes so each partition ends up with a few large Parquet files. I also schedule compaction so  © Shubham Wadekar
the table stays healthy over time.  
File format and compression  
• Write Snappy-compressed Parquet (or ZSTD if your stack supports it well) for all  
processed/curated data. Parquet is columnar, so queries scan only needed columns.  
• Avoid plain CSV/JSON for analytics tables; keep them only in the raw layer for audit.  
• Disable schema merge at write time unless you truly need it, to avoid extra overhead.  
Partitions that actually help  
• Partition by date (dt=YYYY-MM-DD). Add hour (hr=HH) only if the daily volume is very  
high.  
• Do not partition by high-cardinality fields like user_id or session_id; that creates too  
many tiny partitions and many small files.  
• Keep partition columns stable and few in number so engines can prune effectively.  
Write fewer, larger files  
• Before the final write, repartition the DataFrame by the partition column to control the  
number of output files per partition.  
• Aim for 128–512 MB per file. As a rule of thumb, target file_count_per_partition  
≈ partition_size / 256 MB. 
• Tune Spark settings so you don’t create thousands of small tasks: set a sane  
spark.sql.shuffle.partitions and coalesce before the write when needed.  
• Use an atomic publish pattern: write to a temporary run path, compact if needed, then  
move to the final table path.  
Ongoing compaction and housekeeping  
• Add a daily compaction job that reads yesterday’s partitions and rewrites them into the  target file size if they are fragmented. 
• If you use Delta Lake, Apache Hudi, or Apache Iceberg, enable their  
OPTIMIZE/compaction/clustering features so the table maintains good file sizes  
automatically.  
© Shubham Wadekar
• Remove leftover _temporary and failed-run folders, and set “abort incomplete multipart  uploads” on the bucket. 
Extra read speed wins  
• Keep a concise schema: drop unused columns, use correct types (timestamp, bigint,  
decimal).  
• Enable predicate and column pushdown (Parquet filter pushdown) and avoid functions  that break it (for example, apply date filters on partition columns, not on transformed  
expressions).  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• Register tables in Glue Data Catalog with clear partitions; use partition projection if you  have very many partitions to avoid heavy metastore operations.  
• Cache small dimension tables in Spark if you join frequently.  
In simple words: store data as Snappy Parquet, partition mainly by date, and make sure each  partition has only a handful of large files. Use compaction regularly and avoid high-cardinality  © Shubham Wadekar
partitions. This makes Spark and Athena much faster and cheaper.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 28. Your team is running multiple queries on a large dataset stored in an S3 bucket, and the queries are taking too long to execute. How would you optimize Athena queries to improve performance?

I would reduce the amount of data Athena scans and cut the number of files it has to open. I  © Shubham Wadekar
also make queries more selective and keep the table design simple.  
What I fix first  
• Store data in columnar format (Parquet or ORC) with Snappy compression. This lets  
Athena read only the columns it needs instead of whole rows. 
• Right-size files. Aim for about 128–512 MB per file so each partition has a small number  of large files, not thousands of tiny ones.  
• Partition by how we filter. Usually this is by date (dt=YYYY-MM-DD) and maybe region or  site. I avoid high-cardinality partitions like user_id.  
Cut scanned bytes in queries  
• Always filter on partition columns (for example, WHERE dt BETWEEN … AND … AND