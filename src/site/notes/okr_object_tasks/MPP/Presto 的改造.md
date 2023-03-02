---
{"dg-publish":true,"permalink":"/okr-object-tasks/mpp/presto/"}
---


## 架构
adhoc, BI, dqc ，数据探查服务
$\Downarrow$
dispatcher 路由服务用来进行 SQL 统一调度
$\Downarrow$
dispatcher 结合以下条件，动态决定选择最合适的计算引擎执行
* 查询语句的语法特性
* 库表读 HDFS 的数据量
* 引擎的负载情况
汐但是
![Pasted image 20230302055630.png](/img/user/Pasted%20image%2020230302055630.png)

*架构解释*：
* **我们的ADHOC、BI、DQC以及数据探查等服务都是通过自研的Dispatcher路由服务来进行统一SQL调度**
* **Dispatcher会结合查询语句的语法特征，库表读HDFS的数据量以及引擎的负载情况等因素动态决定选择最合适的计算引擎执行**
* **如果是Hive的语法需要用Presto执行的，目前利用Linkedin开源的coral对语句进行转化，如果执行的SQL失败了会做引擎自动降级，降低用户使用门槛；**
* 调度平台支持Spark、Hive、Presto cli的方式提交ETL作业到Yarn进行调度；
* **Presto gateway服务负责对Presto进行多集群管理和路由；**
* 我们通过Kyuubi来提供多租户能力，对不同部门管理各自的Spark Engine，提供adhoc查询；
* 目前各大引擎Hive、Spark、Presto以及HDFS都接入了Ranger来进行权限认证，HDFS通过路径来控制权限，计算引擎通过库/表来控制权限，并且我们通过一套Policy来实现表, column masking和row filter的权限控制；
* 部分组件比如Presto、Alluxio、Kyuubi、Presto Gateway、Dispatcher, 包括混部Yarn集群都已经通过公司k8s平台统一部署调度。




## Presto 线上应用

* 分布式
* MPP(Massive Parallel Processing)架构的SQL查询引擎。
* 全内存计算(部分算子数据也可通过session 配置spill到本地磁盘)
* 流式 pipeline 处理数据  节省内存  更快的响应查询

**优势**（相比 hive spark）：
* shuffle数据不落地(in mem)
* 流式任务执行而不是按stage级别执行（pipeline model 较之 火山优势）
* split为线程级别的调度（goroutine?）
* 数据源插件化 （connector）

**劣势：**
* 比如因为其流式pipeline执行方式的设计，**使其丧失了task级别的recovery机制**，所以Presto目前不是特别适合用来做**大规模的ETL查询**，当然目前社区也在通过对presto进行各种优化来使其适应更大规模的查询，比如Presto Ulimited和Presto on Spark项目。 


#### 使用场景
在B站，Presto主要承担了ADHOC查询、BI查询、DQC(数据校验，包括跨数据源校验)、AI ETL作业、数据探查等场景.

#### 集群规模

七个集群
两个机房
最大单集群节点 400+
总节点1000+

#### 业务增长

* 每日查询数16w左右，
* 每日查询HDFS数据量10PB左右，（目前相比2020年初日查询数增长10倍。 ）

####  Presto 架构
* 基于 PrestoSQL-330 二开和优化。
* 由k8s进行调度管理，包括jmx的采集、监控dashboard、告警 
* ![Pasted image 20230302062332.png](/img/user/Pasted%20image%2020230302062332.png)
* 所有 presto 查询都提交给 presto-gateway
#### gateway 改造
1. **（多 coordinator 幂等调度） 支持多coordinator调度，相同query只能调度到一个集群的一个coordinator。**
2. （coordinator探活） 探测coordinator的状态，如果不Active，则踢出调度列表，给无损发布提供可能。
3. **（多机房调度 流量控制）支持按用户/作业ID来选择机房调度，同时我们还会对Query通过Parser解析依赖的表和分区，根据哪个机房流量读取大，将Query调度到哪个集群。**
4. **（跨集群负载均衡）探测coordinator的负载，主要包括内存、作业是否堵住，支持跨集群负载均衡调度。**
5. （特征拦截）提取了Query特征，相同特征Query提交我们会有一系列拦截措施。 

####  Presto 改造

1. **（单点改造）[[coordinator 多活改造\|coordinator 多活改造]]，解决了coordinator的单点问题。**

2. （灵活调度，业务调度，时间调度）coordinator支持按业务来进行调度，不同业务调度到不同的Worker节点，同时为了增加集群利用率，我们支持按时间跨Label调度， 比如凌晨为adhoc和bi查询的低峰，但却为dqc的高峰，这个时候dqc能够跨Label使用其他Label的计算资源。 

## 稳定性改进

#### [[coordinator 多活改造\|coordinator 多活改造]]

#### Label 改造
* 资源隔离
改造思路：**全局资源共享调配**
1.开发一个服务，负责将已经划好label的配置文件load进内存，并实时检测文件是否更新，如有更新重新load。

2.DiscoveryNodeManager 通过服务发现拿到所有Node之后，将label信息写进InternalNode中。

3.NodeSelector 构建NodeMap的时候也就有了节点label信息了。

4.客户端根据不同业务将label信息传递到coordinator，调度的时候根据label去get到相应的节点即可 
![Pasted image 20230303004621.png](/img/user/Pasted%20image%2020230303004621.png)

#### 实时惩罚
#### 查询限制
#### 其他改造

## 可用性改造