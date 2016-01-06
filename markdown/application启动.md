#Application启动
略过脚本和SparkSubmit程序,直接看SparkContext类   
###配置: 

`spark.ui.showConsoleProgress`=`true` 可以关闭每次action时终端上显示的stage进度条
`spark.driver.allowMultipleContexts`=`false` 判断是否允许一个JVM启动多个sparkcontext  
`spark.serializer`=`org.apache.spark.serializer.JavaSerializer`  
`spark.closure.serializer`=`org.apache.spark.serializer.JavaSerializer`  
`spark.shuffle.manager`=`sort`  :

     "hash" -> "org.apache.spark.shuffle.hash.HashShuffleManager",
     "sort" -> "org.apache.spark.shuffle.sort.SortShuffleManager",
     "tungsten-sort" -> "org.apache.spark.shuffle.unsafe.UnsafeShuffleManager")
`spark.shuffle.blockTransferService`=`netty`:  

	netty=NettyBlockTransferService  
	nio=NioBlockTransferService  : will be remove in spark 1.6.0
`spark.fileserver.port`=`0`  
`spark.unsafe.offHeap`=`false` #true=MemoryAllocator.UNSAFE || false =MemoryAllocator.HEAP  
`spark.ui.enabled`=`true`  
`spark.executor.memory>SPARK_EXECUTOR_MEMORY>SPARK_MEM`=`1024`  
`spark.dynamicAllocation.enabled`=`false` && `spark.executor.instances`=`0` == `0`  #dynamicAllocationEnabled  根据这两个参数来判断是否启动动态分配策略,默认是关闭的  
`spark.cleaner.referenceTracking`=`true`  
`spark.scheduler.mode`=`FIFO`  
`spark.speculation`=`false`  
`spark.scheduler.revive.interval`=`1s` 关于这个解释引用自网上:[Spark任务调度流程及调度策略分析](http://www.cnblogs.com/barrenlake/p/4891589.html)  
	

##SparkContext:
####1. 实例化过程:
只记录了自认为的重点,有很多遗漏 
1.无条件创建了`private val creationSite: CallSite = Utils.getCallSite()` # __*没搞懂callsite是干什么用的,待解决!!!*__  
2.通过配置`spark.driver.allowMultipleContexts`=`true`允许在一个jvm启动多个sparkcontext而不抛出异常,仅仅是报出warn日志,但是正常情况下一个jvm里只允许一个sparkconstext存在的,可以看一下注释:  
	
	Called at the beginning of the SparkContext constructor to ensure that no SparkContext is running.  Throws an exception if a running context is detected and logs a warning if another thread is constructing a SparkContext.  This warning is necessary because the current locking scheme prevents us from reliably distinguishing between cases where another context is being constructed and cases where another constructor threw an exception.  
3.参数sparkConfig被深度复制到私有变量的_conf中 
4.无条件创建`listenerBus=new LiveListenerBus`   
5.无条件创建JobProgressListener
	
	jobProgressListener = new JobProgressListener(_conf)
	listenerBus.addListener(jobProgressListener)
6.通过 `SparkEnv.createDriverEnv(conf, isLocal, listenerBus, SparkContext.numDriverCores(master))`设置了sparkEnv,返回了以下构造参数构造的sparkEnv对象: 
	
	1. executorId="driver",
    2. rpcEnv,
    3. serializer,
    4. closureSerializer,
    5. cacheManager = new CacheManager,
    6. mapOutputTracker = new MapOutputTrackerMaster(conf),
    7. shuffleManager : org.apache.spark.shuffle.sort.SortShuffleManager,
    8. broadcastManager = new BroadcastManager,
    9. blockTransferService =  new NettyBlockTransferService(conf, securityManager, numUsableCores),//numUsableCores=0 driver is not used for execution
    10. blockManager = new BlockManager,
    11. securityManager = new SecurityManager(conf),
    12. httpFileServer = new HttpFileServer(conf, securityManager, fileServerPort),
    13. sparkFilesDir = Utils.createTempDir(Utils.getLocalDir(conf), "userFiles").getAbsolutePath,
    14. metricsSystem = MetricsSystem.createMetricsSystem("driver", conf, securityManager),
    15. shuffleMemoryManager = ShuffleMemoryManager.create(conf, numUsableCores),
    16. executorMemoryManager = new ExecutorMemoryManager(allocator), #allocator=MemoryAllocator.UNSAFE/MemoryAllocator.HEAP 见spark.unsafe.offHeap
    17. outputCommitCoordinator=new OutputCommitCoordinator(conf, isDriver),
    18. conf = SparkConf
7.`_metadataCleaner = new MetadataCleaner(MetadataCleanerType.SPARK_CONTEXT, this.cleanup, _conf)`  

8.`_statusTracker = new SparkStatusTracker(this)`  

9.`_hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf)` 
 
10.`_heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))`  
      
11.`val (sched, ts) = SparkContext.createTaskScheduler(this, master)`创建TaskScheduler,创建过程是根据master进行match,standalone匹配的是`"""spark://(.*)""".r`,在这个case中:  

	val scheduler = new TaskSchedulerImpl(sc)  
	val masterUrls = sparkUrl.split(",").map("spark://" + _)  
    val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)

