## 简介

- percona xtradb cluster
    - MySQL集群方案
    - 包括：
        - Percona XtraDB Server
            
            [WSREP API](https://launchpad.net/wsrep)(write set replication patches) 
            
            通过Galera将不同MySQL实例连接起来，实现多主集群
            
    - 在集群上方搭建中间层，负责负载均衡、读写分离等——使得节点对客户端透明
        
        ![Untitled](../images/percona%20xtradb%20cluster%EF%BC%88PXC%EF%BC%89%206c33823235664af7baa9dc6cffd29e49/Untitled.png)
        
    - 集群由节点组成，每个节点包含相同数据集
    - 推荐的配置是至少有 3 个节点，2个节点也可以使用。各自为主，节点间关系对等。
    - 基本可达到实时同步，在存储引擎层实现的同步复制
        
        ![8888413-c63e7e6f05ff1945.png](../images/percona%20xtradb%20cluster%EF%BC%88PXC%EF%BC%89%206c33823235664af7baa9dc6cffd29e49/8888413-c63e7e6f05ff1945.png)
        
    - CAP
        - C：一致性
        - A：可用性
- 优点
    - 实现了MySQL集群的高可用性和数据的强一致性
    - 完成了真正的多节点读写的集群方案
    - 改善了主从复制延迟问题，基本上达到了实时同步
    - 新加入的节点可以自动部署，无需提交手动备份，维护方便
    - 没有中央管理，可以在任何时候移除任何节点，对集群的运行不会有影响
- 缺点
    - 配置新节点的开销。 添加新节点时，它必须从现有节点之一复制完整的数据集。
    - 木桶原理，任何更新的事务都需要全局验证通过才能在其他节点执行，集群性能受限于性能最差的节点
    - 多节点并发写入时锁冲突问题严重（实时基于存储引擎层来实现同步复制）
    - 写扩大问题，负载过大时不推荐使用
    - 高冗余
    - 只支持InnoDB存储引擎
- 限制
    - **复制仅适用于 InnoDB 存储引擎。**不会复制对其他类型表的任何写入。
    - **不支持显式表锁**：在多主环境下不支持LOCK/UNLOCK TABLES
    - 查询日志不能保存在表中。如果开启查询日志，只能保存到文件中。
        
        ```xml
        log_output = FILE
        ```
        
        使用 general_log 和 general_log_file 选择查询日志记录和日志文件名。
        
    - **最大允许事务大小由 wsrep_max_ws_rows （无限制）和 wsrep_max_ws_size （2GB）变量定义。**LOAD DATA INFILE 处理将每 10 000 行提交一次。因此，由于 LOAD DATA 而导致的大事务将被拆分为一系列小事务。
    - 正在COMMIT的事务可能回滚**。**由于集群级别的乐观并发控制，可以有两个事务写入相同的行并在不同的 Percona XtraDB Cluster 节点中提交，并且其中只有一个可以成功提交。失败的将被中止。
    - **不支持 XA 事务，提交时可能回滚**
        
        > However, for a distributed transaction, you must use the SERIALIZABLE isolation level to achieve ACID properties. It is enough to use REPEATABLE READ for a nondistributed transaction, but not for a distributed transaction。对于分布式事务，必须使用 SERIALIZABLE 隔离级别来实现 ACID 属性。 对非分布式事务使用 REPEATABLE READ 就足够了，但对分布式事务则不行。
        > 
    - **整个集群的写入吞吐量受到最弱节点的限制。**如果一个节点变慢，整个集群就会变慢。如果你对稳定的高性能有要求，那么应该有相应的硬件支持。
    - **建议的最小集群大小为 3 个节点。**第三个节点可以是仲裁者。
    - **在集群模式下运行 Percona XtraDB Cluster 时避免 ALTER TABLE ... IMPORT/EXPORT 工作负载。**
    - **所有表都必须有一个主键。**这确保了相同的行在不同的节点上以相同的顺序出现。没有主键的表不支持 DELETE 语句。SELECT LIMIT返回不同数据集

## 特征

- 高可用性
    
    关闭任何节点，PXC都能继续运行。（即使在节点崩溃或网络不可用的情况下）
    
    节点关闭时对数据进行了更改，则该节点再次加入集群时：
    
    - SST状态快照传输：复制所有数据
        - 当新节点加入集群并从现有节点接收所有数据时，通常使用 SST。
    - IST增量状态转移：仅复制增量数据
        - 需要满足
            - 新加入的节点状态UUID与集群中的节点一致
            - 新加入的节点缺失数据在捐助节点的写集缓存中存在
        - IST使用节点上的缓存机制实现的。 每个节点都包含一个缓存，最后 N 次更改的环形缓冲区（大小可配置），并且该节点能够传输此缓存的一部分。
        显然，只有当需要传输的更改量小于 N 时，才能进行 IST。如果超过 N，则加入节点必须执行 SST。
- PXC严格模式
    
    避免使用不支持和预发布功能，在启动和运行时执行验证。
    
    除非节点充当独立服务器或节点正在bootstrap，否则PXC严格模式默认设置为禁用
    
    - 模式：
        - **DISABLED：**不执行严格模式验证并正常运行
        - **PERMISSIVE：**如果验证失败，记录警告并继续正常运行。
        - **ENFORCING：（默认严格模式）**
            
            如果在启动期间验证失败，停止服务器并抛出错误。
            
            如果运行时验证，拒绝操作并抛出错误。
            
        - **MASTER：**与 ENFORCING 相同，只是不执行显式表锁定的验证。此模式可用于将写入操作隔离到单个节点的集群。
    - 验证，不同模式不同
        - 组复制（可能与PXC冲突）
        - 对使用存储引擎的表进行复制
        - 仅支持默认的基于行的二进制日志记录格式
        - 未定义主键的不良操作
        - 将MySQL数据库中的表作为日志输出目标
        - 显示表锁
        - 自增值交错

## 数据一致性

![Untitled](../images/percona%20xtradb%20cluster%EF%BC%88PXC%EF%BC%89%206c33823235664af7baa9dc6cffd29e49/Untitled%201.png)

### 基于验证的复制

- 当客户端发出commit命令时，在实际提交之前，更改都将被收集到一个写集中并发送到其他所有节点（写集中包含事务信息和更改行的主键）。用写集中的主键与当前节点中未完成事务的所有写集的主键相比较，确定节点是否可以提交事务。同时满足以下三个条件则验证失败（存在冲突）：
    - 两个事务来源于不同节点。
    - 两个事务包含相同的主键。
    - 老事务未提交完成。（GTID）
- 验证失败，节点删除写集，集群回滚原始事务。
- 每个节点单独进行验证。所有节点都以相同的顺序接收事务，事务结果相同。
- 节点之间不交换“是否冲突”的信息，各个节点独立异步处理事务。
- 最后，启动事务的节点可以通知客户端应用程序是否提交了事务。

- 仅当事务通过认证并准备好提交时，GTID 才会增加。事务必须以相同的顺序应用于所有实例。
- 仅有一个回滚线程。

## 架构

![1081074619-5aa493adedb1a.webp](../images/percona%20xtradb%20cluster%EF%BC%88PXC%EF%BC%89%206c33823235664af7baa9dc6cffd29e49/1081074619-5aa493adedb1a.webp)

**DBMS**：在单个节点上运行的数据库服务器。

**wsrep API：**数据库的通用复制插件接口，定义了一组应用程序回调和复制插件调用函数。wsrep api将数据库中的数据改变视为一种状态变化。

**Galera Replication Plugin：**实现写集复制功能的核心模块。

**Group Communication plugins**：Galera Clsuter集群的组通信系统。实现统一全局数据同步策略和集群内所有事务的排序，便于生成GTID。对于每一个节点有2方面工作：

- 完成数据同步。
- 完成与其他节点的通信。

**当存在更新时：**

- 客户端修改数据库内容，本节点的数据库中发生状态更改。（wsrep api通过GTID识别状态更改）
- wsrep hooks将更改转为写集。
- dlopen函数连接wsrep钩子与Galera复制插件。
- Galera复制插件处理写集验证，并将更改复制到集群中的其它节点。

## 流量控制

当一个节点到限制值，阻止所有节点写

- 通过确保事务复制到所有节点并根据集群范围的顺序执行来实现同步复制，事务apply和commit在通过集群复制时异步发生。
- 节点将写入集组织成全局排序，节点从集群接收但尚未apply和commit的事务保留在接收队列中。
- 当接收到的队列达到一定大小时，节点触发流控制，暂停复制，然后通过接收到的队列工作。当它将接收到的队列减少到某设置大小时，节点会恢复复制。

## 配置

安全模块

- 生产环境需要配置安全策略来允许对资源的访问
- SELinux
    
    ```bash
    setenforce 0
    ```
    

默认端口

3306：数据库对外服务的端口号

4444：请求SST，全量同步端口

> wsrep_sst_receive_address=10.11.12.205:5555
> 

4567: pxc cluster相互通信端口

> wsrep_provider_options ="gmcast.listen_addr=tcp://0.0.0.0:4010; “
> 

4568: 传输IST用的

> wsrep_provider_options = "ist.recv_addr=10.11.12.206:7777
>