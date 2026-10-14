# Amazon DynamoDB: The Serverless Cloud-Native Key-Value & Document Engine

## Overview
Amazon DynamoDB is a fully managed, serverless, multi-region NoSQL database service developed by Amazon Web Services (AWS). Engineered for mission-critical web-scale workloads, DynamoDB provides predictable, single-digit millisecond latency at any scale with automatic horizontal partitioning, built-in security, in-memory caching (DAX), and multi-active replication across AWS regions (Global Tables).

---

## Architectural Deep Dive: Partitions, Multi-Paxos & Request Routers

```
+-------------------------------------------------------------------------+
|                       DynamoDB System Architecture                      |
+-------------------------------------------------------------------------+
 Client Application (AWS SDK / boto3)
                │
                ▼
 Request Routers (Stateless fleet: authenticates, signs, hashes partition key)
                │
                ▼
 Storage Node Partition Replica Group (3 Nodes per Partition across 3 AZs)
   ┌───────────────────┬───────────────────┬───────────────────┐
   │ Leader (Paxos)    │ Follower Replica  │ Follower Replica  │
   │ - Handles Writes  │ - Read Replicas   │ - Read Replicas   │
   │ - WAL + B-Tree SSD│ - Async/Sync rep  │ - Async/Sync rep  │
   └───────────────────┴───────────────────┴───────────────────┘
                │
                ├──► DynamoDB Streams (24-hour changelog shard stream)
                └──► Global Secondary Indexes (Asynchronous replication)
```

### 1. Partition Mechanics & Capacity Hard Limits
- **Partition Key Hashing:** The primary partition key is hashed via MD5 to assign the item to a specific storage partition.
- **Single Partition Thresholds:**
  - Maximum storage per physical partition: **10 GB**.
  - Maximum throughput per physical partition: **1,000 WCU** (Write Capacity Units) or **3,000 RCU** (Read Capacity Units).
- If traffic to a single partition key exceeds these limits, DynamoDB throttles the request (`ProvisionedThroughputExceededException`). **Partition Key Design** with high cardinality (e.g., `user_id` or composite hash) is crucial to avoid "Hot Partition" bottlenecks.

### 2. Multi-Paxos Replication & Consistency Modes
- Every partition is replicated across 3 Availability Zones (AZs) in a Multi-Paxos replica group.
- **Eventually Consistent Reads (Default):** Returns data from any replica node immediately. May return stale data if replication is completing ($0.5\text{ RCU}$ per 4 KB).
- **Strongly Consistent Reads:** Routed directly to the Paxos leader replica node, guaranteeing the most up-to-date data ($1.0\text{ RCU}$ per 4 KB).
- **Transactional Reads:** Two-phase commit protocol across partitions ($2.0\text{ RCU}$ per 4 KB).

---

## Real World Stories & Incidents

### 1. Amazon Prime Day Record-Breaking Scale
During Prime Day events, Amazon's entire consumer e-commerce platform runs on DynamoDB:
- In 2023, Amazon Prime Day generated **trillions of API calls**, peaking at tens of millions of requests per second.
- DynamoDB sustained consistent single-digit millisecond response times without manual intervention or database instance provisioning, proving the power of serverless autoscaling.

### 2. The 2015 AWS US-East-1 DynamoDB Outage
In September 2015, a massive cascading failure crippled AWS US-East-1:
- A network blip triggered transient partition reassignment requests.
- The DynamoDB **Metadata Service** (which tracks which storage nodes hold which token partitions) was flooded with membership updates.
- Storage nodes could not renew their lease with the metadata service, causing them to falsely assume they were dead and reject client traffic.
- Over 20 major AWS services (including S3, EC2, and CloudWatch) that depend on DynamoDB internally failed in sympathy, spurring AWS to completely re-architect the metadata lease protocol.

### 3. Lyft's Ride Dispatch Architecture
Lyft shifted core passenger and driver ride state tracking to DynamoDB:
- Ride events (driver accepted, en route, picked up, completed) require sub-10ms state transitions.
- Lyft uses **DynamoDB Streams** connected to AWS Lambda to trigger downstream microservices (ETL to Snowflake, real-time driver receipts, push notifications) completely decoupled from the primary database write.

---

## Capacity Units Calculation Cheat Sheet

