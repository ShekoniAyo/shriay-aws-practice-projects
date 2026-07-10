# SNS → SQS → Lambda → DynamoDB — Hands-On Practice

**Scenario:** PurePak Beverages (fictional FMCG distributor) publishes delivery-status events to SNS whenever a truck update happens. Multiple teams could subscribe to these events, but for this practice you'll build the core path: **SNS topic → SQS queue → Lambda → DynamoDB table**, so every delivery event is durably stored for querying later.

Why this pattern instead of SNS → Lambda directly: SQS sits in the middle as a **buffer**. If the Lambda is slow, erroring, or throttled, messages wait safely in the queue and retry, instead of being lost or overwhelming the function. It's the standard "reliable ingestion" pattern.

---

## 1. Create the DynamoDB table

1. Go to **DynamoDB → Tables → Create table**
2. Table name: `DeliveryEvents`
3. Partition key: `eventId` (String)
4. Leave everything else as default (on-demand capacity is fine for practice)
5. Create the table

## 2. Create the SNS topic

1. Go to **SNS → Topics → Create topic**
2. Type: **Standard**
3. Name: `DeliveryStatusTopic`
4. Create the topic and copy its **ARN** — you'll need it in Step 5

## 3. Create the SQS queue

1. Go to **SQS → Queues → Create queue**
2. Type: **Standard**
3. Name: `DeliveryStatusQueue`
4. Under **Access policy**, leave default for now (you'll grant SNS access via the subscription in the next step)
5. Create the queue and copy its **ARN**

## 4. Subscribe the queue to the topic

1. Open `DeliveryStatusTopic` → **Create subscription**
2. Protocol: `Amazon SQS`
3. Endpoint: paste the `DeliveryStatusQueue` ARN
4. **Enable raw message delivery** — this is important. Without it, your Lambda would receive the message double-wrapped (SQS body containing the full SNS envelope). With it enabled, the SQS message body is exactly what was published to SNS.
5. Create subscription. AWS automatically updates the SQS queue's access policy to allow this specific topic to send it messages — you can verify this under the queue's **Access policy** tab.

## 5. Create the Lambda function

1. Go to **Lambda → Create function**
2. Name: `WriteDeliveryEventToDynamoDBFunction`
3. Runtime: Python 3.12
4. Create the function

### Lambda code

```python
import json
import os
import boto3
import logging
from datetime import datetime, timezone

logger = logging.getLogger()
logger.setLevel(logging.INFO)

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    batch_item_failures = []

    for record in event['Records']:
        message_id = record['messageId']

        try:
            # Raw message delivery is enabled, so record['body'] is exactly
            # what was published to SNS — no envelope to unwrap.
            payload = json.loads(record['body'])

            event_id = payload['eventId']
            truck_id = payload['truckId']
            status = payload['status']
            location = payload.get('location', 'unknown')

            table.put_item(
                Item={
                    'eventId': event_id,
                    'truckId': truck_id,
                    'status': status,
                    'location': location,
                    'receivedAt': datetime.now(timezone.utc).isoformat()
                }
            )

            logger.info(f"Wrote event {event_id} to DynamoDB")

        except KeyError as e:
            # Message is missing a required field — don't retry, it will
            # never succeed. Log it and move on rather than blocking the batch.
            logger.error(f"Message {message_id} missing required field: {e}")

        except Exception as e:
            # Anything else (throttling, transient DynamoDB error) — mark this
            # specific message as failed so SQS retries only this one, not the
            # whole batch.
            logger.error(f"Failed to process message {message_id}: {e}")
            batch_item_failures.append({'itemIdentifier': message_id})

    return {'batchItemFailures': batch_item_failures}
```

A few things worth noticing in this code, since they matter in real production systems:

- **`batchItemFailures`** — Lambda can process several SQS messages in one invocation (a batch). If you just `raise` an exception, the *entire batch* gets retried, including messages that already succeeded. Returning specific failed message IDs here means only the actual failures get retried. This requires **"Report batch item failures"** to be enabled on the event source mapping (Step 7).
- **`KeyError` is handled separately** — a malformed message will never succeed no matter how many times you retry it. Retrying it forever just wastes invocations; better to log it and consider routing genuinely broken messages to a Dead Letter Queue for manual inspection.

## 6. Configure the Lambda's environment and permissions

1. **Environment variable:** `TABLE_NAME` = `DeliveryEvents`
2. **Execution role permissions:** attach a policy allowing:
   - `dynamodb:PutItem` on the `DeliveryEvents` table
   - `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes` on `DeliveryStatusQueue`

   For practice, you can attach the managed policies `AmazonDynamoDBFullAccess` and `AmazonSQSFullAccess`; for anything beyond practice, scope this down to the specific table and queue ARNs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/DeliveryEvents"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:DeliveryStatusQueue"
    }
  ]
}
```

## 7. Connect the queue as a Lambda trigger

1. In the Lambda function, go to **Add trigger**
2. Source: **SQS**
3. Queue: `DeliveryStatusQueue`
4. Batch size: `10` (default — fine for practice)
5. Expand **Additional settings** and enable **Report batch item failures** — this is what makes the `batchItemFailures` return value in the code actually work
6. Add the trigger

## 8. (Optional but recommended) Add a Dead Letter Queue

1. Create a second SQS queue: `DeliveryStatusDLQ`
2. On `DeliveryStatusQueue`, go to **Edit → Dead-letter queue**
3. Enable it, select `DeliveryStatusDLQ`, and set **Maximum receives** to `3`
4. Now, if a message fails processing 3 times, it moves to the DLQ instead of retrying forever — giving you a place to inspect messages that are genuinely broken, without them clogging the main queue.

## 9. Test the full pipeline

1. Go to the `DeliveryStatusTopic` in SNS → **Publish message**
2. Message body:
```json
{
  "eventId": "EVT-5001",
  "truckId": "TRK-22",
  "status": "IN_TRANSIT",
  "location": "Ibadan"
}
```
3. Publish the message
4. Within a few seconds, check the `DeliveryStatusQueue` — it should briefly show 1 message, then drop to 0 once Lambda consumes it (SQS auto-deletes the message once Lambda returns successfully)
5. Go to **DynamoDB → DeliveryEvents → Explore table items** — you should see the new item with `eventId: EVT-5001` and a `receivedAt` timestamp

## 10. Test the failure path

1. Publish a deliberately broken message (missing `eventId`):
```json
{
  "truckId": "TRK-23",
  "status": "DELAYED"
}
```
2. Check **CloudWatch Logs** for the Lambda — you should see the `KeyError` logged
3. Confirm nothing was written to DynamoDB for this message, and that it was **not** endlessly retried (since `KeyError` is handled without adding it to `batchItemFailures`)

## 11. Test the retry path (optional)

1. Temporarily edit the Lambda code to force an exception for testing:
```python
raise Exception("Simulated transient failure")
```
2. Publish a valid message and watch it retry — check the queue's **Messages available** count, which should briefly increase as SQS makes the message visible again after the visibility timeout
3. After 3 failed attempts, confirm it lands in `DeliveryStatusDLQ`
4. Revert the code change afterward

---

## What you've built

A durable ingestion pipeline: **SNS** broadcasts delivery events, **SQS** buffers them so nothing is lost if Lambda is briefly unavailable, **Lambda** writes each event into **DynamoDB** with per-message failure isolation via `batchItemFailures`, and a **DLQ** catches anything that repeatedly fails so it doesn't retry forever or block the queue.

**Suggested next extension:** add a second SQS queue subscribed to the same SNS topic (a second fan-out branch) that only receives `status: DELAYED` events via a filter policy, feeding a separate Lambda that sends an alert — combining this practice with the fan-out pattern from the previous one.
