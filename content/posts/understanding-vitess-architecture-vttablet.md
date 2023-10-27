---
title: "Understanding Vitess Architecture Vttablet"
date: 2023-06-24T20:38:04+08:00
draft: true
---

go/cmd/vttablet/vttablet.go

注册gRPC服务器

TabletServer -- query service

proto/queryservice.proto

TabletManager

proto/topodata.proto

TabletType

- SPARE
- REPLICA
- RDONLY