12.创建了`_dagScheduler = new DAGScheduler(this)`,继而启动`_taskScheduler.start()`  
13.根据dynamicAllocationEnabled的boolean值true/false来创建_executorAllocationManager 为`Some(new ExecutorAllocationManager(this, listenerBus, _conf))`/`None`并启动  
根据`spark.cleaner.referenceTracking`booleantrue/false来创建_cleaner为`Some(new ContextCleaner(this))`/`None`并启动  
14.

	setupAndStartListenerBus()  #设置listenerbus的listener,并启动listenerBus
    postEnvironmentUpdate()  #通知各个listener更新环境信息
    postApplicationStart()  #通知各个listener系统启动
    _taskScheduler.postStartHook() #检查scheduleBackEnd是否启动成功,如果没有成功,等待100毫秒或者被通知
以上是sparkcontext启动主函数的概要分析,下面主要分析这中间一些重要的模块
######1. TaskSchedulerImpl 实例化
除了使用yarn-client和yarn-cluster以外,在1.5.0版本中其他模式使用的tasksheduler使用的都是TaskSchedulerImpl. *创建方式见`上11`*  
`taskResultGetter = new TaskResultGetter(sc.env, this)`  
######2. SparkDeploySchedulerBackend 实例化
在spark的各种部署模式中只有standalone模式以及local[N, maxRetries]模式的shedulerbackend采用SparkDeploySchedulerBackend类,该类继承了CoarseGrainedSchedulerBackend以及AppClientListener,这里主要讲CoarseGrainedSchedulerBackend,实例化过程中只是读取一些配置和声明一些变量.
######3.DAGScheduler 实例化  
简单粗暴,直接`new DAGScheduler(this)`,这中间做了以下事情:  
`metricsSource: DAGSchedulerSource = new DAGSchedulerSource(this)`  
`val outputCommitCoordinator = env.outputCommitCoordinator`
`taskScheduler.setDAGScheduler(this)` taskScheduler和dagScheduler进行了关联,dagscheduler的实例化是在taskscheduler后面,下面会讲到

####2. TaskSchedulerImpl,SparkDeploySchedulerBackend,DAGScheduler  
#####1. 实例化顺序:
抽离其他代码,合并后如下:
	
	val scheduler = new TaskSchedulerImpl(sc)
    val masterUrls = sparkUrl.split(",").map("spark://" + _)
    val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
    scheduler.initialize(backend)
    _dagScheduler = new DAGScheduler(this)
    _taskScheduler.start()
#####2. 内部代码解读:
1.`scheduler.initialize(backend)`:  
taskscheduler和schedulerbackend建立了关联,根据调度模式初始花了一个`rootPool=Pool("", schedulingMode, 0, 0)`,并根据调度模式建立了schedulableBuilder:
	
	schedulableBuilder = {
      schedulingMode match {
        case SchedulingMode.FIFO =>
          new FIFOSchedulableBuilder(rootPool)
        case SchedulingMode.FAIR =>
          new FairSchedulableBuilder(rootPool, conf)
      }
    }
    schedulableBuilder.buildPools()