| Operation Type | Size Unit | Capacity Units Needed |
|---|---|---|
| **Eventually Consistent Read** | 4 KB | $0.5\text{ RCU}$ per 4 KB |
| **Strongly Consistent Read** | 4 KB | $1.0\text{ RCU}$ per 4 KB |
| **Transactional Read** | 4 KB | $2.0\text{ RCU}$ per 4 KB |
| **Standard Write** | 1 KB | $1.0\text{ WCU}$ per 1 KB |
| **Transactional Write** | 1 KB | $2.0\text{ WCU}$ per 1 KB |

*Example:* Writing an 8 KB item requires $8\text{ WCU}$. Reading an 8 KB item strongly consistent requires $2\text{ RCU}$.

---

## Comprehensive DynamoDB Operations Reference (PartiQL & Python SDK)

### 1. Table Creation with GSI, LSI, and Billing Modes

#### Python (boto3) Table Creation:
```python
import boto3

dynamodb = boto3.client('dynamodb', region_name='us-east-1')

response = dynamodb.create_table(
    TableName='EcommerceOrders',
    # Primary Key: Partition Key (PK) + Sort Key (SK)
    KeySchema=[
        {'AttributeName': 'CustomerId', 'KeyType': 'HASH'},  # Partition Key
        {'AttributeName': 'OrderId', 'KeyType': 'RANGE'}     # Sort Key
    ],
    AttributeDefinitions=[
        {'AttributeName': 'CustomerId', 'AttributeType': 'S'},
        {'AttributeName': 'OrderId', 'AttributeType': 'S'},
        {'AttributeName': 'OrderStatus', 'AttributeType': 'S'},
        {'AttributeName': 'OrderDate', 'AttributeType': 'S'}
    ],
    # Global Secondary Index (GSI): Query by OrderStatus and OrderDate
    GlobalSecondaryIndexes=[
        {
            'IndexName': 'GSI_Status_Date',
            'KeySchema': [
                {'AttributeName': 'OrderStatus', 'KeyType': 'HASH'},
                {'AttributeName': 'OrderDate', 'KeyType': 'RANGE'}
            ],
            'Projection': {
                'ProjectionType': 'ALL'  # ALL, KEYS_ONLY, or INCLUDE
            }
        }
    ],
    BillingMode='PAY_PER_REQUEST',  # On-Demand Serverless Capacity
    StreamSpecification={
        'StreamEnabled': True,
        'StreamViewType': 'NEW_AND_OLD_IMAGES'  # Captures both before and after states
    }
)
```

---

### 2. CRUD Operations with Condition Expressions

```python
# 1. PutItem with Idempotency & ConditionExpression
# Ensures order does not already exist
try:
    dynamodb.put_item(
        TableName='EcommerceOrders',
        Item={
            'CustomerId': {'S': 'CUST_4920'},
            'OrderId': {'S': 'ORD_99102'},
            'OrderStatus': {'S': 'PLACED'},
            'TotalAmount': {'N': '249.99'},
            'OrderDate': {'S': '2024-06-01T10:00:00Z'},
            'Items': {'L': [
                {'M': {'Sku': {'S': 'HEADPHONES_PRO'}, 'Qty': {'N': '1'}, 'Price': {'N': '249.99'}}}
            ]}
        },
        ConditionExpression='attribute_not_exists(OrderId)'
    )
except dynamodb.exceptions.ConditionalCheckFailedException:
    print("Order already exists! Preventing duplicate insert.")

# 2. GetItem (Strongly vs Eventually Consistent)
response = dynamodb.get_item(
    TableName='EcommerceOrders',
    Key={
        'CustomerId': {'S': 'CUST_4920'},
        'OrderId': {'S': 'ORD_99102'}
    },
    ConsistentRead=True,  # Strongly consistent read
    ProjectionExpression='OrderId, OrderStatus, TotalAmount'
)

# 3. UpdateItem with Complex Expressions (SET, ADD, REMOVE)
# Atomically changes status to 'PROCESSING' only if currently 'PLACED'
dynamodb.update_item(
    TableName='EcommerceOrders',
    Key={
        'CustomerId': {'S': 'CUST_4920'},
        'OrderId': {'S': 'ORD_99102'}
    },
    UpdateExpression='SET OrderStatus = :new_status, Version = Version + :inc',
    ConditionExpression='OrderStatus = :expected_status',
    ExpressionAttributeValues={
        ':new_status': {'S': 'PROCESSING'},
        ':expected_status': {'S': 'PLACED'},
        ':inc': {'N': '1'}
    },
    ReturnValues='ALL_NEW'  # Returns updated document
)

# 4. DeleteItem with Condition
dynamodb.delete_item(
    TableName='EcommerceOrders',
    Key={
        'CustomerId': {'S': 'CUST_4920'},
        'OrderId': {'S': 'ORD_99102'}
    },
    ConditionExpression='OrderStatus = :cancel_status',
    ExpressionAttributeValues={
        ':cancel_status': {'S': 'CANCELLED'}
    }
)
```

