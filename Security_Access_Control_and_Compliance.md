# Security, Access Control, and Compliance

## 10. A developer accidentally deleted raw data, but versioning is enabled. How would you recover and prevent future issues?

Since versioning is on, the deleted objects aren’t gone they just have delete markers. I can  restore by removing those markers or copying older versions back. After recovery, I would add  © Shubham Wadekar
guardrails so this doesn’t happen again. 
How I recover  
• In S3 console, I enable “show versions” and see that the deleted objects have previous  
versions still available.  
• I either:  
• Delete the delete marker to bring the last version back, or  
• Copy the previous version to a new prefix like s3://lake/raw_restored/ and point  
Glue/Athena to it.  
• For large datasets, I would use S3 Batch Operations or AWS CLI with --version-id to  
restore in bulk.  
How I prevent this in future  
• Enable MFA Delete on the raw bucket so no one can permanently delete or overwrite  
without MFA approval.  
• Use S3 Object Lock (governance mode) to enforce write-once for a set period if  
compliance allows. That way raw data cannot be deleted until the retention expires.  
• Restrict IAM permissions so developers only have write access to raw, not delete. Only  
admins can delete.  
• Set up CloudTrail or S3 EventBridge to alert if anyone tries to delete raw data.  
In simple words: I recover by rolling back to the previous versions since versioning kept them.  Then I put guardrails like MFA Delete, Object Lock, and stricter IAM permissions so raw data  can’t be deleted by mistake again. 
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 11. You need data to remain immutable for 7 years due to compliance. How would you use S3 Object Lock to meet this need while still supporting analytics?

I would keep an immutable “archive copy” of the raw data under Object Lock for 7 years, and do  all analytics from a separate processed copy. This gives me legal-grade immutability without  © Shubham Wadekar
blocking day-to-day work.  
How I set up the immutable archive  
• I create a dedicated archive bucket just for compliance data, for example:  
s3://company-archive-raw/.  
• When creating the bucket, I enable bucket versioning and turn on S3 Object Lock.  
• I choose Compliance mode (cannot be shortened or removed by any user, even admins)  with a default retention of 7 years. If the business needs a little flexibility, I use  
Governance mode but still restrict who can bypass it.  
• I optionally use legal holds for specific investigations. A legal hold freezes objects until  
the hold is removed, independent of the 7-year timer.  
• I lock only the archive bucket. This way, day-to-day analytics isn’t slowed down. 
How I write data into the archive  
• Ingest pipelines write raw files once into the archive bucket using write-once keys (no  
overwrites).  
• I tag objects with source, dt, and retention metadata so auditing is simple.  
• If we need a second region for resilience, I set up S3 Replication with Object Lock  
enabled on the destination bucket so retention carries over.  
How I support analytics safely  
• I keep a separate analytics bucket/prefix, for example: s3://lake/processed/ and  
s3://lake/curated/. These are not object-locked.  
• ETL jobs read from the immutable archive and produce clean Parquet tables in the  
processed/curated area. Analysts and Athena/Glue point only to processed/curated,  
not to the locked archive.  
• If a transformation bug is found, I can always reprocess from the immutable archive  
because it’s untouched and guaranteed complete. 
• I use lifecycle policies on processed/curated to control cost (transition older partitions  
to cheaper classes that Athena can still read, or expire if fully replaceable).  
Operational guardrails  
• I restrict delete/overwrite permissions on the archive bucket to prevent mistakes. Even  
if someone tries, Object Lock will block it.  
• I monitor with S3 Inventory and Storage Lens to prove retention and show auditors the  
evidence.  
• I encrypt both buckets with KMS and log access with CloudTrail for audit trails.  
In simple words: I lock the raw data for 7 years in a special bucket so nobody can change or  delete it, and I do all reporting from a separate processed copy. If something goes wrong, I can  always rebuild from the locked raw data.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 12. Versioning caused multiple large file versions, increasing costs. How would you balance versioning with lifecycle rules to control cost?

