# Scaledaction sentiment-analysis DTK deployment procedure

This document is guide for deploying Scaledaction sentiment-analysis cluster via DTK. Following is the diagram of Scaledaction cluster:

<img src="scaledaction.png" width="1000">
 
Service module used is: scaledaction:sentiment_analysis
Assembly template is: cluster

Assembly template has following nodes:
- akka-seed - Node where ingest-frontend akka jar is running and pickup stream from Twitter. This node has connection to Kafka broker to be able to write output to tweets topic
- kafka-broker - Node where Kafka is installed, kafka broker is running and tweets topic is created
- spark-master - Spark master node where ingest-backend jar is running, picking up messages from Kafka broker, doing processing and writing output to tweets table in Cassandra
- spark-workers - Two additional nodes that act as Spark workers inside spark cluster. Number of these nodes is scalable if needed
- cassandra-seed - Cassandra seed node
- cassandra-peer - Two additional nodes that act as Cassandra peer nodes inside cassandra cluster. Number of these nodes is scalable if needed

## DTK Deployment demo:
Following part of the guide shows procedure of deploying Scaledaction sentiment-analysis cluster. First step is to stage scaledaction:sentiment_analysis/cluster and set needed attributes:

```
dtk:/service-module/scaledaction:sentiment_analysis>stage cluster -v master -t <vpc_target_name>
---
new_service_instance:
  name: sentiment_analysis-cluster
  id: 2148437088
  is_created: true
 
dtk:/service-module/scaledaction:sentiment_analysis>cd
dtk:/>cd service
dtk:/service>cd sentiment_analysis-cluster
dtk:/service/sentiment_analysis-cluster>set-required-attributes
 
Please fill in missing data.
Please enter akka-seed/setup::credentials_aggregator/aws_access_key [STRING]:
: <AWS_ACCESS_KEY>
Please enter akka-seed/setup::credentials_aggregator/aws_secret_key [STRING]:
: <AWS_SECRET_KEY>
--------------------------------- DATA ---------------------------------
akka-seed/setup::credentials_aggregator/aws_access_key : <AWS_ACCESS_KEY>
akka-seed/setup::credentials_aggregator/aws_secret_key : <AWS_SECRET_KEY>
------------------------------------------------------------------------
 
Is provided information ok? (yes|no) yes
dtk:/service/sentiment_analysis-cluster>
```

Now that we have all attributes set, we can start with deployment process. Deployment process is divide in separate actions. We have following actions:
- initial_setup - Initial setup of aws credentials and cli on all nodes where interaction with AWS S3 service will be needed. You can take a look at diagram above to see which nodes are connected to AWS S3
- create_kafka_cluster - Action which deploys kafka broker and creates topic: tweets
- create_cassandra_cluster - Action which deploys cassandra cluster and creates keyspace: twitter and table: tweets
- create_akka_cluster - Action which pulls ingest-frontend.jar from AWS S3
- create_spark_cluster - Action which deploys spark cluster and pulls ingest-backend.jar from AWS S3 on spark master node
- run_ingest_frontend - This action exports needed environment variables on akka-seed node and runs ingest-frontend jar as background service (with nohup)
- run_ingest_backend - This action exports needed environment variables on spark-master node and runs ingest-backend jar as background service (with nohup)

We will execute these actions in following order:

