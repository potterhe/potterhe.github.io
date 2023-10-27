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

```
  - args:
    - --buffer_max_failover_duration=10s
    - --buffer_min_time_between_failovers=20s
    - --buffer_size=1000
    - --cell=edu
    - --cells_to_watch=edu
    - --enable_buffer=true
    - --grpc_max_message_size=67108864
    - --grpc_port=15999
    - --logtostderr=true
    - --mysql_auth_server_impl=static
    - --mysql_auth_server_static_file=/vt/secrets/vtgate-static-auth/users.json
    - --mysql_auth_static_reload_interval=30s
    - --mysql_server_port=3306
    - --port=15000
    - --schema_change_signal_user=schema_user
    - --service_map=grpc-vtgateservice
    - --tablet_types_to_wait=MASTER,REPLICA
    - --topo_global_root=/vitess/vitess-cluster/global
    - --topo_global_server_address=vitess-cluster-etcd-9819e775-client.vitess.svc:2379
    - --topo_implementation=etcd2
    - --transaction_mode=SINGLE
```

`--topo_implementation`
各topoServer的实现通过init机制注册，当前实现有consul, etcd2, k8s, zk2
topo.RegisterFactory("etcd2", Factory{})
tabletconn.RegisterDialer("grpc", grpctabletconn.DialTablet)

topo.Open() <- topo.Server
    OpenServer
        NewWithFactory -- `topo.Factory` interface
            factory.Create -- etcd2topo.Factory.Create <- `topo.Conn` interface
                etcd2topo.NewServer
                    etcd2topo.NewServerWithOpts implements topo.Conn
            NewStatsConn
            ret topo.Server

srvtopo.NewResilientServer(ts, "ResilientSrvTopoServer") 
vtgate.Init (hc = nil)
    NewTabletGateway <- TabletGateway
        serv.GetTopoServer <- topo.Server
        createHealthCheck
            discovery.NewHealthCheck <- HealthCheck -- HealthCheckImpl
                NewCellTabletsWatcher <- TopologyWatcher
                    NewTopologyWatcher
                tw.Start
                    tw.loadTablets
                        tw.getTablets
                            tw.topoServer.GetTabletAliasesByCell
                                ts.ConnForCell <- topo.Conn
                                    ts.GetCellInfo
                                    ts.factory.Create
                                conn.ListDir
                                topoproto.ParseTabletAlias < cell/uid
                        loop
                        tw.topoServer.GetTablet
                            ts.ConnForCell
                            conn.Get
                        end

                        loop
                        tw.healthcheck.AddTablet
                            tablet.grpc 必须打开
                            thc := tabletHealthCheck
                            thc.SimpleCopy  <- TabletHealth
                            hc.broadcast
                            go thc.checkConn(hc)
                                thc.stream
                                    thc.Connection <- `queryservice.QueryService`
                                        thc.connectionLocked
                                            tabletconn.GetDialer <- TabletDialer `grpctabletconn.DialTablet`
                                            grpctabletconn.DialTablet()
                                                ret gRPCQueryClient
                                    
                                    grpctabletconn.gRPCQueryClient.StreamHealth
                                        gRPC -> vttablet: StreamHealth stream
                                        loop
                                        stream.Recv
                                        thc.processResponse
                                            thc.setServingState
                                            hc.updateHealth(thc.SimpleCopy())
                                                hc.broadcast(th)
                                                    hc.subscribers <- th
                                        end
                        end
            
        setupBuffering 与选项`--buffer_implementation`相关，默认`keyspace_events`
            discovery.NewKeyspaceEventWatcher <- KeyspaceEventWatcher
                run
                    HealthCheck.Subscribe

            KeyspaceEventWatcher.Subscribe
                赋值 subs
            TabletGateway.buffer.HandleKeyspaceEvent (Goroutine)
        gw.QueryService
    gw.RegisterStats
    gw.WaitForTablets
    选项`--schema_change_signal`


servenv.RunDefault()

### go/vt/servenv/run.go

servenv.Run(port)

onRunHooks.Fire()

### Hook

### go/vt/vtgate/vtgate.go

结构体 VTGate


https://plantuml.com/zh/activity-diagram-beta