---

### 3. Query vs Scan (The Core Access Patterns)

#### Efficient Query Operation ($O(\log N)$):
```python
# Query: Fetch all orders for customer in a date range using PK + SK condition
response = dynamodb.query(
    TableName='EcommerceOrders',
    KeyConditionExpression='CustomerId = :cid AND OrderId BEGINS_WITH(:prefix)',
    ExpressionAttributeValues={
        ':cid': {'S': 'CUST_4920'},
        ':prefix': {'S': 'ORD_2024'}
    },
    ScanIndexForward=False,  # False = Descending sort by Sort Key; True = Ascending
    Limit=10
)

# Query on Global Secondary Index (GSI)
response = dynamodb.query(
    TableName='EcommerceOrders',
    IndexName='GSI_Status_Date',
    KeyConditionExpression='OrderStatus = :status AND OrderDate >= :start_date',
    ExpressionAttributeValues={
        ':status': {'S': 'PROCESSING'},
        ':start_date': {'S': '2024-06-01'}
    }
)
```

#### Parallel Scan Operation (Scatter-Gather across partitions):
```python
# Scanning an entire table is expensive; use Parallel Scan with workers
segment_id = 0
total_segments = 4

response = dynamodb.scan(
    TableName='EcommerceOrders',
    TotalSegments=total_segments,
    Segment=segment_id,
    FilterExpression='TotalAmount > :min_val',
    ExpressionAttributeValues={
        ':min_val': {'N': '1000.00'}
    }
)
```

---

### 4. ACID Transactions (TransactWriteItems)

Executes cross-item, cross-table atomic transactions (All-or-Nothing):

```python
# E-commerce Checkout: Deduct balance AND decrement inventory atomically
try:
    dynamodb.transact_write_items(
        TransactItems=[
            {
                # 1. Deduct customer balance
                'Update': {
                    'TableName': 'CustomerWallets',
                    'Key': {'CustomerId': {'S': 'CUST_4920'}},
                    'UpdateExpression': 'SET Balance = Balance - :cost',
                    'ConditionExpression': 'Balance >= :cost',
                    'ExpressionAttributeValues': {':cost': {'N': '249.99'}}
                }
            },
            {
                # 2. Decrement product inventory
                'Update': {
                    'TableName': 'Inventory',
                    'Key': {'Sku': {'S': 'HEADPHONES_PRO'}},
                    'UpdateExpression': 'SET Stock = Stock - :qty',
                    'ConditionExpression': 'Stock >= :qty',
                    'ExpressionAttributeValues': {':qty': {'N': '1'}}
                }
            },
            {
                # 3. Create the confirmed order
                'Put': {
                    'TableName': 'EcommerceOrders',
                    'Item': {
                        'CustomerId': {'S': 'CUST_4920'},
                        'OrderId': {'S': 'ORD_99102'},
                        'OrderStatus': {'S': 'CONFIRMED'}
                    }
                }
            }
        ]
    )
    print("Transaction succeeded!")
except dynamodb.exceptions.TransactionCanceledException as e:
    print(f"Transaction aborted due to failed condition check: {e}")
```

---

### 5. PartiQL: SQL-Compatible Queries for DynamoDB

DynamoDB natively supports PartiQL, allowing SQL queries directly against NoSQL tables:

```sql
-- SELECT using PartiQL
SELECT OrderId, TotalAmount 
FROM "EcommerceOrders" 
WHERE CustomerId = 'CUST_4920' AND OrderId = 'ORD_99102';

-- INSERT using PartiQL
INSERT INTO "EcommerceOrders" VALUE {
    'CustomerId': 'CUST_8811',
    'OrderId': 'ORD_1120',
    'OrderStatus': 'PLACED',
    'TotalAmount': 89.50
};

-- UPDATE using PartiQL with Condition
UPDATE "EcommerceOrders" 
SET OrderStatus = 'SHIPPED' 
WHERE CustomerId = 'CUST_4920' AND OrderId = 'ORD_99102'
  AND OrderStatus = 'PROCESSING';

-- DELETE using PartiQL
DELETE FROM "EcommerceOrders" 
WHERE CustomerId = 'CUST_4920' AND OrderId = 'ORD_99102';
```