```
dtk:/service/sentiment_analysis-cluster>exec-sync initial_setup
========================= 2016-04-04 07:53:11 +0000 start 'initial_setup' =========================
 
 
============================================================
STAGE 1: create_nodes_stage
TIME START: 2016-04-04 07:48:18 +0000
OPERATION: CreateNodes
  kafka-broker
  cassandra-seed
  akka-seed
  spark-master
  spark-workers:2
  spark-workers:1
  cassandra-peers:2
  cassandra-peers:1
------------------------------------------------------------
 
============================================================
STAGE 2: bigtop_multiservice
TIME START: 2016-04-04 07:54:34 +0000
COMPONENT: assembly_wide/bigtop_multiservice
STATUS: succeeded
DURATION: 0.2s
------------------------------------------------------------
 
============================================================
STAGE 3: bigtop hiera
TIME START: 2016-04-04 07:54:35 +0000
COMPONENTS:
  kafka-broker/bigtop_multiservice::hiera
  akka-seed/bigtop_multiservice::hiera
  spark-master/bigtop_multiservice::hiera
  node-group:spark-workers/bigtop_multiservice::hiera
STATUS: succeeded
DURATION: 131.1s
------------------------------------------------------------
 
============================================================
STAGE 4: bigtop_base
TIME START: 2016-04-04 07:56:46 +0000
COMPONENTS:
  kafka-broker/bigtop_base
  akka-seed/bigtop_base
  spark-master/bigtop_base
  node-group:spark-workers/bigtop_base
STATUS: succeeded
DURATION: 21.3s
------------------------------------------------------------
 
============================================================
STAGE 5: aws setup
TIME START: 2016-04-04 07:57:07 +0000
COMPONENT: akka-seed/setup::credentials_aggregator
STATUS: succeeded
DURATION: 9.1s
------------------------------------------------------------
 
========================= end: 'initial_setup' (total duration: 245.0s) =========================
Status: OK
dtk:/service/sentiment_analysis-cluster>exec-sync create_kafka_cluster
========================= 2016-04-04 07:58:26 +0000 start 'create_kafka_cluster' =========================
 
 
============================================================
STAGE 1: set zookeeper conf
TIME START: 2016-04-04 07:58:26 +0000
COMPONENTS:
  assembly_wide/zookeeper
  kafka-broker/zookeeper::server
STATUS: succeeded
DURATION: 35.8s
------------------------------------------------------------
 
============================================================
STAGE 2: kafka broker conf
TIME START: 2016-04-04 07:59:02 +0000
COMPONENTS:
  kafka-broker/kafka::broker
  kafka-broker/kafka::topic[tweets]
STATUS: succeeded
DURATION: 33.7s
------------------------------------------------------------
 
========================= end: 'create_kafka_cluster' (total duration: 69.7s) =========================
Status: OK
dtk:/service/sentiment_analysis-cluster>exec-sync create_cassandra_cluster
========================= 2016-04-04 08:05:04 +0000 start 'create_cassandra_cluster' =========================
 
 
============================================================
STAGE 1: host aggregator
TIME START: 2016-04-04 08:05:04 +0000
COMPONENT: assembly_wide/cassandra::cluster
STATUS: succeeded
DURATION: 0.3s
------------------------------------------------------------
 
============================================================
STAGE 2: cassandra seed setup
TIME START: 2016-04-04 08:05:04 +0000
COMPONENTS:
  cassandra-seed/cassandra::java
  cassandra-seed/cassandra::seed
STATUS: succeeded
DURATION: 31.1s
------------------------------------------------------------
 
============================================================
STAGE 3: install cassandra and scaledaction cassandra setup
TIME START: 2016-04-04 08:05:35 +0000
COMPONENTS:
  cassandra-seed/cassandra
  cassandra-seed/scaledaction_common::cassandra
STATUS: succeeded
DURATION: 45.1s
------------------------------------------------------------
 
============================================================
STAGE 4: cassandra peers setup
TIME START: 2016-04-04 08:06:21 +0000
COMPONENT: node-group:cassandra-peers/cassandra::java
STATUS: succeeded
DURATION: 32.2s
------------------------------------------------------------
 
============================================================
STAGE 5: install cassandra
TIME START: 2016-04-04 08:06:53 +0000
COMPONENT: node-group:cassandra-peers/cassandra
STATUS: succeeded
DURATION: 42.8s
------------------------------------------------------------
 
============================================================
STAGE 6: scaledaction cassandra create keyspace and table
TIME START: 2016-04-04 08:07:36 +0000
COMPONENTS:
  cassandra-seed/scaledaction_common::cassandra
  cassandra-seed/scaledaction_common::cassandra
STATUS: succeeded
DURATION: 97.0s
------------------------------------------------------------
 
========================= end: 'create_cassandra_cluster' (total duration: 249.2s) =========================
Status: OK
dtk:/service/sentiment_analysis-cluster>exec-sync create_akka_cluster
========================= start 'create_akka_cluster' =========================
 
 
============================================================
STAGE 1: get scaledaction frontend app from S3
TIME START: 2016-04-04 08:19:33 +0000
COMPONENTS:
  akka-seed/setup
  akka-seed/s3::get_bucket_data[sentiment-ingest-frontend-assembly-1.0.jar]
  akka-seed/scaledaction_common::akka
  spark-master/setup
STATUS: succeeded
DURATION: 84.9s
------------------------------------------------------------
 
========================= end: 'create_akka_cluster' (total duration: 84.9s) =========================
Status: OK
dtk:/service/sentiment_analysis-cluster>exec-sync create_spark_cluster
========================= 2016-04-04 08:55:39 +0000 start 'create_spark_cluster' =========================
 
 
============================================================
STAGE 1: get scaledaction backend app from S3
TIME START: 2016-04-04 08:55:39 +0000
COMPONENTS:
  assembly_wide/spark::cluster
  assembly_wide/hadoop::cluster
  akka-seed/setup
  spark-master/setup
  spark-master/scaledaction_common::spark
  spark-master/s3::get_bucket_data[sentiment-ingest-backend-assembly-1.0.jar]
  spark-master/s3::get_bucket_data[positive.gz]
  spark-master/s3::get_bucket_data[negative.gz]
  spark-master/s3::get_bucket_data[spark-streaming-kafka_2.10-1.5.1.jar]
STATUS: succeeded
DURATION: 53.5s
------------------------------------------------------------
 
============================================================
STAGE 2: spark master and client
TIME START: 2016-04-04 08:56:33 +0000
COMPONENTS:
  spark-master/spark::master
  spark-master/spark::client
STATUS: succeeded
DURATION: 47.0s
------------------------------------------------------------
 
============================================================
STAGE 3: spark worker
TIME START: 2016-04-04 08:57:20 +0000
COMPONENTS:
  node-group:spark-workers/spark::common
  node-group:spark-workers/spark::worker
STATUS: succeeded
DURATION: 149.9s
------------------------------------------------------------
 
========================= end: 'create_spark_cluster' (total duration: 250.6s) =========================
dtk:/service/sentiment_analysis-cluster>exec-sync run_ingest_frontend
========================= 2016-04-04 19:38:10 +0000 start 'run_ingest_frontend' =========================
 
 
============================================================
STAGE 1: execute ingest frontend
TIME START: 2016-04-04 19:38:10 +0000
ACTION: akka-seed/scaledaction_common::akka.run_scaledaction_frontend
STATUS: succeeded
DURATION: 5.4s
RESULTS:
 
NODE: akka-seed
RUN: /bin/bash /etc/puppet/modules/scaledaction_common/files/run_akka_frontend.sh <TWITTER_CONSUMER_KEY> <TWITTER_CONSUMER_SECRET> <TWITTER_TOKEN_KEY> <TWITTER_TOKEN_SECRET> <KAFKA_BROKER_HOST> <KAFKA_BROKER_PORT> /tmp sentiment-ingest-frontend-assembly-1.0.jar Apple (syscall)
RETURN CODE: 0
 
------------------------------------------------------------
 
========================= end: 'run_ingest_frontend' (total duration: 5.4s) =========================
Status: OK
dtk:/service/sentiment_analysis-cluster>exec-sync run_ingest_backend
========================= 2016-04-04 19:38:24 +0000 start 'run_ingest_backend' =========================
 
 
============================================================
STAGE 1: execute ingest backend
TIME START: 2016-04-04 19:38:24 +0000
ACTION: spark-master/scaledaction_common::spark.run_scaledaction_backend
STATUS: succeeded
DURATION: 5.4s
RESULTS:
 
NODE: spark-master
RUN: /bin/bash /etc/puppet/modules/scaledaction_common/files/run_spark_backend.sh <KAFKA_BROKER_HOST> <KAFKA_BROKER_PORT> <CASSANDRA_SEED_HOST> /usr/lib/spark/lib/spark-assembly-1.5.1-hadoop2.6.0.jar /tmp/spark-streaming-kafka_2.10-1.5.1.jar /tmp sentiment-ingest-backend-assembly-1.0.jar /tweet-corpus (syscall)
RETURN CODE: 0
 
------------------------------------------------------------
 
========================= end: 'run_ingest_backend' (total duration: 5.4s) =========================
Status: OK
```

