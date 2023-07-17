---
title: "数据集成 - Sharding"
date: 2023-06-15T10:14:31+08:00
draft: false
---


## 工具

- [Flink CDC](https://ververica.github.io/flink-cdc-connectors/)
- [SeaTunnel](https://seatunnel.apache.org/) - [MySQL-CDC](https://seatunnel.apache.org/docs/2.3.1/connector-v2/source/MySQL-CDC)
- dts

## 文档

- [基于 Flink CDC 同步 MySQL 分库分表构建实时数据湖](https://ververica.github.io/flink-cdc-connectors/master/content/%E5%BF%AB%E9%80%9F%E4%B8%8A%E6%89%8B/build-real-time-data-lake-tutorial-zh.html)
- [多库多表场景下使用Amazon EMR CDC实时入湖最佳实践](https://aws.amazon.com/cn/blogs/china/best-practice-of-using-amazon-emr-cdc-to-enter-the-lake-in-real-time-in-a-multi-database-multi-table-scenario/)


## Doris

### flink-doris-connector

- [Flink Doris Connector](https://doris.apache.org/zh-CN/docs/dev/ecosystem/flink-doris-connector)
- [使用FlinkCDC接入多表或整库示例](https://github.com/apache/doris-flink-connector/blob/master/flink-doris-connector/src/main/java/org/apache/doris/flink/tools/cdc/CdcTools.java)
- [CDCSchemaChangeExample](https://github.com/apache/doris-flink-connector/blob/release-1.1.1/flink-doris-connector/src/test/java/org/apache/doris/flink/CDCSchemaChangeExample.java)
- [DDL 毫秒级同步，Light Schema Change 的设计与实现｜新版本揭秘](https://my.oschina.net/u/5735652/blog/5591562)

```

```

## flink

- [File Systems：Hadoop/Presto S3 文件系统插件](https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/deployment/filesystems/s3/)
https://nightlies.apache.org/flink/flink-docs-release-1.17/docs/deployment/cli/
https://nightlies.apache.org/flink/flink-docs-master/zh/docs/dev/datastream/application_parameters/


### flink-kubernetes-operator

- [flink-kubernetes-operator](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.5/docs/try-flink-kubernetes-operator/quick-start/)
- [CRD Reference](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/custom-resource/reference/)
- [Job Lifecycle Management](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/custom-resource/job-management/)
- [Kubernetes HA](https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/ha/kubernetes_ha/)

### flink-cdc

[MySQL CDC Connector](https://ververica.github.io/flink-cdc-connectors/master/content/connectors/mysql-cdc.html)

## SeaTunnel

https://doris.apache.org/zh-CN/docs/dev/ecosystem/seatunnel
https://seatunnel.apache.org/docs/2.3.1/start-v2/kubernetes/


## xx

- [Flink kubernetes deployment - how to provide S3 credentials from Hashicorp Vault?](https://stackoverflow.com/questions/73579176/flink-kubernetes-deployment-how-to-provide-s3-credentials-from-hashicorp-vault)
