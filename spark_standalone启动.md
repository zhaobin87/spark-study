#Spark standalone启动(1.5)
##Script:
start-all.sh  
--> stat-master.sh/start-slaves.sh  

	标注:
	MATER_CLASS=org.apache.spark.deploy.master.Master  
	WORKER_CLASS=org.apache.spark.deploy.worker.Worker   
	
--> bin/spark-class ${MATER_CLASS}/${WORKER_CLASS} args  
--> java -cp CLASS_PATH org.apache.spark.launcher.Main ${MATER_CLASS/WORKER_CLASS} args  

> ** *总结:bash根据master/worker的不同把响应的full class name作为参数带入了org.apache.spark.launcher.Main中,同样带入的还有其他参数* **

##JAVA/SCALA:  
org.apache.spark.launcher.Main中根据第一个参数是否为(“org.apache.spark.deploy.SparkSubmit” 注:#spark-submit提交任务时携带的full class name#)创建了不同的AbstractCommandBuilder  

在这里创建的为SparkClassCommandBuilder(className,args) 在builder.buildCommand(env)  

其中env是个HashMap<String, String>#中根据不同的环境和参数设置了一系列参数以及环境变量并返回command,然后Main把这些command依次写出(System.out.print)

##Script:
spark-class接收到写会的命令并组装好,用shell命令  exec "${CMD[@]}"  来执行命令提交程序

> ** *总结:上面程序利用脚本和org.apache.spark.launcher.Main交叉组成了提交master/worker的命令以及各种环境变量以及参数,并执行.其中也隐含了spark-submit提交任务的流程.真正提交的主程序为org.apache.spark.deploy.master.Master/Worker
注:在submit-class中主程序为org.spark.deploy.SparkSubmit* **

##Master:  
#####Object Master Main方法:
主要调用了一个函数startRpcEnvAndEndpoint,该函数创建了RpcEnv默认使用的是   
1. `val securityMgr = new SecurityManager(conf) `安全机制--忽略  
2. `val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)`创建了rpc环境,默认使用的是`org.apache.spark.rpc.akka.AkkaRpcEnvFactory`创建的`org.apache.spark.rpc.akka.AkkaRpcEnv`  
3. `val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
      new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))`   


####AkkaRpcEvn setupEndpoint方法
在rpcEnv.setupEndpoint方法中,方法签名是`setupEndpoint(name: String, endpoint: RpcEndpoint): RpcEndpointRef`,在这个方法中   

- 首先创建了ActorRef实例:`lazy val actorRef = actorSystem.actorOf(Props(new Actor with ActorLogReceive with Logging{...}),name)`其中Actor的匿名实例重写了preStart()方法,该方法中调用了endpoint.onStart()方法,也就是master.onStart方法,master由此启动. ``(Akka的Actor对象一旦被初始化就会回调preStart方法) ``

- 继而使用用`endpointRef = new AkkaRpcEndpointRef(defaultAddress, actorRef, conf, initInConstructor = false)`创建了响应的endpoineRef实例,`registerEndpoint(endpoint, endpointRef)`在RpcEnv中的两个map中注册了endpoint以及endpointRef两者的关联关系  

- endpointRef.init

- 最后返回了endpointRef 

####RpcEndpoint注解
Mater类是继承了ThreadSafeRpcEndpoint特质的类,而ThreadSafeRpcEndpoint是继承了RpcEndpoint特质,在RpcEndpoint中有以下接口  
      

	def receive: PartialFunction[Any, Unit]
	def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit]
	def onError(cause: Throwable): Unit
	def onStart(): Unit
	def onStop(): Unit
	def onConnected(remoteAddress: RpcAddress): Unit
	def onDisconnected(remoteAddress: RpcAddress): Unit
	def onNetworkError(cause: Throwable, remoteAddress: RpcAddress): Unit
      
以及已经实现的方法
	
	final def stop(): Unit = {
      val _self = self
      if (_self != null) {
        rpcEnv.stop(_self)
      }
    }
在RpcEndpoint特质的doc上我们可以看到

	/**
	* An end point for the RPC that defines what functions to trigger given a message.
 	* It is guaranteed that `onStart`, `receive` and `onStop` will be called in sequence.
 	* The life-cycle of an endpoint is:
 	* constructor -> onStart -> receive* -> onStop
 	* If any error is thrown from one of [[RpcEndpoint]] methods except `onError`,`onError` will be
 	* invoked with the cause. If `onError` throws an error, [[RpcEnv]] will ignore it.
	*/

receive方法的doc  
`Process messages from [[RpcEndpointRef.send]] or [[RpcCallContext.reply)]]. If receiving a unmatched message, [[SparkException]] will be thrown and sent to onError.`

receiveAndReply方法的doc
	
`Process messages from [[RpcEndpointRef.ask]]. If receiving a unmatched message,[[SparkException]] will be thrown and sent to onError.`

从两个方法的doc可以看到receive以及recieveAndReply分别处理的请求
####Class master onStart方法:

1. 启动了webui
2. 启动了rest server
3. 启动了masterMetricsSystem以及applicationMetricsSystem
4. 根据spark.deploy.recoveryMode参数创建了persistenceEngine以及leaderElectionAgent `用于Master的HA`

Worker的启动和Master大同小异,在Class Worker的onStart函数中调用了registerWithMaster方法,这个方法最终触发发送akka信息到master上进行注册,然后master注册完之后再发回一个akka信息,worker进行处理并开始每隔HEARTBEAT_MILLIS毫秒给master发送心跳,master接到心条之后检查该worker是否已经注册,如果注册了就更新worker信息,如果没有注册则再次寻找该workid,若找到则打印日志并给worker发送重新注册的信息,如果没有则打印日志,不在给该worker发信息


>更多资料  
>[Spark源码分析（一）-Standalone启动过程](http://www.cnblogs.com/tovin/p/3858065.html?utm_source=tuicool&utm_medium=referral)  
>[Spark的Rpc模块的学习](http://www.cnblogs.com/gaoxing/p/4805943.html)
	
