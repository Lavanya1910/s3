1. Your data lake stores raw data in CSV, and Athena queries are slow/expensive. How would you redesign storage format and structure to improve performance and reduce cost?

2. A new JSON log source needs onboarding into your S3 data lake. How would you organize raw, processed, and curated layers for efficient downstream use?

3. Daily ETL jobs are creating too many small files in S3, slowing Spark and Athena queries. How would you fix this in the data lake design?

4. You have 200 TB of logs in S3 Standard, but only the last 3 months are queried. How would you cut costs using storage classes?

5. ML datasets are used heavily for the first few weeks, then rarely. Which S3 storage class would you pick, and how would you automate transitions?

6. Old intermediate ETL files in S3 are driving up costs. How would you apply lifecycle rules and storage classes to clean this up safely?

7. IoT raw data must be kept for 1 year, but analysts only query 30 days. How would you design lifecycle rules for compliance and cost efficiency?

8. Athena queries fail because some files moved to Glacier. How would you redesign lifecycle policies to prevent this issue?

9. ETL jobs generate temporary files in S3 that aren’t needed later. How would you auto-clean them without deleting raw or curated data?

10. A developer accidentally deleted raw data, but versioning is enabled. How would you recover and prevent future issues?

11. You need data to remain immutable for 7 years due to compliance. How would you use S3 Object Lock to meet this need while still supporting analytics?

12. Versioning caused multiple large file versions, increasing costs. How would you balance versioning with lifecycle rules to control cost?

13. Customer data in India must also be stored in the EU for DR. How would you configure Cross-Region Replication (CRR), and what factors must you consider?

14. Some objects are missing in your Same-Region Replication (SRR) bucket. How would you troubleshoot and fix this?

15. You need to replicate only logs/objects to another account’s bucket in a different region. How would you design this policy and IAM setup?

16. A raw data bucket was found to be public. How would you secure it immediately and prevent future risks?

17. Your S3 data lake is accessed by multiple teams with different permissions. How would you design IAM roles and bucket policies for least-privilege access?

18. A partner needs temporary read-only access to certain S3 objects. How would you grant this securely (e.g., pre-signed URLs, access points)?

19. Athena queries large CSV files but only need a few columns. How would you use S3 Select to cut cost and improve performance?

20. A data science team queries JSON logs but only needs some fields. How would you use S3 Select with Parquet/ORC to optimize efficiency?

21. Spark ETL jobs on S3 are slow due to small files and poor partitions. How would you redesign file format, partitioning, and compression for performance and cost?

22. You want Glue ETL to run automatically when new files arrive in S3. How would you design S3 event notifications to avoid duplicates/misses?

23. S3 → Lambda integration misses some large files. How would you troubleshoot and redesign it for both small and large files?

24. You need to push S3 file metadata into SQS for real-time processing. How would you configure S3 event notifications for scale and reliability?

25. IoT data in S3 makes Athena queries slow because of wide scans. How would you redesign the partitioning strategy to improve performance?

26. You’re worried about read-after-write consistency for new files in S3. How would you design the ingestion pipeline to ensure consistent downstream reads?

27. Spark ETL produces thousands of tiny Parquet files per partition in S3, slowing Athena. How would you optimize partitioning and file sizes?

28. Your team is running multiple queries on a large dataset stored in an S3 bucket, and the queries are taking too long to execute. How would you optimize Athena queries to improve performance?