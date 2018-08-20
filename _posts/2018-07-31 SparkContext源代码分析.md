#SparkContext.scala

##初始化过程

##LiveListenerBus
spark event的总线
异步方式将spark event分发给所有的listener

##spark env 初始化
### RpcEnv
### MapOutputTracker
### shuffleManager
### BlockManager
### MemoryManager(StaticMemoryManager, UnifiedMemoryManager)

##JobProgressListener

##HeartbeatReceiver
跟踪task级别的信息输出到SparkUI
实现SparkListener，可以监控stage、task、job

## SchedulerBackend
##TaskScheduler
非local模式
spark context 初始化根据master url创建 scheduler
```scala
      case masterUrl =>
        val cm = getClusterManager(masterUrl) match {
          case Some(clusterMgr) => clusterMgr
          case None => throw new SparkException("Could not parse Master URL: '" + master + "'")
        }
        try {
          val scheduler = cm.createTaskScheduler(sc, masterUrl)
          val backend = cm.createSchedulerBackend(sc, masterUrl, scheduler)
          cm.initialize(scheduler, backend)
          (backend, scheduler)
        } catch {
          case se: SparkException => throw se
          case NonFatal(e) =>
            throw new SparkException("External scheduler cannot be instantiated", e)
        }
```

##DAGScheduler
1、面向stage的调度器，为每一个JOB计算一个DAG图，包含多个stage。
2、计算出一个最小的调度执行计划。
3、将stages以task set的形式提交给task scheduler（根据所运行的集群实现）
4、task set包含多个完全独立的任务，可以立即基于集群中的数据run
5、如果数据不可用，task失败。
6、stage是根据依赖划分，宽依赖窄依赖
7、dagscheduler 根据当前数据缓存状态决定task在哪个executor运行，并将数据传给task scheduler
8、dags 还处理因为shuffle out files 丢失导致的失败。这种情况依赖stage可能需要重新计算
9、stage 内部的task失败，需要有task scheduler处理。重试
10、同一个stage可能running多次，叫attempts
11、如果taskscheder报告task因为map out file lost而失败，dags需要重新submit stage