Finally, when we have all actions are executed, it is time to check whether our cluster works properly and if it is streaming tweets from Twitter all the way to the Cassandra tweets table. The easiest way to check this is to login to cassandra-seed node and use cqlsh prompt to check content of tweets tables:

```
dtk:/service/sentiment_analysis-cluster>cd cassandra-seed
dtk:/service/sentiment_analysis-cluster/cassandra-seed>ssh
You are entering SSH terminal (ec2-user@ec2-52-23-233-196.compute-1.amazonaws.com) ...
Warning: Permanently added 'ec2-52-23-233-196.compute-1.amazonaws.com,52.23.233.196' (ECDSA) to the list of known hosts.
Last login: Tue Feb  2 14:51:12 2016 from 80.65.166.98
 
       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|
 
https://aws.amazon.com/amazon-linux-ami/2015.09-release-notes/
26 package(s) needed for security, out of 70 available
Run "sudo yum update" to apply all updates.
Amazon Linux version 2016.03 is available.
[ec2-user@ip-10-0-0-81 ~]$ cqlsh ec2-52-23-233-196.compute-1.amazonaws.com
Connected to Cassandra_Test_Cluster at ec2-52-23-233-196.compute-1.amazonaws.com:9042.
[cqlsh 5.0.1 | Cassandra 2.2.5 | CQL spec 3.3.1 | Native protocol v4]
Use HELP for help.
cqlsh> use twitter;
cqlsh:twitter> select * from tweets;
 
 tweet                                                                                                                                                                           | batchtime     | query | score | tweet_text
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+-------+-------+---------------------------------------------------------------------------------------------------------------------------------------------------
 {"query": "Apple", "text": "RT @VintageClassics: Congrats #sbs win @larkandlily @RagtagDev @AppleCratesUK @CrowdScreen @creativenature @ButterflyBlind enjoy ur bonkers\u2026"} | 1459798824000 | Apple |     1 | RT @VintageClassics: Congrats #sbs win @larkandlily @RagtagDev @AppleCratesUK @CrowdScreen @creativenature @ButterflyBlind enjoy ur bonkers\u2026
          {"query": "Apple", "text": "RT @McGMaryland: FBI offering iPhone cracking services to federal agencies, local law enforcement https://t.co/C8fsn1WbL9 #Apple #mcggov"} | 1459798890000 | Apple |     1 |          RT @McGMaryland: FBI offering iPhone cracking services to federal agencies, local law enforcement https://t.co/C8fsn1WbL9 #Apple #mcggov
  {"query": "Apple", "text": "Flip Wallet Leather Case Cover For Apple Iphone 5 5S With Free Screen Protector - Buy It N\u2026 https://t.co/u5RW6IyU7C https://t.co/NbDTpFf2hr"} | 1459798848000 | Apple |     1 |  Flip Wallet Leather Case Cover For Apple Iphone 5 5S With Free Screen Protector - Buy It N\u2026 https://t.co/u5RW6IyU7C https://t.co/NbDTpFf2hr
                                                                         {"query": "Apple", "text": "@bearski55 sounds like a lot of fun x \U0001f60a@bern1508 @Sappho31Apples"} | 1459798876000 | Apple |     1 |                                                                         @bearski55 sounds like a lot of fun x \U0001f60a@bern1508 @Sappho31Apples
 {"query": "Apple", "text": "RT @El_Diagonal: Assange, Snowden, Kiriakou, Apple... protagonistas de un conflicto desnivelado contra el imperio de la vigilancia https://\u2026"} | 1459798918000 | Apple |     1 | RT @El_Diagonal: Assange, Snowden, Kiriakou, Apple... protagonistas de un conflicto desnivelado contra el imperio de la vigilancia https://\u2026
                  {"query": "Apple", "text": "At Apple, \u2018Pro\u2019 is the new normal https://t.co/yf5Vgv6yLN #Opinion #129inchiPadPro #97inchiPadPro #Apple #iPhone #News"} | 1459798812000 | Apple |     1 |                  At Apple, \u2018Pro\u2019 is the new normal https://t.co/yf5Vgv6yLN #Opinion #129inchiPadPro #97inchiPadPro #Apple #iPhone #News
 
(6 rows)
```

## Conclusion
As you can see, end-to-end process for Scaledaction sentiment analysis was deployed successfully as we see tweets with query "Apple" get processed through sentiment analysis pipeline and get eventually written in Cassandra tweets table
