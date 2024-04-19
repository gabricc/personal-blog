---
layout: post
image: assets/images/clickhouse/architecture.png
author: gabriel
featured: false
published: true
categories: [ clickhouse, devops, data ]
title: ClickHouse implementation - from a DevOps perspective
---

At the beginning of 2023, after several discussions and some intensive research, we kicked off the ClickHouse Implementation Project. Briefly, the scope of this project covered:
1. retire the Snowflake warehouse;
2. retire Stitch (used for data replication from various sources);
3. move our ETL from depending on proprietary solutions to a 100% open-source stack.

The main motivators for this restructuring were: increasing our analytical capabilities; enabling us to provide better features to our users; achieving cost reduction; and solving replication issues caused by vendor lock-in.

This diagram presents the final architecture we decided to implement:
![](assets/images/clickhouse/architecture.png)

<br>

# Data pipeline
We have data coming from many different sources: some of them are third party providers (e.g. Salesforce, Sendgrid, Google Ads.), others are internal services (e.g. PostgreSQL, GitHub). Because of that, our data pipeline had to be built to be flexible enough to receive all this data. Currently, data is being ingested into our data warehouse (ClickHouse) in three different ways:

1. Airbyte (raw JSON)

![](assets/images/clickhouse/airbyte-json.png)
![](assets/images/system-design/concurrency-example.png)

Data comes from the source as raw JSON to an ingestion database (e.g. google_ads_ingestion);
We run a dbt model on top of the raw data sending the new data to a structured database (e.g. dbt_google_ads).
2. Airbyte (normalized data)

![](assets/images/clickhouse/airbyte-normalized.png)

2.1. Data comes from the source as raw JSON;

2.2. Airbyte runs an internal DBT model to transform that data and make it available in a structured way.

3. ClickHouse MaterializedPostgreSQL Engine

![](assets/images/clickhouse/ch-materializedPostgresql.png)

Data is replicated from the source using a ClickHouse feature called MaterializedPostgreSQL Engine.

<br>

# Clickhouse Infrastructure
ClickHouse is a fast and reliable columnar database. It’s open-source, scales linearly, and can be deployed either as a single instance or as a cluster. In this documentation, we will demonstrate how it was deployed at Slang, including server setup, monitoring and alerting, and automated backups.

### Is a single node enough?
One of the most important steps is to decide whether to run ClickHouse as a single instance or as a cluster. Both come with their own set of tradeoffs.

Depending on your requirements, you may not need a cluster setup. Given enough resources, ClickHouse can process billions of records on a single server. This is simpler and helps keep costs down — that’s why it’s an attractive option for many use cases.

At Slang we run a single-node ClickHouse, as we don’t need a complicated cluster setup with our current scenario. Our instance runs on plain AWS EC2.

### Choice of hardware
Just like all of Slang’s resources running on AWS, the infrastructure was provisioned using Terraform.

