# Serverless Order Tracking App — Hands-On Practice

**Scenario:** PurePak Beverages needs a real order intake form: customers submit an order, it's validated, queued, processed asynchronously, and — unlike a bare "Order submitted!" alert — the customer can actually watch its status change from `PENDING` to `COMPLETED` in real time. An SNS notification fires when processing finishes, ready to plug into email/ops alerts later.

**Architecture:** `Webpage → API Gateway → SubmitOrderFunction → DynamoDB + SQS → ProcessOrderFunction → DynamoDB + SNS`, with a second API route (`GetOrderStatusFunction`) the webpage polls to show live status.

---

# Part 1 — Data layer and queue

## 1. Create the DynamoDB table

1. **DynamoDB → Create table**
2. Table name: `ProductOrders`
3. Partition key: `orderId` (String)
4. Create the table

## 2. Create the SQS queue

1. **SQS → Create queue**
2. Type: Standard, Name: `ProductOrdersQueue`
3. Create the queue and copy its **URL** and **ARN**

## 3. Create the SNS topic

1. **SNS → Create topic**
2. Type: Standard, Name: `OrderProcessedTopic`
3. Create the topic and copy its **ARN**
4. Optional: subscribe your own email to it (**Create subscription → Email**) so you can see the notification land during testing — remember to confirm the subscription from your inbox

---

# Part 2 — Lambda functions

## 1. SubmitOrderFunction

Handles the incoming order: validates it, writes an initial `PENDING` record to DynamoDB, and queues it for processing.

1. **Lambda → Create function**
   - Name: `SubmitOrderFunction`
   - Runtime: Python 3.12
2. Code:

```python
import json
import os
import uuid
import boto3
from datetime import datetime, timezone

sqs = boto3.client('sqs')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])
QUEUE_URL = os.environ['QUEUE_URL']

CORS_HEADERS = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Headers': 'Content-Type',
    'Access-Control-Allow-Methods': 'OPTIONS,POST'
}

def lambda_handler(event, context):
    try:
        body = json.loads(event.get('body') or '{}')
        product_name = (body.get('productName') or '').strip()
        quantity = body.get('quantity')

        errors = []
        if not product_name:
            errors.append('productName is required')
        if not isinstance(quantity, (int, float)) or quantity <= 0:
            errors.append('quantity must be a positive number')

        if errors:
            return respond(400, {'errors': errors})

        order_id = str(uuid.uuid4())
        now = datetime.now(timezone.utc).isoformat()

        table.put_item(Item={
            'orderId': order_id,
            'productName': product_name,
            'quantity': int(quantity),
            'status': 'PENDING',
            'createdAt': now,
            'updatedAt': now
        })

        sqs.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps({
                'orderId': order_id,
                'productName': product_name,
                'quantity': int(quantity)
            })
        )

        return respond(200, {'orderId': order_id, 'status': 'PENDING'})

    except Exception as e:
        return respond(500, {'error': str(e)})

def respond(status_code, body):
    return {
        'statusCode': status_code,
        'headers': CORS_HEADERS,
        'body': json.dumps(body)
    }
```

3. Environment variables: `TABLE_NAME` = `ProductOrders`, `QUEUE_URL` = the queue URL from Part 1
4. Execution role permissions needed: `dynamodb:PutItem` on `ProductOrders`, `sqs:SendMessage` on `ProductOrdersQueue` (managed policies `AmazonDynamoDBFullAccess` + `AmazonSQSFullAccess` are fine for practice)
5. Deploy

## 2. ProcessOrderFunction

Triggered by SQS: marks the order `COMPLETED` and publishes to SNS.

1. **Lambda → Create function**
   - Name: `ProcessOrderFunction`
   - Runtime: Python 3.12
2. Code:

```python
import json
import os
import boto3
from datetime import datetime, timezone

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])
sns = boto3.client('sns')
TOPIC_ARN = os.environ['TOPIC_ARN']

def lambda_handler(event, context):
    batch_item_failures = []

    for record in event['Records']:
        message_id = record['messageId']
        try:
            payload = json.loads(record['body'])
            order_id = payload['orderId']

            table.update_item(
                Key={'orderId': order_id},
                UpdateExpression='SET #s = :status, updatedAt = :updatedAt',
                ExpressionAttributeNames={'#s': 'status'},
                ExpressionAttributeValues={
                    ':status': 'COMPLETED',
                    ':updatedAt': datetime.now(timezone.utc).isoformat()
                }
            )

            sns.publish(
                TopicArn=TOPIC_ARN,
                Subject='Order Processed',
                Message=json.dumps({
                    'orderId': order_id,
                    'productName': payload.get('productName'),
                    'quantity': payload.get('quantity'),
                    'status': 'COMPLETED'
                })
            )

        except Exception as e:
            print(f"Failed to process message {message_id}: {e}")
            batch_item_failures.append({'itemIdentifier': message_id})

    return {'batchItemFailures': batch_item_failures}
```

3. Environment variables: `TABLE_NAME` = `ProductOrders`, `TOPIC_ARN` = the topic ARN from Part 1
4. Execution role permissions: `dynamodb:UpdateItem` on `ProductOrders`, `sns:Publish` on `OrderProcessedTopic`, plus `sqs:ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes` on `ProductOrdersQueue`
5. Deploy
6. **Add trigger → SQS → ProductOrdersQueue**, batch size `10`, and under **Additional settings** enable **Report batch item failures**

## 3. GetOrderStatusFunction

Lets the webpage check on an order after submitting.

1. **Lambda → Create function**
   - Name: `GetOrderStatusFunction`
   - Runtime: Python 3.12
2. Code:

```python
import json
import os
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

CORS_HEADERS = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Headers': 'Content-Type',
    'Access-Control-Allow-Methods': 'OPTIONS,GET'
}

def lambda_handler(event, context):
    order_id = (event.get('pathParameters') or {}).get('orderId')
    if not order_id:
        return respond(400, {'error': 'orderId is required'})

    response = table.get_item(Key={'orderId': order_id})
    item = response.get('Item')

    if not item:
        return respond(404, {'error': 'Order not found'})

    return respond(200, item)

def respond(status_code, body):
    return {
        'statusCode': status_code,
        'headers': CORS_HEADERS,
        'body': json.dumps(body, default=str)
    }
```

3. Environment variable: `TABLE_NAME` = `ProductOrders`
4. Execution role permissions: `dynamodb:GetItem` on `ProductOrders`
5. Deploy

---

# Part 3 — API Gateway

1. **API Gateway → Create REST API**, name it `ProductOrdersAPI`
2. Create resource `/orders`, enable CORS
3. Add method `POST` on `/orders` → Lambda proxy integration → `SubmitOrderFunction`
4. Under `/orders`, create a child resource `/{orderId}`
5. Add method `GET` on `/orders/{orderId}` → Lambda proxy integration → `GetOrderStatusFunction`
6. Select `/orders` → **Actions → Enable CORS** → select all CORS options → confirm (do this for both the `/orders` and `/{orderId}` resources)
7. **Deploy API** to a new stage named `prod`
8. Copy the invoke URL — it will look like:
   `https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod`
   You'll use `<invoke-url>/orders` for submitting and `<invoke-url>/orders/{orderId}` for status checks.

---

# Part 4 — The webpage

This is a single self-contained `index.html` — industrial shipping-manifest design: warehouse-grey paper background, safety-orange accent, condensed headers, monospace order IDs, and a stamped status badge that updates live.

**Before uploading:** replace `YOUR_API_ENDPOINT` on the line near the top of the `<script>` block with your actual invoke URL from Part 3.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PurePak Beverages — Order Desk</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Oswald:wght@500;600;700&family=Inter:wght@400;500;600&family=IBM+Plex+Mono:wght@500;600&display=swap" rel="stylesheet">
<style>
  :root{
    --paper:#f2f4ef; --paper-line:#cdd6cc; --ink:#152420; --ink-soft:#54635b;
    --accent:#ff6a13; --accent-ink:#7a2e00;
    --pending:#c8892b; --pending-bg:#fbeed9;
    --done:#2f7d4f; --done-bg:#e2f3e6;
    --error:#b3261e; --error-bg:#fbe6e4;
    --card:#ffffff;
    --radius:10px;
  }
  *{box-sizing:border-box;}
  body{
    margin:0; background:var(--paper); color:var(--ink);
    font-family:'Inter',system-ui,sans-serif;
    background-image:
      linear-gradient(var(--paper-line) 1px, transparent 1px),
      linear-gradient(90deg, var(--paper-line) 1px, transparent 1px);
    background-size:34px 34px;
    background-position:-1px -1px;
    min-height:100vh;
  }
  header{
    padding:28px 24px 20px; text-align:center;
  }
  .eyebrow{
    font-family:'IBM Plex Mono',monospace; font-size:0.72rem; letter-spacing:0.14em;
    text-transform:uppercase; color:var(--ink-soft); margin-bottom:6px;
  }
  h1{
    font-family:'Oswald',sans-serif; font-weight:700; text-transform:uppercase;
    letter-spacing:0.01em; font-size:2.1rem; margin:0; color:var(--ink);
  }
  h1 span{color:var(--accent);}
  .subtitle{color:var(--ink-soft); font-size:0.95rem; margin-top:6px;}

  .layout{
    max-width:920px; margin:20px auto 60px; padding:0 20px;
    display:grid; grid-template-columns:1fr 1fr; gap:22px;
  }
  @media (max-width:760px){ .layout{grid-template-columns:1fr;} }

  .card{
    background:var(--card); border:1px solid var(--paper-line);
    border-radius:var(--radius); padding:26px 26px 28px;
    box-shadow:0 1px 0 rgba(21,36,32,0.03);
  }
  .card h2{
    font-family:'Oswald',sans-serif; font-size:1.05rem; text-transform:uppercase;
    letter-spacing:0.05em; margin:0 0 18px; color:var(--ink);
    border-bottom:2px solid var(--ink); padding-bottom:10px;
  }

  label{
    display:block; font-size:0.82rem; font-weight:600; margin:16px 0 6px;
    color:var(--ink-soft); text-transform:uppercase; letter-spacing:0.03em;
  }
  input{
    width:100%; padding:11px 12px; border:1.5px solid var(--paper-line);
    border-radius:7px; font-size:0.98rem; font-family:'Inter',sans-serif;
    background:#fbfcfa; color:var(--ink); transition:border-color 0.15s;
  }
  input:focus{ outline:none; border-color:var(--accent); background:#fff; }
  input.invalid{ border-color:var(--error); }
  .field-error{
    font-family:'IBM Plex Mono',monospace; font-size:0.76rem; color:var(--error);
    margin-top:5px; min-height:1em;
  }

  button{
    margin-top:22px; width:100%; padding:13px; border:none; border-radius:7px;
    background:var(--accent); color:#fff; font-family:'Oswald',sans-serif;
    font-size:1rem; font-weight:600; letter-spacing:0.04em; text-transform:uppercase;
    cursor:pointer; transition:background 0.15s, transform 0.1s;
    display:flex; align-items:center; justify-content:center; gap:10px;
  }
  button:hover{ background:var(--accent-ink); }
  button:active{ transform:scale(0.99); }
  button:disabled{ background:#c9ccc4; cursor:not-allowed; }

  .spinner{
    width:16px; height:16px; border-radius:50%;
    border:2.5px solid rgba(255,255,255,0.4); border-top-color:#fff;
    animation:spin 0.7s linear infinite; display:none;
  }
  button.loading .spinner{ display:inline-block; }
  @keyframes spin{ to{ transform:rotate(360deg); } }

  .banner{
    margin-top:16px; padding:10px 12px; border-radius:7px; font-size:0.88rem;
    display:none;
  }
  .banner.show{ display:block; }
  .banner.error{ background:var(--error-bg); color:var(--error); }

  /* Manifest / tracker card */
  .empty-state{
    color:var(--ink-soft); font-size:0.92rem; line-height:1.6; padding:10px 2px;
  }
  .manifest{ display:none; }
  .manifest.show{ display:block; }

  .stamp-row{ display:flex; align-items:center; justify-content:space-between; gap:16px; margin-bottom:18px;}
  .order-id{
    font-family:'IBM Plex Mono',monospace; font-size:0.82rem; color:var(--ink-soft);
    word-break:break-all;
  }
  .stamp{
    font-family:'Oswald',sans-serif; font-weight:700; font-size:0.85rem;
    letter-spacing:0.08em; text-transform:uppercase;
    padding:7px 14px; border-radius:5px; border:2.5px solid;
    transform:rotate(-3deg); white-space:nowrap;
    transition:all 0.3s ease;
  }
  .stamp.pending{ color:var(--pending); border-color:var(--pending); background:var(--pending-bg); }
  .stamp.completed{ color:var(--done); border-color:var(--done); background:var(--done-bg); transform:rotate(-3deg) scale(1.04); }

  .manifest-row{
    display:flex; justify-content:space-between; padding:9px 0;
    border-bottom:1px dashed var(--paper-line); font-size:0.9rem;
  }
  .manifest-row:last-child{ border-bottom:none; }
  .manifest-row .k{ color:var(--ink-soft); }
  .manifest-row .v{ font-weight:600; }

  .track-line{
    margin-top:20px; display:flex; align-items:center; gap:8px;
  }
  .dot{
    width:9px; height:9px; border-radius:50%; background:var(--paper-line); flex-shrink:0;
  }
  .dot.active{ background:var(--pending); animation:pulse 1.4s ease-in-out infinite; }
  .dot.done{ background:var(--done); animation:none; }
  @keyframes pulse{ 0%,100%{opacity:1;} 50%{opacity:0.35;} }
  .track-label{ font-family:'IBM Plex Mono',monospace; font-size:0.74rem; color:var(--ink-soft); }
  .track-fill{ flex:1; height:2px; background:var(--paper-line); position:relative; overflow:hidden; }
  .track-fill .fill{
    position:absolute; left:0; top:0; height:100%; background:var(--done);
    width:0%; transition:width 0.5s ease;
  }

  /* Lookup row */
  .lookup-row{
    display:flex; gap:8px; margin-bottom:6px;
  }
  .lookup-row input{
    flex:1; font-family:'IBM Plex Mono',monospace; font-size:0.85rem;
  }
  .lookup-row button{
    margin-top:0; width:auto; flex-shrink:0; padding:11px 18px;
    font-size:0.82rem;
  }
  .divider{
    border:none; border-top:1px dashed var(--paper-line); margin:20px 0 16px;
  }
</style>
</head>
<body>

<header>
  <div class="eyebrow">PurePak Beverages · Distribution Order Desk</div>
  <h1>Order <span>Manifest</span></h1>
  <div class="subtitle">Submit a stock order and watch it move from the queue to the warehouse.</div>
</header>

<div class="layout">

  <div class="card">
    <h2>01 · Submit Order</h2>
    <form id="order-form" novalidate>
      <label for="productName">Product</label>
      <input type="text" id="productName" placeholder="e.g. Sparkling Citrus 24-pack">
      <div class="field-error" id="productName-error"></div>

      <label for="quantity">Quantity (cases)</label>
      <input type="number" id="quantity" placeholder="e.g. 40" min="1">
      <div class="field-error" id="quantity-error"></div>

      <button type="submit" id="submit-btn">
        <span class="spinner"></span>
        <span id="btn-label">Submit Order</span>
      </button>

      <div class="banner error" id="submit-error"></div>
    </form>
  </div>

  <div class="card">
    <h2>02 · Track Status</h2>

    <label for="lookup-orderid">Look up an order</label>
    <div class="lookup-row">
      <input type="text" id="lookup-orderid" placeholder="Paste an Order ID">
      <button type="button" id="lookup-btn">Look Up</button>
    </div>
    <div class="banner error" id="lookup-error"></div>

    <hr class="divider">

    <div class="empty-state" id="empty-state">
      No order yet — submit one above or look one up by ID.
    </div>

    <div class="manifest" id="manifest">
      <div class="stamp-row">
        <div class="order-id" id="manifest-id"></div>
        <div class="stamp pending" id="stamp">Pending</div>
      </div>

      <div class="manifest-row"><span class="k">Product</span><span class="v" id="manifest-product">—</span></div>
      <div class="manifest-row"><span class="k">Quantity</span><span class="v" id="manifest-qty">—</span></div>
      <div class="manifest-row"><span class="k">Submitted</span><span class="v" id="manifest-created">—</span></div>

      <div class="track-line">
        <div class="dot active" id="dot-1"></div>
        <div class="track-fill"><div class="fill" id="track-fill"></div></div>
        <div class="dot" id="dot-2"></div>
      </div>
      <div class="track-line" style="margin-top:4px;">
        <div class="track-label">Queued</div>
        <div style="flex:1"></div>
        <div class="track-label" id="status-text">Processing…</div>
      </div>
    </div>
  </div>

</div>

<script>
  // Replace with your API Gateway invoke URL from Part 3, e.g.
  // https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod
  const API_BASE = 'YOUR_API_ENDPOINT';

  const form = document.getElementById('order-form');
  const submitBtn = document.getElementById('submit-btn');
  const btnLabel = document.getElementById('btn-label');
  const submitError = document.getElementById('submit-error');

  const emptyState = document.getElementById('empty-state');
  const manifest = document.getElementById('manifest');
  const manifestId = document.getElementById('manifest-id');
  const manifestProduct = document.getElementById('manifest-product');
  const manifestQty = document.getElementById('manifest-qty');
  const manifestCreated = document.getElementById('manifest-created');
  const stamp = document.getElementById('stamp');
  const trackFill = document.getElementById('track-fill');
  const dot2 = document.getElementById('dot-2');
  const statusText = document.getElementById('status-text');

  const lookupInput = document.getElementById('lookup-orderid');
  const lookupBtn = document.getElementById('lookup-btn');
  const lookupError = document.getElementById('lookup-error');

  let pollTimer = null;

  function setFieldError(fieldId, message) {
    document.getElementById(fieldId + '-error').textContent = message || '';
    document.getElementById(fieldId).classList.toggle('invalid', !!message);
  }

  function validate() {
    let ok = true;
    const productName = document.getElementById('productName').value.trim();
    const quantity = document.getElementById('quantity').value;

    if (!productName) {
      setFieldError('productName', 'Product name is required');
      ok = false;
    } else {
      setFieldError('productName', '');
    }

    if (!quantity || Number(quantity) <= 0) {
      setFieldError('quantity', 'Enter a quantity greater than 0');
      ok = false;
    } else {
      setFieldError('quantity', '');
    }

    return ok;
  }

  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    submitError.classList.remove('show');

    if (!validate()) return;

    const productName = document.getElementById('productName').value.trim();
    const quantity = Number(document.getElementById('quantity').value);

    submitBtn.disabled = true;
    submitBtn.classList.add('loading');
    btnLabel.textContent = 'Submitting…';

    try {
      const res = await fetch(`${API_BASE}/orders`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productName, quantity })
      });

      const data = await res.json();

      if (!res.ok) {
        const message = (data.errors && data.errors.join(', ')) || data.error || 'Something went wrong. Try again.';
        submitError.textContent = message;
        submitError.classList.add('show');
        return;
      }

      showManifest(data.orderId, productName, quantity, new Date().toISOString(), 'PENDING');
      startPolling(data.orderId);
      form.reset();

    } catch (err) {
      submitError.textContent = 'Could not reach the order service. Check your connection and try again.';
      submitError.classList.add('show');
    } finally {
      submitBtn.disabled = false;
      submitBtn.classList.remove('loading');
      btnLabel.textContent = 'Submit Order';
    }
  });

  lookupBtn.addEventListener('click', async () => {
    const orderId = lookupInput.value.trim();
    lookupError.classList.remove('show');

    if (!orderId) {
      lookupError.textContent = 'Enter an order ID to look up';
      lookupError.classList.add('show');
      return;
    }

    lookupBtn.disabled = true;
    lookupBtn.textContent = 'Looking up…';

    try {
      const res = await fetch(`${API_BASE}/orders/${orderId}`);
      const data = await res.json();

      if (!res.ok) {
        lookupError.textContent = data.error || 'Order not found. Double-check the ID.';
        lookupError.classList.add('show');
        return;
      }

      showManifest(data.orderId, data.productName, data.quantity, data.createdAt, data.status);

      if (data.status === 'PENDING') {
        startPolling(data.orderId);
      } else if (pollTimer) {
        clearInterval(pollTimer);
      }

    } catch (err) {
      lookupError.textContent = 'Could not reach the order service. Check your connection and try again.';
      lookupError.classList.add('show');
    } finally {
      lookupBtn.disabled = false;
      lookupBtn.textContent = 'Look Up';
    }
  });

  lookupInput.addEventListener('keydown', (e) => {
    if (e.key === 'Enter') lookupBtn.click();
  });

  function showManifest(orderId, productName, quantity, createdAt, status) {
    emptyState.style.display = 'none';
    manifest.classList.add('show');
    manifestId.textContent = orderId;
    manifestProduct.textContent = productName;
    manifestQty.textContent = quantity;
    manifestCreated.textContent = new Date(createdAt).toLocaleTimeString();
    setStamp(status || 'PENDING');
  }

  function setStamp(status) {
    if (status === 'COMPLETED') {
      stamp.textContent = 'Completed';
      stamp.classList.remove('pending');
      stamp.classList.add('completed');
      trackFill.style.width = '100%';
      dot2.classList.add('done');
      statusText.textContent = 'Warehouse confirmed';
    } else {
      stamp.textContent = 'Pending';
      stamp.classList.add('pending');
      stamp.classList.remove('completed');
      trackFill.style.width = '15%';
      dot2.classList.remove('done');
      statusText.textContent = 'Processing…';
    }
  }

  function startPolling(orderId) {
    if (pollTimer) clearInterval(pollTimer);

    pollTimer = setInterval(async () => {
      try {
        const res = await fetch(`${API_BASE}/orders/${orderId}`);
        if (!res.ok) return;
        const data = await res.json();

        if (data.status === 'COMPLETED') {
          setStamp('COMPLETED');
          clearInterval(pollTimer);
        }
      } catch (err) {
        // Silent fail — next poll tick will just try again
      }
    }, 2000);
  }
</script>

</body>
</html>
```

---

# Part 5 — Deploy the webpage

1. Create an S3 bucket, e.g. `purepak-order-desk`
2. **Properties → Static website hosting → Enable**, index document: `index.html`
3. **Permissions → Bucket policy** — allow public reads:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "<YOUR-BUCKET-ARN>/*"
    }
  ]
}
```
4. Upload your edited `index.html` (with `API_BASE` filled in)
5. Open the static website endpoint

---

# Part 6 — Test it end to end

1. Submit an order with a valid product name and quantity — the button should show a spinner, then the tracker card on the right should populate with `PENDING` and a stamped badge tilted like a rubber stamp
2. Watch the tracker — within a few seconds (once `ProcessOrderFunction` picks the message off SQS) the status should flip to `COMPLETED`, the stamp turns green, and the progress line fills
3. Check your email if you subscribed to `OrderProcessedTopic` — you should receive the SNS notification at roughly the same moment the badge flips
4. Try submitting with an empty product name or a `0` quantity — you should see inline red field errors, and no network request should even fire
5. Copy an order ID from the manifest card, refresh the page, paste it into the **Look up an order** field under "Track Status" and click **Look Up** — it should pull the order straight from DynamoDB via `GetOrderStatusFunction` and display it, picking up polling automatically if it's still `PENDING`
6. Try looking up a made-up order ID (e.g. `not-a-real-id`) — you should see an inline "Order not found" message rather than a broken page
7. Try a quantity like `-5` directly via `curl` to confirm the backend rejects it too, not just the frontend:
```bash
curl -X POST <invoke-url>/orders \
  -H "Content-Type: application/json" \
  -d '{"productName":"Test","quantity":-5}'
```
   Expect a `400` with an `errors` array — this confirms validation isn't just cosmetic on the page.

---

## What you've built

A submission-to-confirmation loop with real user feedback: **API Gateway** validates the shape of the request, **SubmitOrderFunction** validates the business rules and writes a `PENDING` record before queueing, **SQS** buffers the work, **ProcessOrderFunction** completes it and notifies via **SNS**, and the **webpage** actually shows the customer what happened — live — instead of a single "success" alert with no way to check back.

**Suggested next extension:** add a `FAILED` status path — if `ProcessOrderFunction` can't fulfill an order (e.g. simulate an out-of-stock check), have it update the record to `FAILED` with a reason, and extend the tracker card to show a red stamp state for that case too.