I keep versioning where I truly need safety (raw and critical data), and I clean up noncurrent  versions everywhere else with clear lifecycle rules. I also change how we write data to avoid  © Shubham Wadekar
creating too many versions in the first place.  
Where I keep versioning and where I don’t 
• Raw and archive buckets: keep versioning on for safety and audit.  
• Processed/curated buckets: keep versioning on, but aggressively clean up old versions  
because we can always rebuild from raw.  
• Scratch/tmp buckets: versioning usually not needed; if enabled, clean them very  
quickly.  
Lifecycle rules I apply to control cost  
• Noncurrent version transitions: move noncurrent versions to cheaper classes after 30  
days (for example, Standard-IA or Glacier Instant Retrieval if we may need a quick  
rollback; Glacier Flexible Retrieval for rarely needed rollbacks).  
• Noncurrent version expiration: delete noncurrent versions after 60–90 days, or keep  
only the last N noncurrent versions (for example, keep the last 1–2) and expire the rest.  
This keeps a safety net without paying for dozens of copies.  
• Abort incomplete multipart uploads after 7 days so abandoned parts don’t waste  
money.  
• Scope rules by prefix or tags. For example, stage=processed gets aggressive cleanup;  
stage=raw keeps longer retention.  
Change write patterns to reduce versions  
• Write new data to new keys instead of overwriting the same key. For example, write by  
partition path dt=YYYY-MM-DD/hr=HH/ so a new run doesn’t create versions of the  
same object.  
• Use atomic “publish”: write to a run_id path, validate, then copy/rename into the final  
partition. This avoids repeated overwrites of the same object and therefore avoids many  
noncurrent versions.  
• Compact outputs so there are fewer, larger files. Fewer files means fewer versions  
overall if something is updated.  
Monitor and tune  
• Use S3 Storage Lens and Cost Explorer to see where noncurrent bytes are growing.  
• If I see frequent rollbacks needed, I extend the noncurrent retention a bit (for example,  
from 30 to 45 days). If no rollbacks happen, I shorten it.  
In simple words: keep versioning where it protects you, but set lifecycle rules to move old  
versions to cheaper storage and delete them after a short, safe window. Also, prefer writing new  files instead of overwriting the same ones so you don’t create lots of versions in the first place.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 16. A raw data bucket was found to be public. How would you secure it immediately and prevent future risks?

First, I would lock it down right away, then I would put permanent guardrails so it can’t  
accidentally go public again.  
Immediate actions  
• Disable public access using the S3 Block Public Access settings on the bucket and at  
the account level. This blocks any ACL or policy that tries to make it public.  
• Review and delete any bucket policies or object ACLs that grant public access (for  
example, Principal = *).  
• Confirm using the S3 console “Access Analyzer” that the bucket is no longer publicly  
accessible.  
Prevent future risks  
• Keep Block Public Access enabled by default at the account level.  
• Enforce IAM policies so developers cannot change public access settings. For example,  deny s3:PutBucketAcl with public grants.  
• Use Service Control Policies (if in an organization) to block making buckets public.  
• Enable CloudTrail and GuardDuty to alert if any changes happen to bucket policies or  
ACLs.  
• Use AWS Config rules such as s3-bucket-public-read-prohibited to continuously check  and auto-remediate if any bucket turns public.  
• Keep data encrypted (SSE-KMS) and give access only via IAM roles with least privilege.  
Extra governance  
• Enable versioning and Object Lock (governance mode) if the data must remain  
immutable.  
• If logs or sensitive data are inside, review access logs to confirm whether anyone  
actually accessed the bucket while it was public.  
In simple words: I immediately block public access and remove open policies, then set  
account-wide Block Public Access, monitoring, and IAM restrictions so no one can accidentally  expose raw data again.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 17. Your S3 data lake is accessed by multiple teams with different permissions. How would you design IAM roles and bucket policies for least-privilege access?

