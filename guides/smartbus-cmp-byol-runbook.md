# Smartbus for CMP BYOL - AWS Runbook

## Purpose

This guide explains how to configure and troubleshoot Smartbus for a Splunk CMP BYOL deployment on AWS.

Smartbus uses two AWS services:

- **Amazon SQS**: stores queue messages that contain metadata and references.
- **Amazon S3**: stores the actual ingestion blobs, also called the large message store.

The SQS queue does not hold the full event payload. It holds a reference to the ingestion blob stored in S3.

---

## Target Audience

- Support engineers
- TechOps engineers
- Developers
- Splunk administrators managing CMP BYOL environments

---

## Scope

### In scope

- Configuring Smartbus for CMP BYOL on AWS
- Troubleshooting Smartbus for CMP BYOL on AWS
- Reviewing Smartbus input/output worker behavior
- Reviewing Smartbus metrics and DLQ behavior

### Out of scope

- Non-BYOL Splunk Cloud Smartbus operations
- Azure Smartbus configuration
- SmartStore warm bucket configuration

---

## Architecture Overview

Smartbus has two main flows:

1. **Output worker flow**
   - Consumes data from the ingestion queue.
   - Serializes the data.
   - Uploads the serialized blob to S3.
   - Sends a storage-reference message to SQS.

2. **Input worker flow**
   - Polls SQS for storage-reference messages.
   - Downloads the referenced ingestion blob from S3.
   - Deserializes the blob.
   - Sends the data to the Splunk indexing queue.
   - Deletes the SQS message and S3 object after successful processing.

---

## AWS Prerequisites

Create or identify the following AWS resources:

- SQS queue for Smartbus
- Optional SQS dead-letter queue
- S3 bucket or S3 prefix for the large message store
- IAM role or IAM user with SQS and S3 permissions
- Optional KMS key or SSE-C configuration for encryption

For multi-site indexer clusters, use separate queues and large message store paths per site where possible.

---

## Configure SQS Queue

1. Go to **Amazon SQS**.
2. Create a new queue.
3. Choose the queue type required by your deployment.
4. Set the queue name.
5. Configure visibility timeout, message retention, and receive settings.
6. Configure encryption.
7. Configure access policy.
8. Configure a dead-letter queue if required.
9. Create the queue.

Recommended production settings:

- Use encryption.
- Configure a DLQ.
- Use a visibility timeout long enough for normal blob download, decoding, and indexing.
- Use IAM roles instead of static access keys when possible.

---

## Configure S3 Large Message Store

Create a dedicated S3 bucket or prefix for Smartbus ingestion blobs.

Recommended layout:

```text
s3://<bucket-name>/<smartbus-prefix>
```

For multi-site indexer clusters, use separate prefixes or buckets per site.

Example:

```text
s3://<bucket-name>/smartbus/site1
s3://<bucket-name>/smartbus/site2
```

The large message store contains customer event data in blob form, so it should be treated as sensitive.

---

## Encryption Guidance

The original runbook recommends strong encryption for the large message store because ingestion blobs contain actual customer event data.

Supported patterns include:

- S3 server-side encryption
- SSE-KMS
- SSE-C, where supported and required

If KMS is used, configure the key policy and IAM permissions so the Splunk indexers can encrypt, decrypt, and generate data keys.

---

## IAM Permissions

Attach permissions to the EC2 instance role or IAM user used by Splunk.

### SQS permissions

```json
{
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
}
```

