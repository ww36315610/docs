
# 使用cobra来处理系统启动的命令行交互

* main.go中import 	"github.com/cockroachdb/cockroach/pkg/cli"
* 在cli/cli.go中使用init方法对cmd进行初始化

```go
// cli/cli.go
func init() {
	// ...

	cockroachCmd.AddCommand(
		StartCmd,
		initCmd,
		certCmd,
		quitCmd,

		sqlShellCmd,
		userCmd,
		zoneCmd,
		nodeCmd,
		dumpCmd,

		// Miscellaneous commands.
		genCmd,
		versionCmd,
		debugCmd,
	)
}
```
* 在cli/start.go中有StartCmd的定义

```go
// cli/start.go
var StartCmd = &cobra.Command{
	Use:   "start",
	Short: "start a node",
	Long: `
Start a CockroachDB node, which will export data from one or more
storage devices, specified via --store flags.

If no cluster exists yet and this is the first node, no additional
flags are required. If the cluster already exists, and this node is
uninitialized, specify the --join flag to point to any healthy node
(or list of nodes) already part of the cluster.
`,
	Example: `  cockroach start --insecure --store=attrs=ssd,path=/mnt/ssd1 [--join=host:port,[host:port]]`,
	RunE:    MaybeShoutError(MaybeDecorateGRPCError(runStart)),
}
```
* 然后系统才开始通过runStart接手进行系统初始化

# 启动server

启动server主要做了3方面的工作：
* 一堆的参数检查设置、log准备
* new 一个server
* 启动server


这边的写法跟之前spring的做法有点相似，实例的初始化跟使用通过不同的模块提供。
但没有搞懂new start在这边业务逻辑上严谨的区别。

需要学习的三个知识点：
* opentracing
* stopper
* grpc


```go
// cli/start.go
func runStart(cmd *cobra.Command, args []string) error {
	// 使用分布式追踪协议opentracing
	tracer := serverCfg.Settings.Tracer
	sp := tracer.StartSpan("server start")
	ctx := opentracing.ContextWithSpan(context.Background(), sp)

	// Set up the logging and profiling output.
 	// stopper ???
	stopper, err := setupAndInitializeLoggingAndProfiling(ctx)
 	// ...
	// grpc 出场
	grpcutil.SetSeverity(log.Severity_WARNING)
	log.Info(ctx, "starting cockroach node")
	var s *server.Server
	errChan := make(chan error, 1)
	go func() {
		if err := func() error {
			// Instantiate the server.
			s, err = server.NewServer(serverCfg, stopper)
			// Attempt to start the server.
			if err := s.Start(ctx); err != nil {
				// ...
			}
			// ... 一堆的善后处理
	}()
	// ... 应用关闭的处理

	return returnErr
}
```

# new server

这边概念很多，暂时先贴代码，后续需要细化概念