The ClickHouse team has various [recommendations](https://tech.slangapp.com/clickhouse-implementation-from-a-devops-perspective-87949dae166b#:~:text=team%20has%20various-,recommendations,-when%20it%20comes) when it comes to choice of hardware.

After some research and testing we decided to go with:

- Instance type: `c7g.8xlarge`.
- OS: Ubuntu Server 22.04 (ARM64)
- Storage: 1TB SSD (gp3)
The reason we picked the c7g family is because c7g instances are powered by the latest generation AWS Graviton3 processors and provide the best price performance in Amazon EC2 for compute-intensive workloads. As you can see in the graph below, we are making good use of this processing power:

![](assets/images/clickhouse/cpu-graph.png)

We chose EBS volumes with gp3 as they’re [faster and cheaper](https://aws.amazon.com/blogs/storage/migrate-your-amazon-ebs-volumes-from-gp2-to-gp3-and-save-up-to-20-on-costs/) than gp2. Also, we can optionally split the disk into multiple volumes as this can increase throughput, or have better costs if you tier your storage (e.g. automatically move data older than 3 months onto slower disks).

### Server setup
Some actions we took to make the server more robust, following ClickHouse best practices:

**Mount the EBS volume as separate disk**
```
# tail -n 1 /etc/fstab
UUID="e253d9d4-96e4-4c3e-8db8-863b4cff124e" /var/lib/clickhouse ext4 rw,noatime 0 2
```
The `noatime` option disables writing file access times every time a file is read. This can improve performance for programs that access the file system frequently (such as databases), and has virtually [no impact](https://askubuntu.com/a/1116092) on your applications.

**Increase the nofile limit**
```
echo "
* hard nofile 262144
* soft nofile 262144" >> /etc/security/limits.conf
```

**Disable swap files**

ClickHouse doesn’t like swap files, as they can slow down performance when they kick in. We could disable them entirely, but that can lead to service disruptions in cases where the system needs more memory than what’s available.

Instead, we’ll hint to the OS to not rely on swap files unless absolutely necessary. Open the `/etc/sysctl.conf` file and adjust the VM swappiness setting:
```
echo "vm.swappiness=1" >> /etc/sysctl.conf
```

### Clickhouse installation
We installed ClickHouse using the Ubuntu package manager (apt). The installation instructions can be found [here](https://clickhouse.com/docs/en/install/).

### Clickhouse security
We have taken some precautions to enhance the security of the environment.

**User security**

- default user was disabled;
- each user/service has its own credentials with limited permissions following the principle of least privilege;
- users were created specifying the source where they can login, and login attempts with a different source IP address are denied:
```
CREATE USER service_airbyte HOST IP '10.0.0.0/20' IDENTIFIED WITH sha256_hash BY '***';
```

**Server security**

- unused ports were disabled;
- server security group only allows specific ports like: 9000, 8123 and 9363.

**User limits**

[Here is a list](https://clickhouse.com/docs/en/operations/server-configuration-parameters/settings) of all the available seetings. What we’ve done so far:
```
max_concurrent_queries=100
max_server_memory_usage_to_ram_ratio=0.9
max_memory_usage_for_user=13GB
```

### Logs

ClickHouse logs can be found under `/var/log/clickhouse-server`. Inside this directory, we have two main files:

**clickhouse-server.err.log** → errors thrown by the server are stored here.

**clickhouse-server.log** → this file contains logs of all operations being done inside Clickhouse.

The error log file is also being routed to our Elastic deployment.

### Server monitoring & alerting 📊
Server and database metrics are exported and stored in Prometheus through node_exporter and ClickHouse native exporter. These metrics are displayed in Grafana dashboards. I won’t dive into details here, but the dashboards can be easily found on the internet.

Alerting rules were created using Prometheus AlertManager; these are the rules we have configured so far:

![](assets/images/clickhouse/alerting.png)

### Automated backups ☢️
Backups are handled by ClickHouse’s built-in [backup feature](https://clickhouse.com/docs/en/operations/backup/#configuring-backuprestore-to-use-an-s3-endpoint) and stored inside an S3 bucket.
We have a cron inside ClickHouse server that triggers incremental backups daily:
```
# Clickhouse backup
0 0 * * * /root/scripts/clickhouse-backup.sh
```
The script was developed following the [ClickHouse Backup Documentation](https://clickhouse.com/docs/en/operations/backup).

## Airbyte
Just like ClickHouse, Airbyte has many installation methods, and every company should choose what best fits its use case. For Slang, after some research and testing, we decided to deploy Airbyte on our AWS managed Kubernetes cluster (EKS). One of the main reasons for this is the elasticity that Kubernetes provides. Airbyte is creating and killing pods every minute, and having the ability to scale the infrastructure together with it is a huge pro.

**Choice of hardware**: You may think that we don’t have to worry about hardware as we are deploying it on a Managed K8S Cluster, but that would be wrong. Airbyte deployment can get very tricky if we don’t choose the right instance type and size for the NodeGroup machines. All of the core services (e.g. server, db, worker…) must be in the same node because they share a disk, so choosing an instance that fits all the services is the first step. This is the one we are using: `c5.xlarge`.

**Installation**: Airbyte was installed following the documentation steps described [here](https://docs.airbyte.com/deploying-airbyte/on-kubernetes/) and the applied `.yaml` files are at the following path: `kube/overlays/stable-with-resource-limits/`.

**Authentication**: The open source version doesn’t have an authentication mechanism. In order to overcome that, we are using the Authelia Project (definitely check it out!) as authentication system. Within this tool, we specify which users have authorization to log in.

**Airbyte sources**: Each source has a different configuration procedure, so we won’t cover this here as it can be found in Airbyte’s documentation. But these are the sources we are actively using: PostgreSQL, GitHub, Google Sheets, Google Ads, Google Analytics, LinkedIn Ads, Pardot, Salesforce, SendGrid, and Facebook Marketing.

**Airbyte destinations**: The destinations correspond to our ClickHouse deployment, but each connection sends data to a different database.

**Airbyte alerting**: When sync attempts go wrong, an alert is triggered and sent to one of our Slack channels.

<br>

### DBT
Despite DBT supporting ClickHouse fully, DBT Cloud specifically does not support running DBT on a ClickHouse instance, and because of that we had to create a structure to run the DBT jobs on our own servers. To handle that, we decided to create a Jenkins job, which gives us more visibility into the logs and also sends alerts in case of failures.

<br>

## Impact and what next ✈️
With this implementation, we were able to reduce a lot of our costs related to proprietary ETL tools without sacrificing effectiveness.

The whole product team has access to dashboards they can query directly from ClickHouse, and this is helping to make our team much more data-driven.

We’d love to hear any feedback you might have about this article. Feel free to get in touch with us via [Instagram](https://www.instagram.com/meetslang/?utm_source=medium&utm_medium=medium-internal&utm_campaign=organic) or [LinkedIn](https://www.linkedin.com/company/meetslang?utm_source=medium&utm_medium=medium-internal&utm_campaign=organic).

If you’re interested in what we’re doing here, we’re hiring across our engineering and product teams! You can see all of our open roles [here](https://slangapp.com/careers?utm_source=medium&utm_medium=medium-internal&utm_campaign=organic).