### S3 bucket permissions

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:ListBucket"
  ],
  "Resource": "arn:aws:s3:::<bucket-name>"
}
```

### S3 object permissions

```json
{
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
```

### KMS permissions, if using KMS

```json
{
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

## Splunk Configuration Overview

Smartbus requires a matching `remote_queue` stanza in:

- `outputs.conf`
- `inputs.conf`

It also requires the remote queue pipeline to be enabled in:

- `default-mode.conf`

For indexer clusters, deploy these settings from the Cluster Manager under:

```text
$SPLUNK_HOME/etc/manager-apps/<app-name>/local/
```

After bundle push, the settings appear on indexers under:

```text
$SPLUNK_HOME/etc/peer-apps/<app-name>/local/
```

Do not edit `peer-apps` directly.

---

## Create the Cluster Manager App

Run on the Cluster Manager:

```bash
sudo mkdir -p /opt/splunk/etc/manager-apps/smartbus_aws/local
```

---

## Configure `outputs.conf`

Use `outputs.conf` when Splunk needs to send data to Smartbus.

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

### Static credential option

If you are not using an EC2 IAM role, add the following to the stanza:

```conf
remote_queue.sqs_smartbus.access_key = <access-key>
remote_queue.sqs_smartbus.secret_key = <secret-key>
```

### Encryption option

If using SSE-C or KMS-related Smartbus encryption settings, include the appropriate large message store settings:

```conf
remote_queue.sqs_smartbus.large_message_store.encryption_scheme = <encryption-scheme>
remote_queue.sqs_smartbus.large_message_store.key_id = <key-id>
remote_queue.sqs_smartbus.large_message_store.key_refresh_interval = 24h
remote_queue.sqs_smartbus.large_message_store.kms_endpoint = https://kms.<aws-region>.amazonaws.com
```

---

## Configure `inputs.conf`

Use `inputs.conf` when Splunk needs to consume data from Smartbus.

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

### Static credential option

```conf
remote_queue.sqs_smartbus.access_key = <access-key>
remote_queue.sqs_smartbus.secret_key = <secret-key>
```

### Dead-letter queue option

```conf
remote_queue.sqs_smartbus.dead_letter_queue.name = <dlq-queue-name>
```

### Visibility timeout option

The default visibility timeout is typically 60 seconds. Adjust if workers need more time to process messages.

```conf
remote_queue.sqs_smartbus.timeout.visibility = 60
```

---

## Enable the Remote Queue Pipeline

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

## Push the Cluster Bundle

Run on the Cluster Manager:

```bash
sudo chown -R splunk:splunk /opt/splunk/etc/manager-apps/smartbus_aws
sudo -u splunk /opt/splunk/bin/splunk validate cluster-bundle
sudo -u splunk /opt/splunk/bin/splunk apply cluster-bundle
```

Use a rolling restart if Splunk indicates one is required.

---

## Verify Configuration on Indexers

Run on each indexer:

```bash
sudo -u splunk /opt/splunk/bin/splunk btool outputs list remote_queue:<queue-name> --debug
sudo -u splunk /opt/splunk/bin/splunk btool inputs list remote_queue:<queue-name> --debug
sudo -u splunk /opt/splunk/bin/splunk btool default-mode list --debug | grep -A3 remotequeue
```

Confirm that the configuration is coming from:

```text
/opt/splunk/etc/peer-apps/smartbus_aws/local/
```

---

## Smartbus Output Worker

The output worker sends data from Splunk to Smartbus.

### Output worker flow

1. Connect to the SQS queue.
2. Consume data from the ingestion queue.
3. Serialize the data.
4. Upload the blob to S3.
5. Send a storage-reference message to SQS.
6. Record metrics.

### Enable output worker debugging

```bash
sudo -u splunk /opt/splunk/bin/splunk set log-level SQSSmartbusOutputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SmartbusOutputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_OutputQueue -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_Common -level DEBUG
```

### Output worker success indicators

Look for these patterns in `splunkd.log`:

```text
Successfully finished checking connectivity for remote queue=<queue-name>
SendMessageTimeout tick
Sending event data to object_uri=...
Sending storage-reference-message to ... result=true
```

### Output worker metrics

```spl
index=_internal source=*metrics.log group=smartbus name=ingestor
```

Useful metric series:

- `encode`
- `message_thruput`
- `per_message_stats`
- `receipt_upload`
- `upload` with `upload_type=datastore`
- `upload` with `upload_type=bus`

Useful fields:

- `queue_name`
- `queue_type`
- `successful_message_count`
- `failed_message_count`
- `event_count`
- `raw_size_bytes`
- `encoded_size_bytes`
- `upload_size_bytes`
- `upload_time_ms`
- `upload_service`

---

## Smartbus Input Worker

The input worker receives data from Smartbus and sends it to the indexing queue.

### Input worker flow

1. Connect to the SQS queue.
2. Poll or peek messages from SQS.
3. Parse the SQS message to obtain the S3 object URI.
4. Download the blob from S3.
5. Deserialize the blob into `pData`.
6. Send the data to the indexing queue.
7. Account for committed events.
8. Delete the SQS message and S3 object after successful processing.
9. Renew message visibility when needed.
10. Send failed messages to DLQ paths when required.

### Enable input worker debugging

```bash
sudo -u splunk /opt/splunk/bin/splunk set log-level SQSSmartbusInputWorker -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_InputQueue -level DEBUG
sudo -u splunk /opt/splunk/bin/splunk set log-level SQS_Common -level DEBUG
```

### Input worker success indicators

Look for these patterns in `splunkd.log`:

```text
Peeked messages, msgCount=<count>
Received event reference from SQS
Finished downloading object ... status=Y
Parsing pdata done
Finished getting object, success=1
Accounted for committed events
Deleting object ... success=Y
```

### Input worker metrics

```spl
index=_internal source=*metrics.log group=smartbus name=indexer
```

Useful metric series:

- `download` with `download_type=bus`
- `decode`
- `message_thruput`
- `per_message_stats`
- `ack`
- `receipt_download`

Useful fields:

- `queue_name`
- `queue_type`
- `successful_message_count`
- `failed_message_count`
- `event_count`
- `raw_size_bytes`
- `decode_size_bytes`
- `decoding_time_ms`
- `bus_download_bytes`
- `datastore_download_bytes`
- `indexing_time_ms`

---

## Message Renewal

SQS uses a visibility timeout. When a worker receives a message, the message becomes invisible to other workers for the visibility timeout period.

If processing takes longer than the visibility timeout, the worker should renew the message before the timeout expires.

Typical behavior:

- Default timeout: `60` seconds
- Renewal occurs shortly before expiration
- Example: for a 60-second timeout, renewal may occur around 50 seconds after receipt

Configure timeout if needed:

```conf
remote_queue.sqs_smartbus.timeout.visibility = 60
```

Look for renew logs:

```text
Successfully renewed message with message_id=...
```

Repeated renewals can indicate slow or stuck indexing.

---

## Dead Letter Queue Behavior

Messages may be sent to a DLQ path when they cannot be processed successfully.

Common DLQ categories:

- Blob decode or parsing failure
- Repeated renewals caused by slow or stuck indexing
- Repeated processing failures until the queue's `maxReceiveCount` threshold is reached

Common S3 DLQ paths:

```text
<large-message-store-path>/dlq_decode_failure/
<large-message-store-path>/dlq_repeat_renew/
<large-message-store-path>/dlq/
```

### DLQ log patterns

```text
Successfully sent a SQS DLQ message to S3 with location=...
message_type=message decoding failed
```

```text
Successfully sent a SQS DLQ message to S3 with location=...
message_type=message renewed repeatedly
```

Messages that land in DLQ should be analyzed later and may be repaired or re-ingested.

---

## Common Configuration Issues

### Splunkd crashes during startup

A bad Smartbus configuration may cause pre-flight failure during initialization.

Look for:

```text
NOAH INTENTIONAL CRASH noahCrashReason="Phase-1 pre-flight checks failed when configuring the remote queue input worker"
```

Check:

- `inputs.conf` syntax
- `outputs.conf` syntax
- `default-mode.conf` syntax
- Queue name
- AWS region
- SQS endpoint
- S3 endpoint
- S3 path
- Credentials or IAM role
- KMS or SSE-C settings

### Cannot connect to SQS

Check:

```bash
grep -iE "SQS|remote queue|GetQueueUrl|connectivity|error|failed" /opt/splunk/var/log/splunk/splunkd.log | tail -200
```

Validate:

- Queue exists
- Region is correct
- Endpoint is correct
- IAM role or credentials are valid
- SQS queue policy allows access
- Network path to SQS exists through NAT gateway, proxy, or VPC endpoint

### Cannot upload or download S3 blob

Check:

```bash
grep -iE "S3|object_uri|large_message_store|download|upload|error|failed" /opt/splunk/var/log/splunk/splunkd.log | tail -200
```

Validate:

- Bucket exists
- Prefix exists or can be created
- IAM permissions allow object operations
- Bucket policy allows access
- KMS key policy allows access, if using KMS
- Encryption settings match the bucket/prefix requirements
- Network path to S3 exists through NAT gateway, proxy, or VPC endpoint

### Messages go to DLQ

Check:

```bash
grep -iE "DLQ|dead_letter|Invalid payload_size|Cannot register|Too many fields|decode_failure|repeat_renew" /opt/splunk/var/log/splunk/splunkd.log | tail -200
```

Typical causes:

- Invalid or corrupt blob
- Blob parsing failure
- Encoding/decoding mismatch
- Very large events
- Excessive fields
- Slow or blocked indexing queue
- Visibility timeout too short
- Insufficient permissions to delete processed queue messages or objects

---

## Useful Searches

### All Smartbus metrics

```spl
index=_internal source=*metrics.log group=smartbus
```

### Output worker metrics

```spl
index=_internal source=*metrics.log group=smartbus name=ingestor
| stats sum(successful_message_count) as successful_messages
        sum(failed_message_count) as failed_messages
        sum(event_count) as events
        sum(raw_size_bytes) as raw_bytes
        sum(encoded_size_bytes) as encoded_bytes
  by queue_name series upload_service
```

### Input worker metrics

```spl
index=_internal source=*metrics.log group=smartbus name=indexer
| stats sum(successful_message_count) as successful_messages
        sum(failed_message_count) as failed_messages
        sum(event_count) as events
        sum(raw_size_bytes) as raw_bytes
        sum(decode_size_bytes) as decoded_bytes
  by queue_name series download_service
```

### DLQ-related logs

```spl
index=_internal source=*splunkd.log* SQSSmartbusInputWorker
("DLQ" OR "dead letter" OR "dlq_decode_failure" OR "dlq_repeat_renew")
```

### Queue connectivity logs

```spl
index=_internal source=*splunkd.log* (SQS_Common OR RemoteQueueInputWorker OR RemoteQueueOutputWorker)
("Successfully finished checking connectivity" OR "GetQueueUrl" OR error OR failed)
```

### Blob transfer logs

```spl
index=_internal source=*splunkd.log* (SQSSmartbusInputWorker OR SQSSmartbusOutputWorker)
("object_uri" OR "Sending event data" OR "Finished downloading object" OR "Parsing pdata done")
```

---

## Placeholder Reference

Replace the following values before use:

```text
<queue-name>
<dlq-queue-name>
<aws-region>
<account-id>
<bucket-name>
<prefix>
<access-key>
<secret-key>
<encryption-scheme>
<key-id>
```

---

## Quick Validation Checklist

### AWS

- [ ] SQS queue exists.
- [ ] DLQ exists, if required.
- [ ] S3 bucket/prefix exists.
- [ ] IAM role or user has SQS permissions.
- [ ] IAM role or user has S3 permissions.
- [ ] KMS permissions exist, if applicable.
- [ ] SQS queue policy allows access.
- [ ] S3 bucket policy allows access.
- [ ] Network access to SQS and S3 exists.

### Splunk

- [ ] `outputs.conf` has the correct `remote_queue` stanza.
- [ ] `inputs.conf` has the correct `remote_queue` stanza.
- [ ] `default-mode.conf` enables the remote queue pipeline.
- [ ] Cluster bundle validates successfully.
- [ ] Cluster bundle applies successfully.
- [ ] Indexers show the config under `peer-apps`.
- [ ] Smartbus debug logs show successful queue connectivity.
- [ ] Smartbus metrics appear in `_internal`.
- [ ] No DLQ errors are appearing unexpectedly.

---

## Notes

- Prefer IAM roles on EC2 over static access keys.
- Do not store secrets in shared documentation.
- Do not edit Smartbus app files directly under `peer-apps` on indexers.
- Keep SQS, S3, and KMS resources in the same region unless there is a specific design reason not to.
- Treat the large message store as sensitive because it contains actual event payloads.
