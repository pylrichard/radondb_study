## 入口

bench\benchyou.go

### 保存参数

```
rootCmd.PersistentFlags().IntVar(&writeThreads, "write-threads", 32, "number of write threads to use(Default 32)")
```

### 解析参数

xcmd\common.go\parseConf()

在命令中调用parseConf()解析得到参数

### 添加命令处理

```
rootCmd.AddCommand(xcmd.NewSeqCommand())
```

## 命令处理流程

### seq

xcmd\seq.go

seqCommandFn 对应seq命令

xcmd\common.go

```
func start(conf *xcommon.Conf) {
	var workers []xworker.Worker
	wthds := conf.WriteThreads
	rthds := conf.ReadThreads
	uthds := conf.UpdateThreads
	dthds := conf.DeleteThreads

	/*
		创建读写/更新/删除Worker
	 */
	iworkers := xworker.CreateWorkers(conf, wthds)
	insert := sysbench.NewInsert(conf, iworkers)
	workers = append(workers, iworkers...)

	qworkers := xworker.CreateWorkers(conf, rthds)
	query := sysbench.NewQuery(conf, qworkers)
	workers = append(workers, qworkers...)

	uworkers := xworker.CreateWorkers(conf, uthds)
	update := sysbench.NewUpdate(conf, uworkers)
	workers = append(workers, uworkers...)

	dworkers := xworker.CreateWorkers(conf, dthds)
	delete := sysbench.NewDelete(conf, dworkers)
	workers = append(workers, dworkers...)

	// 创建Monitor，监控所有Worker
	monitor := NewMonitor(conf, workers)

	/*
		启动测试
	 */
	insert.Run()
	query.Run()
	update.Run()
	delete.Run()
	monitor.Start()

	/*
		监控测试过程，达到条件阈值conf.MaxRequest停止测试
	 */
	done := make(chan bool)
	go func(i xworker.Handler, q xworker.Handler, u xworker.Handler, d xworker.Handler, max uint64) {
		if max == 0 {
			return
		}

		for {
			time.Sleep(time.Millisecond * 10)
			all := i.Rows() + q.Rows() + u.Rows() + d.Rows()
			if all >= max {
				done <- true
			}
		}
	}(insert, query, update, delete, conf.MaxRequest)

	select {
	case <-time.After(time.Duration(conf.MaxTime) * time.Second):
	case <-done:
	}

	/*
		停止测试
	 */
	insert.Stop()
	query.Stop()
	update.Stop()
	delete.Stop()
	monitor.Stop()
}
```

## iostat

xcmd\common.go

```
func start(conf *xcommon.Conf) {
	monitor := NewMonitor(conf, workers)
	monitor.Start()
}
```

xcmd\monitor.go

```
func NewMonitor(conf *xcommon.Conf, workers []xworker.Worker) *Monitor {
	return &Monitor{
		ios: xstat.NewIOS(conf),
	}
}

func (m *Monitor) Start() {
	m.ios.Start()
}
```

xstat\iostat.go

```
func NewIOS(conf *xcommon.Conf) *IOS {
	return &IOS{
		cmd: "iostat -x -g ALL 1 2",
	}
}

func (v *IOS) Start() {
	go func() {
		for _ = range v.t.C {
			if err := v.fetch(); err != nil {
				log.Printf("iostat.fetch.error[%v]\n", err)
			}
		}
	}()
}
```

## worker

### 创建Worker连接MySQL

xworker\worker.go

```
func CreateWorkers(conf *xcommon.Conf, threads int) []Worker {
	var workers []Worker
	var conn driver.Conn
	var err error

	utf8 := "utf8"
	dsn := fmt.Sprintf("%s:%d", conf.MysqlHost, conf.MysqlPort)
	for i := 0; i < threads; i++ {
		// 创建MySQL连接
		if conn, err = driver.NewConn(conf.MysqlUser, conf.MysqlPassword, dsn, conf.MysqlDb, utf8); err != nil {
			log.Panicf("create.worker.error:%v", err)
		}
		workers = append(workers, Worker{
			S: conn,
			M: &Metric{},
			E: conf.MysqlTableEngine,
			N: conf.OltpTablesCount,
		},
		)
	}
	return workers
}
```

## sysbench

### insert

sysbench\insert.go

```
func (insert *Insert) Run() {
	threads := len(insert.workers)
	for i := 0; i < threads; i++ {
		insert.lock.Add(1)
		go insert.Insert(&insert.workers[i], threads, i)
	}
}
```

