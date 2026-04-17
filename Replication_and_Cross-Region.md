# Replication and Cross-Region

## 13. Customer data in India must also be stored in the EU for DR. How would you configure Cross-Region Replication (CRR), and what factors must you consider?

I would set up one S3 bucket in India as the source and one S3 bucket in the EU as the  
destination, enable versioning on both, and create a replication rule that copies new and  
© Shubham Wadekar
changed objects automatically. I also make sure encryption and permissions are correct so the  replication never gets stuck.  
How I set it up  
• Create the source bucket in India (for example, ap-south-1) and the destination bucket  
in the EU (for example, eu-west-1 or eu-central-1).  
• Turn on bucket versioning on both buckets. Replication requires versioning.  
• Create a replication rule in the source bucket:  
• Scope it by prefix or object tags if we only want to replicate specific data (for  
example, only customer data).  
• Choose replicate delete markers if DR requires keeping deletes in sync, or skip it  
if you want the EU copy to be “backup style”. 
• Turn on replication metrics and optional Replication Time Control (RTC) if we  
need a time-bound SLA for replication.  
• Configure object ownership in the rule to make the destination bucket owner the owner  of replicated objects. This avoids access issues later.  
• Encryption:  
• If we use SSE-S3, no extra work.  
• If we use SSE-KMS, create or pick KMS keys in both regions. Update the key  
policies so the S3 replication role can decrypt with the source key and encrypt  
with the destination key. Multi-Region KMS keys make this simpler.  
• IAM and roles:  
• Let S3 create the replication role automatically or provide a role with 
s3:ReplicateObject, s3:ReplicateDelete, and KMS permissions.  
• Test with a small file and check the object’s replication status in its metadata. 
Factors I consider before choosing regions and settings  
• Compliance and data transfer rules. Confirm cross-border transfer is allowed and  
documented. Sometimes legal asks for specific EU region choices.  
• Recovery point and time objectives. If we need tighter guarantees, I enable RTC;  
otherwise standard CRR is usually enough.  
• Cost. Cross-region data transfer and extra PUT requests add cost. I set lifecycle rules on  the EU bucket to use cheaper storage classes after a few weeks if the DR copy is rarely  
read.  
• Storage classes and query behavior. If analytics will also run in the EU during DR, I keep  the EU copy in a class Athena can read (Standard, Intelligent-Tiering, or Glacier Instant  
Retrieval). If it is true cold DR, I may move older objects to Glacier classes in the EU.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• Security. Encrypt both sides, restrict who can read the EU copy, and log all access.  
Consider S3 Object Lock in the EU if the DR copy must also be immutable.  
• Existing data. CRR only replicates new objects by default. If we must copy historical  
data, I run S3 Batch Replication once.  
In simple terms: I enable versioning, add a replication rule from India to the EU, give S3 the right  © Shubham Wadekar
KMS and IAM permissions, and test. Then I tune lifecycle, cost, and security so the DR copy is  safe, compliant, and affordable.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 14. Some objects are missing in your Same-Region Replication (SRR) bucket. How would you troubleshoot and fix this?

