# General Guide: Configure Smartbus for Splunk Indexers on AWS

## Purpose

This guide provides a generalized process for configuring Splunk Smartbus with AWS SQS and S3 for Splunk indexers.

Smartbus uses two AWS services:

```text
SQS = queue metadata / references
S3  = large message store for ingestion blobs
```

The SQS queue stores metadata that points to the actual ingestion blob stored in S3. For clustered indexers, deploy the Splunk configuration from the **Cluster Manager** so it is pushed to the indexers under `peer-apps`.

---

## 1. AWS Prerequisites

Create or identify the following AWS resources:

```text
SQS queue
S3 bucket or S3 prefix for the large message store
Optional SQS dead-letter queue
Optional KMS key or SSE-C encryption material
IAM role or IAM user with SQS and S3 permissions
```

Recommended for production:

```text
Use one SQS queue per indexer site.
Use one S3 bucket or prefix per indexer site.
Use EC2 IAM roles instead of static access keys where possible.
Use encryption for the large message store.
Configure a dead-letter queue.
```

---

## 2. Required IAM Permissions

Attach permissions to the EC2 instance role or IAM user used by the indexers.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SmartbusSQS",
      "Effect": "Allow",
      "Action": [
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes",
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:<region>:<account-id>:<queue-name>"
    },
    {
      "Sid": "SmartbusS3Bucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::<bucket-name>"
    },
    {
      "Sid": "SmartbusS3Objects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::<bucket-name>/<prefix>/*"
    }
  ]
}
```

If using KMS, also allow:

```json
{
  "Sid": "SmartbusKMS",
  "Effect": "Allow",
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "arn:aws:kms:<region>:<account-id>:key/<key-id>"
}
```

---

## 3. Create a Smartbus App on the Cluster Manager

Run on the **Cluster Manager**:

```bash
sudo mkdir -p /opt/splunk/etc/manager-apps/smartbus_aws/local
```

---

## 4. Configure `outputs.conf`

Use this when the indexers need to send data into Smartbus.

```bash
sudo tee /opt/splunk/etc/manager-apps/smartbus_aws/local/outputs.conf >/dev/null <<'EOF'
[remote_queue:<queue-name>]
remote_queue.type = sqs_smartbus
remote_queue.sqs_smartbus.encoding_format = s2s
remote_queue.sqs_smartbus.auth_region = <aws-region>
remote_queue.sqs_smartbus.endpoint = https://sqs.<aws-region>.amazonaws.com
remote_queue.sqs_smartbus.large_message_store.endpoint = https://s3.<aws-region>.amazonaws.com
remote_queue.sqs_smartbus.large_message_store.path = s3://<bucket-name>/<prefix>
remote_queue.sqs_smartbus.max_count.max_retries_per_part = 3
remote_queue.sqs_smartbus.retry_policy = max_count
EOF
```

If you are **not** using an EC2 IAM role, add these to the stanza:

```conf
remote_queue.sqs_smartbus.access_key = <access-key>
remote_queue.sqs_smartbus.secret_key = <secret-key>
```

If using IAM-based authentication, do not include the static access key and secret key.

---

## 5. Configure `inputs.conf`

Use this when the indexers need to consume from Smartbus.

```bash
sudo tee /opt/splunk/etc/manager-apps/smartbus_aws/local/inputs.conf >/dev/null <<'EOF'
[remote_queue:<queue-name>]
remote_queue.type = sqs_smartbus
remote_queue.sqs_smartbus.auth_region = <aws-region>
remote_queue.sqs_smartbus.endpoint = https://sqs.<aws-region>.amazonaws.com
remote_queue.sqs_smartbus.large_message_store.endpoint = https://s3.<aws-region>.amazonaws.com
remote_queue.sqs_smartbus.large_message_store.path = s3://<bucket-name>/<prefix>
remote_queue.sqs_smartbus.max_count.max_retries_per_part = 3
EOF
```

With a dead-letter queue:

```conf
remote_queue.sqs_smartbus.dead_letter_queue.name = <dlq-queue-name>
```

With static credentials:

```conf
remote_queue.sqs_smartbus.access_key = <access-key>
remote_queue.sqs_smartbus.secret_key = <secret-key>
```

---

## 6. Enable the Remote Queue Pipeline

Create `default-mode.conf`:

```bash
sudo tee /opt/splunk/etc/manager-apps/smartbus_aws/local/default-mode.conf >/dev/null <<'EOF'
[pipeline:remotequeueruleset]
disabled = false

[pipeline:ruleset]
disabled = true

[pipeline:remotequeuetyping]
disabled = false

[pipeline:remotequeueoutput]
disabled = false

