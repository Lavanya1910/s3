# File Management and Cleanup

## 9. ETL jobs generate temporary files in S3 that aren’t needed later. How would you auto-clean them without deleting raw or curated data?

I would separate temporary files from important data and use lifecycle rules to auto-delete  
them after a short time. The key is to clearly tag or store temporary files in their own prefix so  © Shubham Wadekar
cleanup rules never touch raw or curated layers.  
How I organize  
• I create clear prefixes:  
• s3://lake/raw/ → permanent source of truth 
• s3://lake/processed/ → business-ready data  
• s3://lake/tmp/ or s3://lake/stage/ → temporary ETL files 
• I also tag temporary files with stage=tmp or stage=intermediate at the time they are  
written.  
How I auto-clean  
• I set a lifecycle rule only for the tmp prefix (or objects with stage=tmp tag).  
• The rule deletes files after a short time, usually 1–7 days depending on how long jobs  
might need them for retries.  
• I also configure “abort incomplete multipart uploads after 7 days” so half-written files  
don’t accumulate. 
How I make it safe  
• Rules are scoped only to the tmp prefix or tmp-tagged objects, so raw and curated are  
untouched.  
• Jobs always write temp files to a run-specific subfolder like  
s3://lake/tmp/run_id=UUID/, then promote only the final good outputs into processed.  
This makes cleanup predictable.  
• If debugging is needed, I can extend retention to 7–14 days, but it’s still auto-cleaned.  
In simple words: I keep temp files in their own folder, tag them, and set lifecycle rules to delete  them after a few days. This keeps costs low and makes sure raw and curated data are never  touched.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar