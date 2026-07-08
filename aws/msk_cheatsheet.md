# Amazon MSK (Managed Kafka) Cheatsheet

---

## Core Concepts

- MSK = **fully managed Apache Kafka** — real-time streaming with the full Kafka ecosystem
- Built on **Pub/Sub**: producers publish to **topics**, consumers subscribe — fully decoupled
- **MSK Serverless** auto-scales throughput/storage; no broker sizing

---

## Pub/Sub Model

| Role | Does |
| --- | --- |
| **Producer** | Writes messages to a topic (doesn't know consumers) |
| **Topic** | Named channel; buffers + categorizes messages |
| **Consumer** | Reads from a topic (doesn't know producers) |
| **Consumer group** | Set of consumers sharing the read load |

- One event published once → consumed independently by many services

---

## MSK vs Kinesis

| | MSK | Kinesis |
| --- | --- | --- |
| Tech | Open-source Kafka | Proprietary AWS |
| Scaling | More control (or Serverless) | Auto, serverless |
| Ease | Needs Kafka expertise | Simpler setup |
| Delivery | Configurable (up to exactly-once) | At-least-once |
| Deployment | Cloud / on-prem / hybrid | AWS only |
| Best for | Existing Kafka apps, sustained high volume | Simple AWS-native streaming, variable load |

---

## Serverless Setup (high level)

1. VPC with **2 subnets** in different AZs; security group open on **port 9098** (IAM auth port)
2. MSK → Create serverless cluster (auth: IAM)
3. Copy the **bootstrap server** string (used by producers/consumers)
4. EC2 client in same VPC with Kafka CLI + `aws-msk-iam-auth` JAR

### client.properties (IAM auth)
```
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

### Create a topic
```bash
bin/kafka-topics.sh --create \
  --bootstrap-server <bootstrap-server> \
  --command-config client.properties \
  --topic wikimedia-events --partitions 1
```

- Serverless MSK manages replication automatically
- EC2 role needs `kafka-cluster:Connect`, `CreateTopic`, `WriteData`, `ReadData`, etc.

---

## Python Producer (IAM/OAuth)

```python
from kafka import KafkaProducer
from aws_msk_iam_sasl_signer import MSKAuthTokenProvider
import json

class TP:
    def token(self): return MSKAuthTokenProvider.generate_auth_token('us-east-1')[0]

producer = KafkaProducer(
    bootstrap_servers=['<bootstrap>:9098'],
    security_protocol='SASL_SSL', sasl_mechanism='OAUTHBEARER',
    sasl_oauth_token_provider=TP(),
    value_serializer=lambda v: json.dumps(v).encode('utf-8'))
producer.send('wikimedia-events', {"wiki": "enwiki", "title": "Python"})
producer.flush()
```

---

## Consuming

- **EC2 KafkaConsumer** — custom code (same SASL/OAUTHBEARER config)
- **MSK Connect (S3 Sink)** — managed Kafka Connect → writes topic data to S3, no code
- **Lambda MSK trigger** — invoke a function per batch of records
- **Glue Streaming** — `spark.readStream.format("kafka")` → S3/DynamoDB/Redshift

---

## Monitoring (CloudWatch → AmazonMSKServerless)

- `BytesInPerSec`, `BytesOutPerSec`, `ActiveConnections`, `KafkaConsumerLag`
- MSK Connect → Connectors → status `Running`, tasks healthy