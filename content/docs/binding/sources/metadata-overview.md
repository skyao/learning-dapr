---
title: "资源绑定的Metadata总结"
linkTitle: "Metadata总结"
weight: 739
date: 2021-01-31
description: >
  Dapr资源绑定的Metadata总结
---

总结一下各种binding实现中 metadata 的设计和使用: 

| 实现                   | 配置级别的metadata                                           | 请求级别的metadata                                           |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| alicloud oss           |                                                              | key                                                          |
| HTTP                   | url / method                                                 | 无                                                           |
| cron                   | schedule                                                     | 无                                                           |
| MQTT                   | url / topic                                                  | 无                                                           |
| RabbitMQ               | host / queueName / durable <br />deleteWhenUnused / prefetchCount | ttlInSeconds                                                 |
| Redis                  | host / password / enableTLS / <br />maxRetries / maxRetryBackoff | key                                                          |
| Influx                 | url / token / org / bucket                                   | 无                                                           |
| Kafka                  | brokers / topics / publishTopic<br />consumerGroup / authRequried<br />saslUsername / saslPassword | key                                                          |
| Kubernetes             | namespace / resyncPeriodInSec /                              | 无                                                           |
| twilio-sendgrid        | apiKey / emailFrom / emailTo<br />subject / emailCc / emailBcc | emailFrom / emailTo / subject<br />emailCc / emailBcc        |
| twilio-sms             | toNumber / fromNumber / accountSid<br />authToken / timeout  | toNumber                                                     |
| twitter                | consumerKey / consumerSecret / accessToken<br />accessSecret / query | query / lang / result / since_id                             |
| gcp-bucket             | bucket / type / project_id / private_key_id<br />private_key / client_email / client_id<br />auth_uri / token_uri <br />auth_provider_x509_cert_url / client_x509_cert_url | name                                                         |
| gcp-pubsub             | topic / subscription / type / <br /> project_id / private_key_id / private_key<br />client_email / client_id / auth_uri / token_uri<br />auth_provider_x509_cert_url / client_x509_cert_url | topic                                                        |
| Azure-blobstorage      | storageAccount / storageAccessKey / container                | blobName / ContentType / ContentMD5<br />ContentEncoding / ContentLanguage<br />ContentDisposition / CacheControl |
| Azure-cosmosDB         | url / masterKey / database / <br />collection / partitionKey | 无                                                           |
| Azure-EventGrid        | tenantId / subscriptionId / clientId<br />clientSecret / subscriberEndpoint<br />handshakePort / scope<br />eventSubscriptionName / accessKey<br />topicEndpoint | 无                                                           |
| Azure-EventHubs        | connection /  consumerGroup / storageAccountName / <br />storageAccountKey / storageContainerName <br /> partitionID / partitionKey | partitionKey                                                 |
| Azure-ServiceBusQueues | connectionString / queueName / ttl                           | id / correlationID / ttlInSeconds                            |
| Azure-SignalR          | connectionString / hub                                       | hub / group / user                                           |
| Azure-storagequeue     |                                                              | ttlInSeconds                                                 |
| Aws-dynamodb           | region / endpoint / accessKey<br />secretKey / table         | 无                                                           |
| Aws-kinesis            | streamName / consumerName / region <br />endpoint / accessKey <br />secretKey / mode | partitionKey                                                 |
| Aws-s3                 | region / endpoint / accessKey<br />secretKey / bucket        | key                                                          |
| Aws-sns                | topicArn / region / endpoint<br />accessKey / secretKey      | 无                                                           |
| Aws-sqs                | queueName / region / endpoint<br />accessKey / secretKey     | 无                                                           |
|                        |                                                              |                                                              |
|                        |                                                              |                                                              |