I would split access by “who does what” (producer, consumer, curator, admin), scope  
permissions to only the prefixes each team needs, and enforce guardrails at the bucket level so  © Shubham Wadekar
mistakes don’t open up data. I keep encryption and network controls consistent for everyone.  How I organize the data and access  
• I organize data by domains and layers so it’s easy to grant narrow access, for example: 
o s3://lake/raw/<domain>/… 
o s3://lake/processed/<domain>/… 
o s3://lake/curated/<domain>/… 
• I create IAM roles per team persona:  
• producer_role_<domain>: can write only to raw/<domain> and list the bucket  
paths it needs  
• curator_role_<domain>: can read raw/<domain>, write processed/<domain>,  
and manage schema/metadata if required  
• consumer_role_<team>: read-only to curated/<team_or_domain> paths  
• lake_admin_role: limited set of admins who can manage policies, not used for  
day-to-day work  
• Users and jobs assume these roles (no long-lived IAM users). 
Least-privilege IAM policies (key ideas)  
• For read-only consumers:  
• s3:ListBucket with a condition to allow listing only keys that start with allowed  
prefixes (s3:prefix, s3:delimiter)  
• s3:GetObject only on arn:aws:s3:::lake/curated/<team_or_domain>/*  
• For producers:  
• s3:PutObject and s3:AbortMultipartUpload only on  
arn:aws:s3:::lake/raw/<domain>/*  
• s3:ListBucket limited to raw/<domain> prefixes  
• Optional s3:GetObject if their job needs to read its own uploads for validation  
• For curators:  
• s3:GetObject on raw/<domain>/*  
• s3:PutObject on processed/<domain>/* (and maybe curated/<domain>/* if they  
publish)  
• s3:ListBucket scoped to those prefixes  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
Bucket-level guardrails (one-time controls)  
• Block public access at account and bucket level.  
• Enforce TLS only: bucket policy denies any request where aws:SecureTransport is false.  
• Enforce encryption on write: bucket policy denies s3:PutObject if x-amz-server-side 
encryption is missing or not aws:kms.  
• Enforce bucket-owner-enforced object ownership so uploads default to bucket owner  
and ACLs are not needed.  
• Optional network control: require S3 access via specific VPC endpoints using  
aws:SourceVpce in the bucket policy.  
KMS key access  
• Use a dedicated KMS key for the lake.  
• KMS key policy allows only the lake roles to encrypt/decrypt.  
• For consumer roles, allow kms:Decrypt; for producer roles, allow kms:Encrypt and  
kms:Decrypt if they need to verify; scope with conditions on the bucket and prefixes.  
Scaling and simplicity with S3 Access Points (optional but helpful)  
• Create one access point per domain or team, each with its own policy limited to that  
prefix.  
• Optionally restrict access points to specific VPCs (VPC-only access).  
• Point jobs and Athena workgroups to the right access point alias to avoid long bucket  
policies.  
Extra safety and hygiene  
• Permission boundaries for developer roles so they can’t grant themselves more S3  
rights.  
• AWS Config rules to alert on any bucket that becomes public or any policy that allows  
wild access.  
• CloudTrail and S3 server access logs enabled for auditing.  
• For extremely large partition sets, partition projection and read-only views to further  
reduce accidental scans.  
In simple words: I give each team a role that can only see and touch its own folders, I block  
public and unencrypted access at the bucket, and I control KMS keys so only approved roles  can read or write. This keeps access tight and easy to manage.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 

## 18. A partner needs temporary read-only access to certain S3 objects. How would you grant this securely (e.g., pre-signed URLs, access points)?

I choose the simplest secure method based on the use case duration and scale. For one-off or  small batches, I use short-lived pre-signed URLs. For ongoing access, I set up a cross-account  © Shubham Wadekar
role or an access point with a tight policy. In all cases, I make sure KMS permissions match.  Option 1: pre-signed URLs (fastest for small, short-term sharing)  
• When to use: a few files, access for hours or days, minimal setup.  
• How I do it:  
• Generate pre-signed GET URLs with a short expiry (for example, 1–24 hours).  
• If using SSE-KMS, the signing role must have kms:Decrypt permission; the URL  
will work without exposing KMS keys to the partner.  
• Send the links over a secure channel. Rotate them if needed.  
• Pros: no partner IAM setup, time-limited, easy to revoke by rotating objects or expiring  
URLs.  
• Cons: not ideal for thousands of objects or long-term access.  
Option 2: cross-account IAM role the partner can assume (best for ongoing integrations)  
• When to use: repeated access, automation, many objects.  
• How I do it:  
• In my account, create role PartnerReadOnlyRole with a trust policy allowing the  
partner’s AWS account to assume the role (use an external ID to prevent  
confused-deputy issues).  
• Attach an IAM policy that allows s3:ListBucket (scoped by prefix) and  
s3:GetObject only on the allowed prefixes, for example  
arn:aws:s3:::lake/exports/partnerX/*.  
• Update the bucket policy to allow that role to GetObject and List with the same  
prefix scope.  
• If encrypted with KMS, allow that role kms:Decrypt on the lake’s KMS key with a  
condition limiting it to the bucket and prefix.  
• Partner configures an IAM role or user in their account to assume  
PartnerReadOnlyRole and uses temporary credentials.  
• Pros: least-privilege, auditable, easy to turn off by removing trust or policy.  
• Cons: initial setup coordination.  
Option 3: S3 Access Point for the partner (clean scoping, can be VPC-restricted)  
• When to use: you want a dedicated endpoint and policy per partner.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar 
• How I do it:  
• Create an access point pointing to the bucket, set its policy to allow  
s3:GetObject on the specific prefixes for the partner’s account or role. 
• Optionally restrict the access point to a specific partner VPC (if they connect via  
PrivateLink or have a peering setup).  
© Shubham Wadekar
• Ensure KMS permissions include the partner principal that uses the access  
point.  
• Pros: keeps bucket policy simple, easy to see and manage partner-specific access.  
• Cons: still requires KMS coordination and careful policy writing.  
Revocation and monitoring  
• To revoke access, remove the trust relationship (cross-account role), delete or change  
the access point policy, or let pre-signed URLs expire.  
• Enable S3 server access logs or CloudTrail data events to audit what the partner  
accessed.  
• Use object tags (for example, partner=partnerX) and scope policies to those tags for  
flexible control.  
Extra safeguards  
• Deny list in bucket policy for any action outside the approved prefixes or without TLS.  
• Set narrow expirations for pre-signed URLs and avoid sharing entire listings unless  
required.  
• If the data is sensitive, package it under an “exports” prefix so nothing else is ever  
exposed by mistake.  
In simple words: for quick, short-term sharing I use pre-signed URLs. For ongoing read-only  access, I create a cross-account role or an access point that can only read a specific folder,  and I make sure KMS allows just that role. It’s easy to audit and easy to turn off.  
All rights reserved. Personal use only. Redistribution or resale is prohibited © Shubham Wadekar