2.`_taskScheduler.start()`:  
首先启动了backend:`backend.start()`
又根据参数启动了一个定时线程,推测是否有任务需要调度
	
	if (!isLocal && conf.getBoolean("spark.speculation", false)) {
      logInfo("Starting speculative execution thread")
      speculationScheduler.scheduleAtFixedRate(new Runnable {
        override def run(): Unit = Utils.tryOrStopSparkContext(sc) {
          checkSpeculatableTasks()
        }
      }, SPECULATION_INTERVAL_MS, SPECULATION_INTERVAL_MS, TimeUnit.MILLISECONDS)
    }
    
     // Check for speculatable tasks in all our active jobs.
  	def checkSpeculatableTasks() {
      var shouldRevive = false
      synchronized {
        shouldRevive = rootPool.checkSpeculatableTasks()
      }
      if (shouldRevive) {
        backend.reviveOffers()
      }
    }
3.`backend.start()`  
SparkDeploySchedulerBackend.start()首先调用的是CoarseGrainedSchedulerBackend.start():`super.start()`  
首先.创建了driverEndpoint:RpcEndpintRef,其中的rpcEndpoint为DriverEndpoint,DriverEndpoint在它的onStart方法中启动了一个定时进程,循环定时发送ReviveOffers信息,详情ctrl+f `spark.scheduler.revive.interval`  
其次.搞了一系列参数,这些参数更多是`executor:org.apache.spark.executor.CoarseGrainedExecutorBackend`的参数,实例化了以下类:  
	
	val command = Command("org.apache.spark.executor.CoarseGrainedExecutorBackend",args, sc.executorEnvs, classPathEntries ++ testingClassPath, libraryPathEntries, javaOpts)
	val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory,command, appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor)
    client = new AppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
    client.start()
4.AppClient.start()  
创建了clientendpoint:ClientEndpoint 的rpcEndpointRef,akka内部调用了clientendpoint.onStart()方法,这个方法中调用了`registerWithMaster(1)`这里向master发送了`RegisterApplication(appDescription, self)`信息,master接收到之后创建了application并注册,然后发送给applclient `RegisteredApplication(app.id, self)`信息,然后调用scheduler()方法  
5.`schedule()`  
<font color="red">__下面代码关于driverinfo的用途没有看懂,在这里mark一下__</font>
	
	private def schedule(): Unit = {
		if(state != RecoveryState.ALIVE) { return }
    	// Drivers take strict precedence over executors
		val shuffledWorkers = Random.shuffle(workers) // Randomization helps balance drivers
    	for (worker <- shuffledWorkers if worker.state == WorkerState.ALIVE) {
      	for (driver <- waitingDrivers) {
        	if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
         	 launchDriver(worker, driver)
        	  waitingDrivers -= driver
       	 }
      	}
   	 }
   	 startExecutorsOnWorkers()
	}
这段代码最后执行了`startExecutorsOnWorkers()`  
6`startExecutorsOnWorkers()` 
上代码:
	
	private def startExecutorsOnWorkers(): Unit = {
    // Right now this is a very simple FIFO scheduler. We keep trying to fit in the first app
    // in the queue, then the second app, etc.
    for (app <- waitingApps if app.coresLeft > 0) {
      val coresPerExecutor: Option[Int] = app.desc.coresPerExecutor
      // Filter out workers that don't have enough resources to launch an executor
      val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
        .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
          worker.coresFree >= coresPerExecutor.getOrElse(1))
        .sortBy(_.coresFree).reverse
      val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)

      // Now that we've decided how many cores to allocate on each worker, let's allocate them
      for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
        allocateWorkerResourceToExecutors(
          app, assignedCores(pos), coresPerExecutor, usableWorkers(pos))
      }
     }
    }  
在这里,遍历有所有`waitingApps:ArrayBuffer[ApplicationInfo]`中所有`app.coresLeft>0`的app,并在内部遍历所有worker,根据woker的状态是否存活`_.state == WorkerState.ALIVE`以及worker的内存是否大于等于每个execuotr分配的内存`worker.memoryFree >= app.desc.memoryPerExecutorMB`且woker剩余的cores是否大约等于每个executor分配的cores(默认为个cores)`worker.coresFree >= coresPerExecutor.getOrElse(1)`查找出了可用的woker,并把这些woker按照剩余cores的多少降序排序.  

