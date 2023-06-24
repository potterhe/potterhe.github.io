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
结构体 vtgateHandler 实现了 Handler 接口, 有一部分未实现，在UnimplementedHandler里标记

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

慢连接 SlowConnectWarnThreshold

ConnectionReady(未做实现)

循环处理

Conn.handleNextCommand

    readEphemeralPacket
    handleComQuery
    parseComQuery
        多语句切分
    loop
        execQuery
            handler.ComQuery
                callinfo, callerid
                olap -> vtg.StreamExecute
                vtg.Execute
                    vtg.executor.Execute
                        Execute.execute
                               .newExecute
                                    startTxIfNecessary
                                    newVCursorImpl
                                    parseAndValidateQuery
                                    getPlan -> engine.Plan
                                        PrepareAST
                                        cacheAndBuildStatement
                                            planbuilder.BuildFromStmt
                                                createInstructionFor -> planResult
                                                    stmt.(type)
                                                        CURD
                                                            buildSelectPlan
                                                                checkUnsupportedExpressions
                                                                handleDualSelects -> engine.Primitive
                                                                getPlan -> logicalPlan
                                                                    primitiveBuilder.processSelect
                                                            buildRoutePlan
                                                planResult.primitive

                                    executePlan
                                        Executor.executePlan -> qr(查询结果)
                                            vcursor.ExecutePrimitive
                                                primitive.TryExecute // 调用 vttablet route.go
                                                    executeInternal
                                                        findRoute -- srvtopo.ResolvedShard vindex
                                                        executeShards
                                                            vcursor.ExecuteMultiShard
                                                                vc.executor.ExecuteMultiShard -- Executor.ExecuteMultiShard
                                                                    Executor.ScatterConn.ExecuteMultiShard -- ScatterConn
                                                                        multiGoTransaction
                                                                            getQueryService -- queryservice.QueryService
                                                                                rs.Gateway.QueryServiceByAlias -- TabletGateway
                                                                                    HealthCheckImpl.TabletConnection
                                                                                        tabletHealthCheck.Connection
                                                                                            tabletHealthCheck.connectionLocked
                                                                                                grpctabletconn.DialTablet -> gRPCQueryClient
                                                                                                    grpcclient.Dial
                                                                            qs.Execute -- TabletGateway
                                                                                gRPCQueryClient.Execute
                                                                                    queryservicepb.QueryClient.Execute -- gRPC -- tablet
                                                                                        Proto3ToResult
                                                        OrderBy
                                                    qr.Truncate
    end

### go/mysql/server.go

结构体 Listener 
interface Handler

### go/mysql/conn.go

type Conn struct


### go/vt/vtgate/executor.go

Executor

### go/vt/vtgate/engine/primitive.go
Primitive interface
    vcursorImpl 实现了这个接口

## Vitess gRPC 协议

gRPC 定义

proto/vtgate.proto

## 启动过程

### go/cmd/vtgate/vtgate.go

vtgate.Init
servenv.RunDefault()

### go/vt/servenv/run.go

servenv.Run(port)

onRunHooks.Fire()

### Hook

### go/vt/vtgate/vtgate.go

结构体 VTGate


https://plantuml.com/zh/activity-diagram-beta