I would check the basics first (versioning, rule scope, encryption, and permissions), look at  
replication metrics to see which objects failed, and then re-replicate only the missing ones.  What I check first  
• Versioning. Make sure versioning is enabled on both source and destination. Without it,  replication does not work.  
• Rule scope. Confirm the replication rule actually covers the objects’ prefixes or tags. If  
the rule filters by a tag that the objects do not have, they will be skipped.  
• Encryption. If the source objects use SSE-KMS, make sure the replication role can  
decrypt with the source KMS key and encrypt with the destination KMS key. Replication  
does not work with customer-provided SSE-C keys.  
• Ownership and ACLs. Use bucket owner enforced and set the replication rule to change  object ownership to the destination bucket owner. This avoids “replicated but not  
readable” cases. 
• Replication status. On a missing object in the source bucket, check the x-amz 
replication-status. If it shows FAILED or PENDING, we know replication attempted or is  
stuck.  
Where I look for errors  
• S3 replication metrics and dashboard in the S3 console. I look for failed operations,  
bytes pending replication, and any recent spikes.  
• CloudWatch logs and EventBridge notifications if we enabled them for replication  
failures.  
• KMS key policy. Very often the missing objects are the ones encrypted with a key that  
the replication role cannot use.  
Common root causes and fixes  
• Rule did not apply to those objects. Fix the rule filter (prefix or tags) and re-run  
replication for those objects using S3 Batch Replication.  
• No permission to use KMS. Update the key policies to allow the S3 replication role to  
decrypt source and encrypt destination. After fixing, trigger Batch Replication for  
affected keys.  
• Objects existed before SRR was turned on. Set up S3 Batch Replication once to backfill  historical objects.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• Delete markers or metadata-only changes. If we need deletes to replicate, enable  
“replicate delete markers” in the rule. Otherwise the destination will keep old copies. 
• Application overwrote keys too often, creating many versions rapidly. Replication may  
lag or fail if the role or limits were misconfigured. Reduce overwrites by writing to run 
specific keys and publishing atomically.  
© Shubham Wadekar
How I catch up and prevent repeats  
• Use S3 Batch Replication to copy only the missing objects. I scope it by date or prefix so  it is fast and cheap.  
• Keep replication metrics and alarms on so we know quickly if failures return.  
• Add a small daily check that lists source keys for the last day and confirms they exist in  the destination. If not, it alerts and optionally triggers Batch Replication automatically.  
In simple terms: I verify versioning, rule filters, KMS permissions, and ownership. I use the S3  replication metrics to find what failed, fix the cause, and backfill the missing objects with Batch  Replication. Then I keep simple alarms so it does not happen again.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 15. You need to replicate only logs/objects to another account’s bucket in a different region. How would you design this policy and IAM setup?

I would use Cross-Region Replication (CRR) with cross-account permissions, scope the rule  only to the log prefix, and set IAM and bucket policies so replication is automatic and secure.  © Shubham Wadekar
How I design it  
• Source bucket: in my account, where raw logs land (for example, ap-south-1).  
• Destination bucket: in the other account, in a different region (for example, eu-west-1).  
This bucket is owned by the partner account but grants permissions to my replication  
role.  
• Both buckets must have versioning enabled.  
Replication rule setup  
• On the source bucket, I create a replication rule scoped only to the logs prefix, for  
example s3://my-source-bucket/logs/.  
• The rule includes “replicate delete markers” if we want deletes mirrored; if it’s DR, we  
usually skip delete replication so the destination keeps a backup copy.  
• I enable replication metrics and possibly Replication Time Control (RTC) if SLA  
guarantees are needed.  
IAM and bucket policies  
• Replication role (in my account):  
• Trust policy: trusted by S3 service.  
• Permissions: s3:GetObjectVersionForReplication, s3:GetObjectVersionAcl,  
s3:GetObjectVersionTagging on the source bucket.  
• Permissions: s3:ReplicateObject, s3:ReplicateDelete, s3:ReplicateTags on the  
destination bucket.  
• If encrypted with KMS, the role also needs kms:Decrypt on the source key and  
kms:Encrypt on the destination key.  
• Destination bucket policy (in the other account):  
• Grants my replication role permission to put objects.  
• Example statement: Allow principal = arn:aws:iam::<source-account 
id>:role/S3ReplicationRole, action = s3:ReplicateObject, s3:ReplicateDelete,  
resource = arn:aws:s3:::destination-bucket/*  
Encryption  
• If using SSE-S3, nothing extra.  
• If using SSE-KMS, I set up a KMS key in the destination account and give the replication  
role encrypt permissions.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Testing  
• I upload a small test object into s3://my-source-bucket/logs/.  
• In the destination account’s bucket, I confirm it appears and its metadata shows  
Replication status = COMPLETED.  
In simple words: I scope replication to just the logs prefix, create a replication role in my  
© Shubham Wadekar
account, grant it permission in the other account’s bucket, and test with a small object. This  way only logs are replicated, and nothing else.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar