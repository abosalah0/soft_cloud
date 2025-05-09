# Order Notification System - Serverless (AWS)

This project implements an **event-driven order processing system** using AWS serverless services. It processes incoming order messages via SQS, triggers a Lambda function, and stores valid data in DynamoDB. Faulty messages are routed to a DLQ for further inspection.

---

## 🧩 Architecture

```
+------------+        +-------------+        +-------------------+
|            |        |             |        |                   |
|  Producer  +------->+   SQS Queue +------->+  Lambda Function  +----+
| (Simulator)|        |             |        |                   |    |
+------------+        +-------------+        +-------------------+    |
                                                                      v
                                                                 +---------+
                                                                 | DynamoDB|
                                                                 +---------+
                                                                      |
                                                                      v
                                                           +-------------------+
                                                           |  Dead Letter Queue |
                                                           +-------------------+
```

---

## 🚀 Setup Instructions

### 1. **Create DynamoDB Table**

* Table name: `Orders`
* Partition key: `orderId` (String)
* Capacity mode: On-demand

### 2. **Create the Primary SQS Queue**

* Name: `order-queue`
* Set visibility timeout: `30 seconds` (or more depending on processing time)
* Configure a **Dead Letter Queue (DLQ)**:

  * Create a queue: `order-dlq`
  * Set redrive policy on `order-queue`: maxReceiveCount = 3, DLQ = `order-dlq`

### 3. **Create the Lambda Function**

* Runtime: Python 3.12
* Permissions:

  * SQS message processing
  * DynamoDB `PutItem` access
* Environment variable: `ORDERS_TABLE_NAME=Orders`
* Attach the Lambda trigger to `order-queue`
* Function code example:

```python
import json
import boto3
import os

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['ORDERS_TABLE_NAME'])

def lambda_handler(event, context):
    for record in event['Records']:
        try:
            body = json.loads(record['body'])
            item = {
                'orderId': str(body['orderId']),
                'product': body['product'],
                'quantity': int(body['quantity'])
            }
            table.put_item(Item=item)
        except Exception as e:
            print(f"Error processing record: {record}. Error: {e}")
            raise e
```

### 4. **Test**

* Send a message to `order-queue` (JSON):

```json
{
  "orderId": "12345",
  "product": "Laptop",
  "quantity": 1
}
```

* Observe successful records in DynamoDB.
* Corrupted messages will be sent to DLQ after 3 failed retries.

---

## 🛠️ Tools Used

* AWS Lambda
* Amazon SQS & DLQ
* Amazon DynamoDB
* IAM Roles

---

## ✅ Features

* Serverless architecture
* Event-driven processing
* Resilience via retry + DLQ
* Simple error handling and debugging via DLQ analysis

---

## 📂 Bonus (Optional)

* Infrastructure-as-Code (CloudFormation / Terraform) can be added to automate deployment.
