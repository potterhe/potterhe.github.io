---
title: "Vitess Architecture VTGate"
date: 2023-06-23T07:25:59+08:00
draft: true
---

## VTGate

[VTGate](https://vitess.io/docs/16.0/concepts/vtgate/) 是一个轻量级代理服务器，它将流量路由到正确的 [VTTablet](https://vitess.io/docs/16.0/concepts/tablet/) 服务器并将合并的结果返回给客户端。 它支持 MySQL 协议和 Vitess gRPC 协议。 因此，您的应用程序可以连接到 VTGate，就像它是 MySQL 服务器一样。

## MySQL Server

### go/vt/vtgate/plugin_mysql_server.go

initMySQLProtocol() -> 	servenv.OnRun(initMySQLProtocol)
结构体 vtgateHandler 实现了 Handler 接口

mysqlListener.Accept()

#### mysql.server.go
go handle(conn)
分配 connBufferPooling sync.Pool bufio.Reader

vtgateHandler.NewConnection

// First build and send the server handshake packet.
writeHandshakeV10

readEphemeralPacketDirect

parseClientHandshakePacket -> user , schemaName

tls

negotiateAuthMethod -> AuthServer

//Set initial db name

writeOKPacket

### go/mysql/server.go

结构体 Listener 
interface Handler

## Vitess gRPC 协议

gRPC 定义

proto/vtgate.proto

## 启动过程

### go/cmd/vtgate/vtgate.go

servenv.RunDefault()

### go/vt/servenv/run.go

servenv.Run(port)

onRunHooks.Fire()

### Hook

https://plantuml.com/zh/activity-diagram-beta