` val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)`用相当精巧切复杂的方式计算了可用的woker上分配的cores数量,然后就是遍历可用woker并且woker里分配的cores大于0,去把worker的资源分配给executor

`allocateWorkerResourceToExecutors`
	

	private def allocateWorkerResourceToExecutors(
		app: ApplicationInfo,
		ssignedCores: Int,
		coresPerExecutor: Option[Int],
		worker: WorkerInfo): Unit = {
		// If the number of cores per executor is specified, we divide the cores assigned
		// to this worker evenly among the executors with no remainder.
		// Otherwise, we launch a single executor that grabs all the assignedCores on this worker.
		val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
		val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
		for (i <- 1 to numExecutors) {
			val exec = app.addExecutor(worker, coresToAssign)
			launchExecutor(worker, exec)
			app.state = ApplicationState.RUNNING
		}
	}
	
	private def launchExecutor(worker: WorkerInfo,exec: ExecutorDesc): Unit = {
		logInfo("Launching executor " + exec.fullId + " on worker " + worker.id)
    	worker.addExecutor(exec)
    	worker.endpoint.send(LaunchExecutor(masterUrl,exec.application.id, exec.id, exec.application.desc, exec.cores, exec.memory))
    	exec.application.driver.send(ExecutorAdded(exec.id, worker.id, worker.hostPort, exec.cores, exec.memory))
	}
上面就是在woker上分配资源并用Worker的RpcEndpointRef发送了LaunchExecutor以及用AppClient的RpcEndpointRef发送了ExecutorAdded信息

`AppClient`接收到ExecutorAdded信息后调用了listener.executorAdded方法,这个listener是AppClient在SparkDeploySchedulerBackend中初始化的时候传入的this也就是SparkDeploySchedulerBackend实现了AppClientListener接口

`Worker`接收到了LaunchExecutor后创建了app的本地目录,创建ExecutorRunner并执行了其`start()`方法,然后`sendToMaster(ExecutorStateChanged(appId, execId, manager.state, None, None))`

在`ExecutorRunner.start()`中: 根据ApplicationDescription构建了executor进程所需要的命令以及参数,启动进程并把进程的输出写入到准备好的本地文件中.`ApplicationDescription`是在`SparkDeploySchedulerBackend`中创建的,见上文.这里启动的进程引用的类是`org.apache.spark.executor.CoarseGrainedExecutorBackend` **至此:启动了CoarseGrainedExecutorBackend**

在Master接收到`ExecutorStateChanged`后,又通知了AppClient `ExecutorUpdated`,Appclient接收到这个信息后,根据传入的状态做了不同操作,在这里只是打印日志,当`ExecutorState.isFinished(state)`时候触发`listener.executorRemoved(fullId, message.getOrElse(""), exitStatus)`

####3.CoarseGrainedExecutorBackend启动:
1.获取了一系列参数,在ojbect CoarseGrainedExecutorBackend 根据传入的`--driver-url`获取了`CoarseGrainedSchedulerBackend`的rpcEndpointRef,并发送了`RetrieveSparkProps`返回的是props信息,之后的重要代码如下:

    env.rpcEnv.setupEndpoint("Executor", new CoarseGrainedExecutorBackend(env.rpcEnv,                     
        driverUrl,executorId, sparkHostPort, cores, userClassPath, env))
    workerUrl.foreach { url =>
        env.rpcEnv.setupEndpoint("WorkerWatcher", new WorkerWatcher(env.rpcEnv, url))
      }
2.`CoarseGrainedExecutorBackend.onStart()`  
创建了`CoarseGrainedSchedulerBackend`的rpcEndpointRef,发送需要反馈的信息`RegisterExecutor`,反馈信息为`RegisteredExecutor`,然后把这个消息发送给自身,自己拿到这个信息后创建了`executor = new Executor(executorId, hostname, env, userClassPath, isLocal = false)`

3.`WorkerWatcher.onStart()` do nothing

至此,马马虎虎结束,很多地方没有分析到
>更多更好文档:  
>[Spark任务调度流程及调度策略分析](http://www.cnblogs.com/barrenlake/p/4891589.html)  