[pipeline:typing]
disabled = true
EOF
```

---

## 7. Push the Cluster Bundle

Run on the **Cluster Manager**:

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/manager-apps/smartbus_aws
sudo -u splunk /opt/splunk/bin/splunk validate cluster-bundle
sudo -u splunk /opt/splunk/bin/splunk apply cluster-bundle
```

---

## 8. Verify Config on Each Indexer

Run on each indexer:

```bash
sudo -u splunk /opt/splunk/bin/splunk btool outputs list remote_queue:<queue-name> --debug
sudo -u splunk /opt/splunk/bin/splunk btool inputs list remote_queue:<queue-name> --debug
sudo -u splunk /opt/splunk/bin/splunk btool default-mode list --debug | grep -A3 remotequeue
```

Expected location after bundle push:

```text
/opt/splunk/etc/peer-apps/smartbus_aws/local/
```

Do **not** edit `peer-apps` directly. Make changes on the Cluster Manager under:

```text
/opt/splunk/etc/manager-apps/
```

---

## 9. Restart Indexers if Needed

```bash
sudo -u splunk /opt/splunk/bin/splunk restart
```

For a production indexer cluster, use a rolling restart through the Cluster Manager.

---

## 10. Enable Debug Logging

Run on each indexer when troubleshooting:

```bash
sudo -u splunk /opt/splunk/bin/splunk set log-level SQSSmartbusInputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQSSmartbusOutputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_Common -level DEBUG
```

Optional additional debug components:

```bash
sudo -u splunk /opt/splunk/bin/splunk set log-level SmartbusOutputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_OutputQueue -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_InputQueue -level DEBUG
```

---

## 11. Check Splunk Logs

```bash
grep -iE "smartbus|remote_queue|SQS|SQSSmartbus|RemoteQueue|s3|queue" \
/opt/splunk/var/log/splunk/splunkd.log | tail -200
```

Good signs:

```text
Successfully finished checking connectivity for remote queue=<queue-name>
Sending event data to object_uri=...
Sending storage-reference-message to ...
Peeked messages, msgCount=...
Finished downloading object ...
Parsing pdata done
```

---

## 12. Search Internal Metrics

```spl
index=_internal source=*metrics.log group=smartbus
```

Useful fields:

```text
queue_name
queue_type
successful_message_count
failed_message_count
event_count
raw_size_bytes
encoded_size_bytes
upload_service
upload_time_ms
```

---

## 13. Common Issues

### SQS Works but Ingestion Fails

Check S3 permissions. SQS only carries metadata; the actual data is stored in S3.

Check:

```text
S3 bucket policy
S3 object permissions
IAM role or IAM user permissions
KMS key policy, if applicable
VPC endpoint policy, if using S3 endpoints
```

### Queue Connection Failure

Check connectivity:

```bash
curl -I https://sqs.<aws-region>.amazonaws.com
curl -I https://s3.<aws-region>.amazonaws.com
```

Also verify:

```text
VPC routing
NAT gateway or VPC endpoints
Security groups
Network ACLs
IAM role attached to EC2
SQS queue policy
S3 bucket policy
KMS key policy
```

### Messages Going to DLQ

Check for common Smartbus ingestion errors:

```bash
grep -iE "DLQ|dead_letter|Invalid payload_size|Cannot register|Too many fields" \
/opt/splunk/var/log/splunk/splunkd.log | tail -200
```

Common DLQ-related causes include:

```text
Invalid payload_size
Cannot register new_channel
Too many fields
Malformed or oversized events
Improper line breaking
Problematic source data
```

### Static Credentials Not Working

Verify:

```text
Access key is active.
Secret key is correct.
IAM user has SQS permissions.
IAM user has S3 permissions.
IAM user has KMS permissions if encryption uses KMS.
No explicit deny exists in the bucket, queue, or key policy.
```

### IAM Role Not Working

Verify from the indexer:

```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

If IMDSv2 is required:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

## 14. Placeholder Reference

Replace these values before use:

```text
<queue-name>
<dlq-queue-name>
<aws-region>
<account-id>
<bucket-name>
<prefix>
<access-key>
<secret-key>
<key-id>
```

---

## 15. Minimal Checklist

```text
[ ] SQS queue created
[ ] Optional DLQ created
[ ] S3 bucket or prefix created
[ ] IAM role or user has SQS permissions
[ ] IAM role or user has S3 permissions
[ ] KMS permissions configured if needed
[ ] outputs.conf remote_queue stanza created
[ ] inputs.conf remote_queue stanza created
[ ] default-mode.conf remote queue pipelines enabled
[ ] Cluster bundle validated
[ ] Cluster bundle pushed
[ ] Config verified with btool on indexers
[ ] Debug logs checked
[ ] Smartbus metrics visible in _internal
```