```go
// server/server.go
// NewServer creates a Server from a server.Config.
func NewServer(cfg Config, stopper *stop.Stopper) (*Server, error) {
	// 初始化server
	s := &Server{
		// ...
	}
	// ... 带拦截器的grpc
	s.grpc = rpc.NewServerWithInterceptor(s.rpcContext, s.Intercept())

	// gossip相关初始化
	s.gossip = gossip.New(
		// ...
	)

	// ...A DistSender provides methods to access Cockroach's monolithic, distributed key value store
	s.distSender = kv.NewDistSender(distSenderCfg, s.gossip)
	// ... ???
	s.tcsFactory = kv.NewTxnCoordSenderFactory(
		// ...
	 )
	// ... DB is a database handle to a single cockroach cluster
	s.db = client.NewDBWithContext(s.tcsFactory, s.clock, dbCtx)
	// ... NodeLiveness encapsulates information on node liveness and provides
	// an API for querying, updating, and invalidating node
	// liveness
	s.nodeLiveness = storage.NewNodeLiveness(
		// ...
	)
	// ... StorePool maintains a list of all known stores in the cluster and information on their health.
	s.storePool = storage.NewStorePool(
		// ...
	)

	// RaftTransport handles the rpc messages for raft
	s.raftTransport = storage.NewRaftTransport(
		s.cfg.AmbientCtx, st, storage.GossipAddressResolver(s.gossip), s.grpc, s.rpcContext,
	)
	// ... LeaseManager manages acquiring and releasing per-table leases每个table的租约
	s.leaseMgr = sql.NewLeaseManager(
		// ...
	)

	// ... Start begins a monitoring region.

	// creates a new engine for DistSQL processors to use when the
	// working set is larger than can be stored in memory
	// 这就是我们在数据目录里看到2.0多出来的rocksdb
	tempEngine, err := engine.NewTempEngine(s.cfg.TempStorageConfig)
	// 应用关闭时的回调注册
	s.stopper.AddCloser(tempEngine)
	// Remove temporary directory linked to tempEngine after closing
	// tempEngine.
	s.stopper.AddCloser(stop.CloserFn(func() {
		err = os.RemoveAll(s.cfg.TempStorageConfig.Path)
	}))

	// Set up admin memory metrics for use by admin SQL executors.
	s.adminMemMetrics = sql.MakeMemMetrics("admin", cfg.HistogramWindowInterval())

	// 初始化时间序列DB ???
	s.tsDB = ts.NewDB(s.db, s.cfg.Settings)
	// 初始化时间序列DB对应的server
	s.tsServer = ts.MakeServer(s.cfg.AmbientCtx, s.tsDB, nodeCountFn, s.cfg.TimeSeriesServerConfig, s.stopper)

	// The InternalExecutor will be further initialized later, as we create more
	// of the server's components. There's a circular dependency - many things
	// need an InternalExecutor, but the InternalExecutor needs an ExecutorConfig,
	// which in turn needs many things. That's why everybody that needs an
	// InternalExecutor takes pointers to this one instance.
	sqlExecutor := sql.InternalExecutor{}

	// ... A Node manages a map of stores (by store ID) for which it serves traffic.
	s.node = NewNode(
		storeCfg, s.recorder, s.registry, s.stopper,
		txnMetrics, nil /* execCfg */, &s.rpcContext.ClusterID)
	// SessionRegistry stores a set of all sessions on this node
	s.sessionRegistry = sql.MakeSessionRegistry()
	// Registry creates Jobs and manages their leases and cancelation.
	s.jobRegistry = jobs.MakeRegistry(
		s.cfg.AmbientCtx, s.clock, s.db, &sqlExecutor, &s.nodeIDContainer, st, func(opName, user string) (interface{}, func()) {
			// This is a hack to get around a Go package dependency cycle. See comment
			// in sql/jobs/registry.go on planHookMaker.
			return sql.NewInternalPlanner(opName, nil, user, &sql.MemoryMetrics{}, &execCfg)
		})

	// implements the server for the distributed SQL APIs.
	s.distSQLServer = distsqlrun.NewServer(ctx, distSQLCfg)
	 // allocates and returns a new REST server for administrative APIs.
	s.admin = newAdminServer(s, &sqlExecutor)
	// provides a RESTful status API. ???
	s.status = newStatusServer(
		// ...
	)
	// allocates and returns a new REST server for authentication APIs.
	s.authentication = newAuthenticationServer(s, &sqlExecutor)
	for _, gw := range []grpcGatewayServer{s.admin, s.status, s.authentication, &s.tsServer} {
		gw.RegisterService(s.grpc)
	}
	// initServer manages the temporary init server used during bootstrapping.
	s.initServer = newInitServer(s)
	// VirtualSchemaHolder is a type used to provide convenient access to virtual database and table descriptors
	virtualSchemas, err := sql.NewVirtualSchemaHolder(ctx, st)
	// Set up Executor sql语句的执行着
	s.sqlExecutor = sql.NewExecutor(execCfg, s.stopper)
	// 监听pg sql请求 ???
	s.pgServer = pgwire.MakeServer(
		// ...
	)
	// ...

	sqlExecutor.ExecCfg = &execCfg
	s.execCfg = &execCfg

	s.leaseMgr.SetExecCfg(&execCfg)
	s.leaseMgr.RefreshLeases(s.stopper, s.db, s.gossip)

	s.node.InitLogger(&execCfg)

	return s, nil
}

```

# start server

```go
// server/server.go
// Start starts the server on the specified port, starts gossip and initializes
// the node using the engines from the server's context. This is complex since
// it sets up the listeners and the associated port muxing, but especially since
// it has to solve the "bootstrapping problem": nodes need to connect to Gossip
// fairly early, but what drives Gossip connectivity are the first range
// replicas in the kv store. This in turn suggests opening the Gossip server
// early. However, naively doing so also serves most other services prematurely,
// which exposes a large surface of potentially underinitialized services. This
// is avoided with some additional complexity that can be summarized as follows:
//
// - before blocking trying to connect to the Gossip network, we already open
//   the admin UI (so that its diagnostics are available)
// - we also allow our Gossip and our connection health Ping service
// - everything else returns Unavailable errors (which are retryable)
// - once the node has started, unlock all RPCs.
//
// The passed context can be used to trace the server startup. The context
// should represent the general startup operation.
func (s *Server) Start(ctx context.Context) error {
	httpServer := netutil.MakeServer(s.stopper, tlsConfig, s)

	ln, err := net.Listen("tcp", s.cfg.Addr)

	s.rpcContext.SetLocalInternalServer(s.node)

	httpLn, err := net.Listen("tcp", s.cfg.HTTPAddr)

	s.stopper.RunWorker(workersCtx, func(workersCtx context.Context) {
		<-s.stopper.ShouldQuiesce()
		if err := httpLn.Close(); err != nil {
			log.Fatal(workersCtx, err)
		}
	})

	if tlsConfig != nil {
		// ...
	}

	// 这边有很多的stopper回调事件注册
	s.stopper.RunWorker(workersCtx, func(context.Context) {
		// ...
	})

	// Attempt to set TCP keep-alive on connection. Don't fail on errors.
	setTCPKeepAlive := func(ctx context.Context, conn net.Conn) {
		// ...
	}

	// Also throw the landing page in there. It won't work well, but it's better than a 404.
	// The remaining endpoints will be opened late, when we're sure that the subsystems they
	// talk to are functional.
	s.mux.Handle("/", http.FileServer(&assetfs.AssetFS{
		Asset:     ui.Asset,
		AssetDir:  ui.AssetDir,
		AssetInfo: ui.AssetInfo,
	}))

	// Initialize grpc-gateway mux and context in order to get the /health
	// endpoint working even before the node has fully initialized.
	jsonpb := &protoutil.JSONPb{
		EnumsAsInts:  true,
		EmitDefaults: true,
		Indent:       "  ",
	}
	protopb := new(protoutil.ProtoPb)
	gwMux := gwruntime.NewServeMux(
		gwruntime.WithMarshalerOption(gwruntime.MIMEWildcard, jsonpb),
		gwruntime.WithMarshalerOption(httputil.JSONContentType, jsonpb),
		gwruntime.WithMarshalerOption(httputil.AltJSONContentType, jsonpb),
		gwruntime.WithMarshalerOption(httputil.ProtoContentType, protopb),
		gwruntime.WithMarshalerOption(httputil.AltProtoContentType, protopb),
		gwruntime.WithOutgoingHeaderMatcher(authenticationHeaderMatcher),
	)

	// Setup HTTP<->gRPC handlers.
	c1, c2 := net.Pipe()

	for _, gw := range []grpcGatewayServer{s.admin, s.status, s.authentication, &s.tsServer} {
		if err := gw.RegisterGateway(gwCtx, gwMux, conn); err != nil {
			return err
		}
	}
	s.mux.Handle("/health", gwMux)

	// Engine is the interface that wraps the core operations of a key/value store.
	// Engines is a container of engines, allowing convenient closing.
	s.engines, err = s.cfg.CreateEngines(ctx)

	for _, storeSpec := range s.cfg.Stores.Specs {
		if storeSpec.InMemory {
			continue
		}
		// ???
		for base, val := range listenerFiles {
			file := filepath.Join(storeSpec.Path, base)
			if err := ioutil.WriteFile(file, []byte(val), 0644); err != nil {
				return errors.Wrapf(err, "failed to write %s", file)
			}
		}
	}

	// 根据磁盘上的rocksdb文件，检验文件所属版本跟当前版本是否兼容
	bootstrappedEngines, _, _, err := inspectEngines(
		ctx, s.engines, s.cfg.Settings.Version.MinSupportedVersion,
		s.cfg.Settings.Version.ServerVersion, &s.rpcContext.ClusterID)

	// Signal readiness. This unblocks the process when running with
	// --background or under systemd. At this point we have bound our
	// listening port but the server is not yet running, so any
	// connection attempts will be queued up in the kernel. We turn on
	// servers below, first HTTP and later pgwire. If we're in
	// initializing mode, we don't start the pgwire server until after
	// initialization completes, so connections to that port will
	// continue to block until we're initialized.
	if err := sdnotify.Ready(); err != nil {
		log.Errorf(ctx, "failed to signal readiness using systemd protocol: %s", err)
	}

	// Filter the gossip bootstrap resolvers based on the listen and
	// advertise addresses. 就是从join里把自己摘出来
	filtered := s.cfg.FilterGossipBootstrapResolvers(ctx, unresolvedListenAddr, unresolvedAdvertAddr)
	// bigboss here
	s.gossip.Start(unresolvedAdvertAddr, filtered)

	if len(bootstrappedEngines) > 0 {
		// 处理节点重启，涉及的时钟同步问题，必要时sleep
		if maxOffset := s.clock.MaxOffset(); maxOffset != timeutil.ClocklessMaxOffset {
			sleepDuration = maxOffset - timeutil.Since(startTime)
		}
		if sleepDuration > 0 {
			log.Infof(ctx, "sleeping for %s to guarantee HLC monotonicity", sleepDuration)
			time.Sleep(sleepDuration)
		}
	} else if len(s.cfg.GossipBootstrapResolvers) == 0 {
		// 单节点启动，搞个新集群 这边具体的启动过程，没搞懂???
		if err := s.node.bootstrap(ctx, s.engines, bootstrapVersion); err != nil {
			return err
		}
	} else {
		log.Info(ctx, "no stores bootstrapped and --join flag specified, awaiting init command.")
	}

	// We ran this before, but might've bootstrapped in the meantime. This time
	// we'll get the actual list of bootstrapped and empty engines.
	bootstrappedEngines, emptyEngines, cv, err := inspectEngines(
		ctx, s.engines, s.cfg.Settings.Version.MinSupportedVersion,
		s.cfg.Settings.Version.ServerVersion, &s.rpcContext.ClusterID)
	if err != nil {
		return errors.Wrap(err, "inspecting engines")
	}

	// Now that we have a monotonic HLC wrt previous incarnations of the process,
	// init all the replicas. At this point *some* store has been bootstrapped or
	// we're joining an existing cluster for the first time.
	// 这边会开gossip同步
	if err := s.node.start(
		// ...
	);

	s.refreshSettings()

	raven.SetTagsContext(map[string]string{
		"cluster":   s.ClusterID().String(),
		"node":      s.NodeID().String(),
		"server_id": fmt.Sprintf("%s-%s", s.ClusterID().Short(), s.NodeID()),
	})

	// We can now add the node registry.
	s.recorder.AddNode(s.registry, s.node.Descriptor, s.node.startedAt, s.cfg.AdvertiseAddr, s.cfg.HTTPAddr)

	// Begin recording runtime statistics.
	s.startSampleEnvironment(DefaultMetricsSampleInterval)

	// Begin recording time series data collected by the status monitor.
	s.tsDB.PollSource(
		s.cfg.AmbientCtx, s.recorder, DefaultMetricsSampleInterval, ts.Resolution10s, s.stopper,
	)

	// Begin recording status summaries.
	s.node.startWriteSummaries(DefaultMetricsSampleInterval)

	// Create and start the schema change manager only after a NodeID
	// has been assigned.
	var testingKnobs *sql.SchemaChangerTestingKnobs
	if s.cfg.TestingKnobs.SQLSchemaChanger != nil {
		testingKnobs = s.cfg.TestingKnobs.SQLSchemaChanger.(*sql.SchemaChangerTestingKnobs)
	} else {
		testingKnobs = new(sql.SchemaChangerTestingKnobs)
	}

	// 管理gossip同步过来的table schema变更信息
	sql.NewSchemaChangeManager(
		s.cfg.AmbientCtx,
		s.execCfg,
		testingKnobs,
		*s.db,
		s.node.Descriptor,
		s.execCfg.DistSQLPlanner,
	).Start(s.stopper)

	// Start starts workers for the executor.
	s.sqlExecutor.Start(ctx, s.execCfg.DistSQLPlanner)
	s.distSQLServer.Start()
	// 还是老样子，去gossip那订阅消息干活
	s.pgServer.Start(ctx, s.stopper)

	s.serveMode.set(modeOperational)

	s.mux.Handle(adminPrefix, authHandler)
	s.mux.Handle(ts.URLPrefix, authHandler)
	s.mux.Handle(statusPrefix, authHandler)
	s.mux.Handle(authPrefix, gwMux)
	s.mux.Handle(statusVars, http.HandlerFunc(s.status.handleVars))
	log.Event(ctx, "added http endpoints")

	log.Infof(ctx, "starting %s server at %s", s.cfg.HTTPRequestScheme(), unresolvedHTTPAddr)
	log.Infof(ctx, "starting grpc/postgres server at %s", unresolvedListenAddr)
	log.Infof(ctx, "advertising CockroachDB node at %s", unresolvedAdvertAddr)

	log.Event(ctx, "accepting connections")

	// Begin the node liveness heartbeat. Add a callback which
	// 1. records the local store "last up" timestamp for every store whenever the
	//    liveness record is updated.
	// 2. sets Draining if Decommissioning is set in the liveness record
	decommissionSem := make(chan struct{}, 1)
	// bigboss here
	s.nodeLiveness.StartHeartbeat(ctx, s.stopper, func(ctx context.Context) {
		now := s.clock.Now()
		if err := s.node.stores.VisitStores(func(s *storage.Store) error {
			return s.WriteLastUpTimestamp(ctx, now)
		}); err != nil {
			log.Warning(ctx, errors.Wrap(err, "writing last up timestamp"))
		}

		if liveness, err := s.nodeLiveness.Self(); err != nil && err != storage.ErrNoLivenessRecord {
			log.Warning(ctx, errors.Wrap(err, "retrieving own liveness record"))
		} else if liveness != nil && liveness.Decommissioning && !liveness.Draining {
			select {
			case decommissionSem <- struct{}{}:
				s.stopper.RunWorker(ctx, func(context.Context) {
					defer func() {
						<-decommissionSem
					}()

					// Don't use ctx because there is an associated timeout
					// meant to be used when heartbeating.
					if _, err := s.Drain(context.Background(), GracefulDrainModes); err != nil {
						log.Warningf(ctx, "failed to set Draining when Decommissioning: %v", err)
					}
				})
			default:
				// Already have an active goroutine trying to drain; don't add a
				// second one.
			}
		}
	})

	{
		var regLiveness jobs.NodeLiveness = s.nodeLiveness
		if testingLiveness := s.cfg.TestingKnobs.RegistryLiveness; testingLiveness != nil {
			regLiveness = testingLiveness.(*jobs.FakeNodeLiveness)
		}
		if err := s.jobRegistry.Start(
			ctx, s.stopper, regLiveness, jobs.DefaultCancelInterval, jobs.DefaultAdoptInterval,
		); err != nil {
			return err
		}
	}

	// Before serving SQL requests, we have to make sure the database is
	// in an acceptable form for this version of the software.
	// We have to do this after actually starting up the server to be able to
	// seamlessly use the kv client against other nodes in the cluster.
	var mmKnobs sqlmigrations.MigrationManagerTestingKnobs
	if migrationManagerTestingKnobs := s.cfg.TestingKnobs.SQLMigrationManager; migrationManagerTestingKnobs != nil {
		mmKnobs = *migrationManagerTestingKnobs.(*sqlmigrations.MigrationManagerTestingKnobs)
	}
	migMgr := sqlmigrations.NewManager(
		s.stopper,
		s.db,
		s.sqlExecutor,
		s.clock,
		mmKnobs,
		&s.internalMemMetrics,
		s.NodeID().String(),
	)
	if err := migMgr.EnsureMigrations(ctx); err != nil {
		select {
		case <-s.stopper.ShouldQuiesce():
			// Avoid turning an early shutdown into a fatal error. See #19579.
			return errors.New("server is shutting down")
		default:
			log.Fatal(ctx, err)
		}
	}
	log.Infof(ctx, "done ensuring all necessary migrations have run")
	close(serveSQL)

	log.Info(ctx, "serving sql connections")
	// Start servicing SQL connections.

	pgCtx := s.pgServer.AmbientCtx.AnnotateCtx(context.Background())
	s.stopper.RunWorker(pgCtx, func(pgCtx context.Context) {
		select {
		case <-serveSQL:
		case <-s.stopper.ShouldQuiesce():
			return
		}
		netutil.FatalIfUnexpected(httpServer.ServeWith(pgCtx, s.stopper, pgL, func(conn net.Conn) {
			connCtx := log.WithLogTagStr(pgCtx, "client", conn.RemoteAddr().String())
			setTCPKeepAlive(connCtx, conn)

			var serveFn func(ctx context.Context, conn net.Conn) error
			if !s.cfg.UseLegacyConnHandling {
				serveFn = s.pgServer.ServeConn2
			} else {
				serveFn = s.pgServer.ServeConn
			}
			if err := serveFn(connCtx, conn); err != nil && !netutil.IsClosedConnection(err) {
				// Report the error on this connection's context, so that we
				// know which remote client caused the error when looking at
				// the logs.
				log.Error(connCtx, err)
			}
		}))
	})
	if len(s.cfg.SocketFile) != 0 {
		log.Infof(ctx, "starting postgres server at unix:%s", s.cfg.SocketFile)

		// Unix socket enabled: postgres protocol only.
		unixLn, err := net.Listen("unix", s.cfg.SocketFile)
		if err != nil {
			return err
		}

		s.stopper.RunWorker(workersCtx, func(workersCtx context.Context) {
			<-s.stopper.ShouldQuiesce()
			if err := unixLn.Close(); err != nil {
				log.Fatal(workersCtx, err)
			}
		})

		s.stopper.RunWorker(pgCtx, func(pgCtx context.Context) {
			select {
			case <-serveSQL:
			case <-s.stopper.ShouldQuiesce():
				return
			}
			netutil.FatalIfUnexpected(httpServer.ServeWith(pgCtx, s.stopper, unixLn, func(conn net.Conn) {
				connCtx := log.WithLogTagStr(pgCtx, "client", conn.RemoteAddr().String())
				if err := s.pgServer.ServeConn(connCtx, conn); err != nil &&
					!netutil.IsClosedConnection(err) {
					// Report the error on this connection's context, so that we
					// know which remote client caused the error when looking at
					// the logs.
					log.Error(connCtx, err)
				}
			}))
		})
	}

	// Record that this node joined the cluster in the event log. Since this
	// executes a SQL query, this must be done after the SQL layer is ready.
	s.node.recordJoinEvent()

	if s.cfg.PIDFile != "" {
		if err := ioutil.WriteFile(s.cfg.PIDFile, []byte(fmt.Sprintf("%d\n", os.Getpid())), 0644); err != nil {
			log.Error(ctx, err)
		}
	}

	if s.cfg.ListeningURLFile != "" {
		pgURL, err := s.cfg.PGURL(url.User(security.RootUser))
		if err == nil {
			err = ioutil.WriteFile(s.cfg.ListeningURLFile, []byte(fmt.Sprintf("%s\n", pgURL)), 0644)
		}

		if err != nil {
			log.Error(ctx, err)
		}
	}

	log.Event(ctx, "server ready")

	return nil
}
```
