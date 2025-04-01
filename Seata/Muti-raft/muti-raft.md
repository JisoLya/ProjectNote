
要点包括：
1. 不增加物理实例，而是在现有物理实例上创建多个Raft组
2. 在三个物理实例上，每个实例同时承担不同Raft组的Leader、Follower和Learner角色
3. 需要考虑节点新增、移除以及宕机的解决方案
4. 结合代码来说明实现方案

首先，我需要了解这个issue的背景和动机，然后提出方案的整体架构，接着详细说明代码层面的实现，最后讨论各种边缘情况的处理策略。</think># [FEATURE] 增强Seata集群稳定性与可扩展性：Multi-Raft模式设计与实现

## 问题描述

当前Seata的Raft模式采用单一Raft组实现，虽然保证了数据一致性，但存在以下问题：

1. **单Leader瓶颈**：所有事务请求都必须经过单一Leader节点处理，成为性能瓶颈
2. **水平扩展受限**：随着事务量增长，单一Raft组无法通过增加节点线性提升系统整体性能
3. **全局故障影响**：Leader选举期间，整个系统暂时不可用
4. **资源利用率低**：大多数Follower节点处于被动接收日志状态，资源未充分利用

## 方案概述

提议引入Multi-Raft架构，将事务数据按照一定规则分片，每个分片由独立的Raft组管理，在不增加物理实例的情况下创建多个虚拟Raft组。在N个物理节点上运行M个Raft组（M>N），使每个物理节点同时承担不同Raft组的Leader/Follower/Learner角色。

## 详细设计

### 1. 架构设计

![Multi-Raft架构示意图](https://i.imgur.com/6iw2qbE.png)

在三节点集群中创建多个Raft组的示例：
- 物理节点A：Group1-Leader, Group2-Follower, Group3-Follower
- 物理节点B：Group1-Follower, Group2-Leader, Group3-Follower
- 物理节点C：Group1-Follower, Group2-Follower, Group3-Leader

### 2. 关键组件设计

#### 2.1 分片策略

```java
public interface ShardingStrategy {
    /**
     * 根据XID确定Raft组
     * @param xid 事务ID
     * @return Raft组ID
     */
    String determineGroup(String xid);
    
    /**
     * 获取所有Raft组
     * @return 所有组ID
     */
    List<String> getAllGroups();
}

// 哈希分片策略实现
@LoadLevel(name = "consistent-hash")
public class ConsistentHashShardingStrategy implements ShardingStrategy {
    private final ConsistentHashRouter<String> router;
    private final List<String> allGroups;
    
    public ConsistentHashShardingStrategy(List<String> groups) {
        this.allGroups = groups;
        this.router = new ConsistentHashRouter<>(groups, 256);
    }
    
    @Override
    public String determineGroup(String xid) {
        return router.getNode(xid);
    }
    
    @Override
    public List<String> getAllGroups() {
        return allGroups;
    }
}
```

#### 2.2 多组Raft服务器管理

修改`RaftServerManager`以支持多组：

```java
public class MultiRaftServerManager {
    private static final Logger LOGGER = LoggerFactory.getLogger(MultiRaftServerManager.class);
    private static final Map<String, RaftServer> RAFT_SERVER_MAP = new ConcurrentHashMap<>();
    private static final AtomicBoolean INIT = new AtomicBoolean(false);
    private static volatile boolean RAFT_MODE;
    private static RpcServer rpcServer;
    
    public static void init(List<String> groups) {
        if (INIT.compareAndSet(false, true)) {
            RAFT_MODE = true;
            
            // 初始化配置
            Configuration config = ConfigurationFactory.getInstance();
            String dataDir = config.getConfig(ConfigurationKeys.STORE_FILE_DIR, 
                DEFAULT_RAFT_DATA_DIR) + separator + "raft";
            
            // 创建RPC服务器
            rpcServer = createAndInitRpcServer();
            
            // 为每个组创建RaftServer实例
            for (String group : groups) {
                String groupDataDir = dataDir + File.separator + group;
                PeerId serverId = getPeerIdFromConfig(group);
                NodeOptions nodeOptions = createNodeOptions(group);
                
                try {
                    RaftServer raftServer = new RaftServer(groupDataDir, group, serverId, nodeOptions, rpcServer);
                    RAFT_SERVER_MAP.put(group, raftServer);
                    LOGGER.info("Created RaftServer for group: {}", group);
                } catch (IOException e) {
                    LOGGER.error("Failed to create RaftServer for group: {}", group, e);
                    throw new RuntimeException("Failed to initialize Raft server for group: " + group, e);
                }
            }
        }
    }
    
    public static void start() {
        RAFT_SERVER_MAP.forEach((group, raftServer) -> {
            try {
                raftServer.start();
                LOGGER.info("Started RaftServer for group: {}", group);
            } catch (IOException e) {
                LOGGER.error("Failed to start RaftServer for group: {}", group, e);
                throw new RuntimeException(e);
            }
        });
        
        // 启动RPC服务器
        if (rpcServer != null) {
            rpcServer.registerProcessor(new PutNodeInfoRequestProcessor());
            SerializerManager.addSerializer(SerializerType.JACKSON.getCode(), new JacksonBoltSerializer());
            if (!rpcServer.init(null)) {
                throw new RuntimeException("Failed to start RPC server!");
            }
        }
    }
    
    public static boolean isLeader(String group) {
        AtomicReference<RaftStateMachine> stateMachine = new AtomicReference<>();
        Optional.ofNullable(RAFT_SERVER_MAP.get(group)).ifPresent(raftServer -> {
            stateMachine.set(raftServer.getRaftStateMachine());
        });
        RaftStateMachine raftStateMachine = stateMachine.get();
        return !isRaftMode() && RAFT_SERVER_MAP.isEmpty() || 
               (raftStateMachine != null && raftStateMachine.isLeader());
    }
    
    // ... 其他方法
}
```

#### 2.3 Session管理适配Multi-Raft

修改`SessionHolder`以支持多组：

```java
public class SessionHolder {
    private static final Map<String, SessionManager> SESSION_MANAGER_MAP = new ConcurrentHashMap<>();
    
    public static void init(SessionMode sessionMode) {
        if (SessionMode.RAFT.equals(sessionMode)) {
            // 加载分片策略
            ShardingStrategy shardingStrategy = EnhancedServiceLoader.load(ShardingStrategy.class);
            List<String> groups = shardingStrategy.getAllGroups();
            
            // 为每个组创建会话管理器
            for (String group : groups) {
                SessionManager sessionManager = EnhancedServiceLoader.load(
                    SessionManager.class, 
                    SessionMode.RAFT.getName(),
                    new Object[]{ROOT_SESSION_MANAGER_NAME + "-" + group}
                );
                SESSION_MANAGER_MAP.put(group, sessionManager);
            }
            
            // 初始化并启动Multi-Raft服务器
            MultiRaftServerManager.init(groups);
            MultiRaftServerManager.start();
        } else {
            // 现有代码保持不变
        }
    }
    
    public static SessionManager getRootSessionManager(String group) {
        return SESSION_MANAGER_MAP.get(group);
    }
}
```

#### 2.4 上下文管理

创建`MultiRaftContext`类管理多组上下文：

```java
public class MultiRaftContext {
    private static final ThreadLocal<String> CURRENT_GROUP = new ThreadLocal<>();
    private static ShardingStrategy shardingStrategy;
    
    public static void setShardingStrategy(ShardingStrategy strategy) {
        shardingStrategy = strategy;
    }
    
    public static String bindGroupByXid(String xid) {
        String group = shardingStrategy.determineGroup(xid);
        CURRENT_GROUP.set(group);
        return group;
    }
    
    public static String getGroup() {
        return CURRENT_GROUP.get();
    }
    
    public static void unbindGroup() {
        CURRENT_GROUP.remove();
    }
}
```

#### 2.5 协调器改进

修改`RaftCoordinator`以支持多组路由：

```java
public class MultiRaftCoordinator extends DefaultCoordinator implements ApplicationListener<ClusterChangeEvent> {
    private static final Map<String, Boolean> GROUP_LEADERS = new ConcurrentHashMap<>();
    
    public MultiRaftCoordinator(RemotingServer remotingServer) {
        super(remotingServer);
    }
    
    @Override
    public <T extends AbstractTransactionRequest, S extends AbstractTransactionResponse> 
    void exceptionHandleTemplate(Callback<T, S> callback, T request, S response) {
        // 提取XID并确定组
        String xid = extractXidFromRequest(request);
        String group = MultiRaftContext.bindGroupByXid(xid);
        
        try {
            if (!isLeader(group)) {
                throw new TransactionException(TransactionExceptionCode.NotRaftLeader,
                    "Current TC is not the leader for group " + group);
            }
            super.exceptionHandleTemplate(callback, request, response);
        } catch (TransactionException tex) {
            LOGGER.error("Transaction exception while processing request: {}", request, tex);
            callback.onTransactionException(request, response, tex);
        } finally {
            MultiRaftContext.unbindGroup();
        }
    }
    
    private boolean isLeader(String group) {
        return GROUP_LEADERS.getOrDefault(group, Boolean.FALSE);
    }
    
    @Override
    public void onApplicationEvent(ClusterChangeEvent event) {
        GROUP_LEADERS.put(event.getGroup(), event.isLeader());
    }
    
    private String extractXidFromRequest(AbstractTransactionRequest request) {
        if (request instanceof GlobalBeginRequest) {
            // 新事务，使用应用ID作为分片键
            String applicationId = RootContext.getApplicationId();
            return applicationId != null ? applicationId : "default";
        } else if (request instanceof GlobalCommitRequest) {
            return ((GlobalCommitRequest) request).getXid();
        } else if (request instanceof GlobalRollbackRequest) {
            return ((GlobalRollbackRequest) request).getXid();
        } else if (request instanceof BranchRegisterRequest) {
            return ((BranchRegisterRequest) request).getXid();
        } else if (request instanceof BranchReportRequest) {
            return ((BranchReportRequest) request).getXid();
        }
        // 默认返回空，将在后续逻辑中处理
        return "";
    }
}
```

### 3. 配置设计

```yaml
seata:
  raft:
    # 启用Multi-Raft模式
    multi-raft-enabled: true
    
    # 分片策略
    sharding:
      strategy: consistent-hash
      group-count: 3
    
    # 通用Raft配置
    snapshot-interval: 600
    apply-batch: 32
    max-append-buffer-size: 262144
    max-replicator-inflight-msgs: 256
    election-timeout-ms: 2000
    serialization: jackson
    
    # 组特定配置
    groups:
      group-1:
        data-dir: ${user.home}/seata/data/raft/group-1
        server-addr: 192.168.0.1:7091,192.168.0.2:7091,192.168.0.3:7091
      
      group-2:
        data-dir: ${user.home}/seata/data/raft/group-2
        server-addr: 192.168.0.1:7092,192.168.0.2:7092,192.168.0.3:7092
      
      group-3:
        data-dir: ${user.home}/seata/data/raft/group-3
        server-addr: 192.168.0.1:7093,192.168.0.2:7093,192.168.0.3:7093
```

### 4. 异常场景处理

#### 4.1 节点宕机处理

当某个物理节点宕机时，该节点上的所有Raft角色都会失效。Multi-Raft架构的优势在于，只有该节点作为Leader的组需要重新选举，其他组不受影响继续提供服务。

```java
// 节点宕机检测和处理
public class NodeFailureHandler {
    private static final ScheduledExecutorService SCHEDULER = 
        new ScheduledThreadPoolExecutor(1, new NamedThreadFactory("node-failure-handler", true));
    
    public static void scheduleNodeHealthCheck() {
        SCHEDULER.scheduleWithFixedDelay(() -> {
            ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
            for (String group : strategy.getAllGroups()) {
                try {
                    RaftServer raftServer = MultiRaftServerManager.getRaftServer(group);
                    if (raftServer != null) {
                        checkGroupHealth(group, raftServer);
                    }
                } catch (Exception e) {
                    LOGGER.error("Failed to check health for group: {}", group, e);
                }
            }
        }, 30, 30, TimeUnit.SECONDS);
    }
    
    private static void checkGroupHealth(String group, RaftServer raftServer) {
        // 获取组成员配置
        Configuration conf = RouteTable.getInstance().getConfiguration(group);
        if (conf == null) {
            LOGGER.warn("Cannot get configuration for group: {}", group);
            return;
        }
        
        // 检查各节点状态
        List<PeerId> peers = conf.getPeers();
        for (PeerId peer : peers) {
            boolean isAlive = checkPeerAlive(peer);
            if (!isAlive) {
                LOGGER.warn("Peer {} in group {} is not responding", peer, group);
                
                // 如果是本节点为该组的Leader，且不可达，可能需要触发重新选举
                if (raftServer.getRaftStateMachine().isLeader() && 
                    peer.equals(raftServer.getServerId())) {
                    LOGGER.error("Current node is leader for group {} but unstable", group);
                    // 可以考虑主动降级为Follower
                    // raftServer.getNode().stepDown(...);
                }
            }
        }
    }
    
    private static boolean checkPeerAlive(PeerId peer) {
        // 实现节点健康检查逻辑
        // 可以通过JRaft的RPC机制或其他方式检查节点可达性
        return true; // 示例返回
    }
}
```

#### 4.2 新增节点处理

当需要向集群添加新节点时，需要为每个Raft组配置新节点角色：

```java
public class NodeScaleHandler {
    
    /**
     * 向集群添加新节点
     * @param newNodeId 新节点ID (ip:port)
     */
    public static void addNode(String newNodeId) {
        ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        
        for (String group : strategy.getAllGroups()) {
            try {
                // 为该组添加节点
                addNodeToGroup(group, newNodeId);
            } catch (Exception e) {
                LOGGER.error("Failed to add node {} to group {}", newNodeId, group, e);
            }
        }
    }
    
    private static void addNodeToGroup(String group, String nodeId) throws Exception {
        // 从配置获取组的Leader地址
        RaftServer raftServer = MultiRaftServerManager.getRaftServer(group);
        if (raftServer == null || !raftServer.getRaftStateMachine().isLeader()) {
            // 本节点不是该组的Leader，找到Leader节点
            RouteTable.getInstance().refreshLeader(
                MultiRaftServerManager.getCliClientServiceInstance(), 
                group, 
                3000);
            PeerId leaderId = RouteTable.getInstance().selectLeader(group);
            
            if (leaderId == null) {
                throw new Exception("Cannot determine leader for group " + group);
            }
            
            // 通过Leader添加节点
            PeerId newPeer = new PeerId(nodeId);
            CliService cliService = MultiRaftServerManager.getCliServiceInstance();
            cliService.addPeer(group, leaderId, newPeer);
            LOGGER.info("Added node {} to group {} through leader {}", nodeId, group, leaderId);
        } else {
            // 本节点是Leader，直接添加
            PeerId newPeer = new PeerId(nodeId);
            raftServer.getNode().addPeer(newPeer, status -> {
                if (status.isOk()) {
                    LOGGER.info("Successfully added node {} to group {}", nodeId, group);
                } else {
                    LOGGER.error("Failed to add node {} to group {}: {}", 
                        nodeId, group, status.getErrorMsg());
                }
            });
        }
    }
}
```

#### 4.3 节点移除处理

从集群移除节点的处理方式：

```java
public class NodeScaleHandler {
    
    /**
     * 从集群移除节点
     * @param nodeId 要移除的节点ID
     */
    public static void removeNode(String nodeId) {
        ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        
        for (String group : strategy.getAllGroups()) {
            try {
                // 检查该节点在组中的角色
                checkNodeRoleBeforeRemove(group, nodeId);
                
                // 从该组移除节点
                removeNodeFromGroup(group, nodeId);
            } catch (Exception e) {
                LOGGER.error("Failed to remove node {} from group {}", nodeId, group, e);
            }
        }
    }
    
    private static void checkNodeRoleBeforeRemove(String group, String nodeId) throws Exception {
        // 获取组配置
        Configuration conf = RouteTable.getInstance().getConfiguration(group);
        if (conf == null) {
            throw new Exception("Cannot get configuration for group " + group);
        }
        
        // 检查节点是否为Leader
        PeerId peerId = new PeerId(nodeId);
        PeerId leaderId = RouteTable.getInstance().selectLeader(group);
        
        if (peerId.equals(leaderId)) {
            LOGGER.warn("Node {} is the leader of group {}, consider transferring leadership first", 
                nodeId, group);
            
            // 可以尝试将Leader角色转移给其他节点
            transferLeadership(group, nodeId);
        }
    }
    
    private static void transferLeadership(String group, String currentLeaderId) throws Exception {
        // 获取组配置中的其他节点
        Configuration conf = RouteTable.getInstance().getConfiguration(group);
        List<PeerId> peers = conf.getPeers();
        
        // 找到一个非当前Leader的节点
        PeerId currentLeader = new PeerId(currentLeaderId);
        for (PeerId peer : peers) {
            if (!peer.equals(currentLeader)) {
                // 转移Leadership
                CliService cliService = MultiRaftServerManager.getCliServiceInstance();
                cliService.transferLeader(group, currentLeader, peer);
                LOGGER.info("Transferred leadership of group {} from {} to {}", 
                    group, currentLeaderId, peer);
                
                // 等待新Leader选举完成
                Thread.sleep(2000);
                break;
            }
        }
    }
    
    private static void removeNodeFromGroup(String group, String nodeId) throws Exception {
        RaftServer raftServer = MultiRaftServerManager.getRaftServer(group);
        if (raftServer == null || !raftServer.getRaftStateMachine().isLeader()) {
            // 本节点不是该组的Leader，找到Leader节点
            RouteTable.getInstance().refreshLeader(
                MultiRaftServerManager.getCliClientServiceInstance(), 
                group, 
                3000);
            PeerId leaderId = RouteTable.getInstance().selectLeader(group);
            
            if (leaderId == null) {
                throw new Exception("Cannot determine leader for group " + group);
            }
            
            // 通过Leader移除节点
            PeerId peerToRemove = new PeerId(nodeId);
            CliService cliService = MultiRaftServerManager.getCliServiceInstance();
            cliService.removePeer(group, leaderId, peerToRemove);
            LOGGER.info("Removed node {} from group {} through leader {}", 
                nodeId, group, leaderId);
        } else {
            // 本节点是Leader，直接移除
            PeerId peerToRemove = new PeerId(nodeId);
            raftServer.getNode().removePeer(peerToRemove, status -> {
                if (status.isOk()) {
                    LOGGER.info("Successfully removed node {} from group {}", 
                        nodeId, group);
                } else {
                    LOGGER.error("Failed to remove node {} from group {}: {}", 
                        nodeId, group, status.getErrorMsg());
                }
            });
        }
    }
}
```

### 5. 数据迁移和重新平衡

当需要调整分片数量或重新平衡数据时：

```java
public class DataRebalancer {
    
    /**
     * 重新平衡Raft组数据
     * @param oldGroups 旧组列表
     * @param newGroups 新组列表
     */
    public static void rebalance(List<String> oldGroups, List<String> newGroups) {
        // 1. 计算需要迁移的数据
        Map<String, String> migrationMap = calculateMigrationPlan(oldGroups, newGroups);
        
        // 2. 执行数据迁移
        for (Map.Entry<String, String> entry : migrationMap.entrySet()) {
            String sourceGroup = entry.getKey();
            String targetGroup = entry.getValue();
            migrateData(sourceGroup, targetGroup);
        }
    }
    
    private static Map<String, String> calculateMigrationPlan(List<String> oldGroups, List<String> newGroups) {
        // 计算数据迁移方案
        // 示例简化实现
        Map<String, String> migrationMap = new HashMap<>();
        
        // 如果只是增加组，则需要从现有组迁移部分数据到新组
        List<String> addedGroups = new ArrayList<>(newGroups);
        addedGroups.removeAll(oldGroups);
        
        if (!addedGroups.isEmpty() && !oldGroups.isEmpty()) {
            // 简单地将每个旧组的一部分数据迁移到新组
            int oldGroupSize = oldGroups.size();
            for (int i = 0; i < addedGroups.size(); i++) {
                String sourceGroup = oldGroups.get(i % oldGroupSize);
                String targetGroup = addedGroups.get(i);
                migrationMap.put(sourceGroup, targetGroup);
            }
        }
        
        return migrationMap;
    }
    
    private static void migrateData(String sourceGroup, String targetGroup) {
        LOGGER.info("Migrating data from group {} to group {}", sourceGroup, targetGroup);
        
        // 1. 获取源组会话管理器
        SessionManager sourceManager = SessionHolder.getRootSessionManager(sourceGroup);
        if (sourceManager == null) {
            LOGGER.error("Source session manager not found for group {}", sourceGroup);
            return;
        }
        
        // 2. 获取目标组会话管理器
        SessionManager targetManager = SessionHolder.getRootSessionManager(targetGroup);
        if (targetManager == null) {
            LOGGER.error("Target session manager not found for group {}", targetGroup);
            return;
        }
        
        // 3. 获取需要迁移的全局会话
        List<GlobalSession> sessionsToMigrate = selectSessionsToMigrate(sourceManager, targetGroup);
        
        // 4. 迁移会话
        for (GlobalSession session : sessionsToMigrate) {
            try {
                // 暂停源组对该会话的处理
                sourceManager.removeGlobalSession(session);
                
                // 在目标组添加会话
                targetManager.addGlobalSession(session);
                
                LOGGER.info("Migrated session {} from group {} to group {}", 
                    session.getXid(), sourceGroup, targetGroup);
            } catch (Exception e) {
                LOGGER.error("Failed to migrate session {}", session.getXid(), e);
                // 处理迁移失败的情况
                try {
                    // 尝试恢复到源组
                    sourceManager.addGlobalSession(session);
                } catch (Exception ex) {
                    LOGGER.error("Failed to restore session {} to source group", 
                        session.getXid(), ex);
                }
            }
        }
    }
    
    private static List<GlobalSession> selectSessionsToMigrate(
            SessionManager sourceManager, String targetGroup) {
        // 根据新的分片策略选择应该迁移到目标组的会话
        List<GlobalSession> result = new ArrayList<>();
        ShardingStrategy newStrategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        
        // 获取所有活跃会话
        Collection<GlobalSession> allSessions = sourceManager.allSessions();
        
        // 根据新策略选择属于目标组的会话
        for (GlobalSession session : allSessions) {
            String xid = session.getXid();
            String mappedGroup = newStrategy.determineGroup(xid);
            
            if (targetGroup.equals(mappedGroup)) {
                result.add(session);
            }
        }
        
        return result;
    }
}
```

## 实现收益

1. **性能提升**：不同Raft组的Leader分布在不同节点上，可以并行处理事务，提高整体吞吐量
2. **资源利用率提高**：同一物理节点可以同时扮演不同角色，充分利用CPU和内存资源
3. **可用性提升**：单个节点故障只影响其作为Leader的Raft组，其他组仍能正常提供服务
4. **可伸缩性增强**：可以根据实际需求动态调整Raft组数量，实现更精细的水平扩展

## 实施计划

1. 阶段一：核心框架实现
   - 实现Multi-Raft服务器管理
   - 实现分片策略接口和默认实现
   - 改造Session管理以支持多组

2. 阶段二：请求路由和处理
   - 实现上下文管理
   - 改造协调器支持多组路由
   - 增强异常处理机制

3. 阶段三：高级特性
   - 实现节点故障自动处理
   - 实现动态节点扩展
   - 实现数据迁移和重平衡

## 总结

Multi-Raft模式能够在不增加物理实例的情况下，显著提升Seata集群的性能、可用性和伸缩性。通过在现有物理节点上创建多个虚拟Raft组，每个节点同时承担不同组的不同角色，实现了资源的更高效利用，并降低了单点故障的影响范围。

这一改进将使Seata更好地应对高并发事务处理场景，为企业级用户提供更稳定、可靠的分布式事务服务。


<think>在Seata中，事务阶段1通常指的是全局事务的准备阶段（AT模式中的全局事务开始和注册阶段，分支事务的注册）。当这个阶段的节点宕机，我需要详细分析不同角色节点宕机的影响和处理策略。

我需要考虑以下几种情况：
1. TC节点（Raft Leader）宕机 - 正在处理事务注册请求的Leader宕机
2. TC节点（Raft Follower）宕机 - 参与日志复制的Follower宕机
3. TM客户端节点宕机 - 事务发起方宕机
4. RM客户端节点宕机 - 资源管理器宕机

针对Multi-Raft模式，还需要考虑对应Raft组的影响和处理方式。</think># Seata Multi-Raft模式下事务阶段1节点宕机的处理策略

在Seata的事务处理流程中，阶段1通常指事务的准备阶段，包括全局事务的开始、分支事务的注册等操作。下面详细分析Multi-Raft模式下，阶段1中不同角色节点宕机的影响和处理策略。

## 1. TC Leader节点宕机

假设在事务阶段1，负责处理该事务的Raft组Leader节点宕机：

### 1.1 影响分析

1. **事务注册中断**：正在进行的全局事务开始或分支事务注册请求将失败
2. **日志状态不确定**：
   - 如果事务已提交到Raft日志并成功复制到多数节点，则该事务最终会被应用
   - 如果事务尚未成功复制到多数节点，则该事务会被丢弃

### 1.2 处理策略

```java
public class Phase1FailureHandler {
    
    /**
     * 处理Leader宕机的全局事务准备阶段故障
     */
    public static void handleLeaderFailureInPhase1(String group) {
        LOGGER.info("Handling leader failure in Phase 1 for group {}", group);
        
        // 1. 等待新Leader选举完成
        waitForNewLeaderElection(group);
        
        // 2. 获取新Leader
        PeerId newLeader = RouteTable.getInstance().selectLeader(group);
        if (newLeader == null) {
            LOGGER.error("Failed to elect new leader for group {}", group);
            return;
        }
        
        LOGGER.info("New leader elected for group {}: {}", group, newLeader);
        
        // 3. 处理进行中的事务
        SessionManager sessionManager = SessionHolder.getRootSessionManager(group);
        if (sessionManager == null) {
            return;
        }
        
        // 4. 找出处于Begin状态且没有分支的全局事务
        Collection<GlobalSession> sessions = sessionManager.allSessions();
        List<GlobalSession> orphanBeginSessions = sessions.stream()
            .filter(session -> session.getStatus() == GlobalStatus.Begin &&
                    session.getBranchSessions().isEmpty() &&
                    System.currentTimeMillis() - session.getBeginTime() < 
                        ConfigurationFactory.getInstance().getInt(ConfigurationKeys.GLOBAL_TRANSACTION_TIMEOUT, 60000))
            .collect(Collectors.toList());
        
        // 5. 对于孤立的Begin事务，设置它们为超时回滚状态
        for (GlobalSession session : orphanBeginSessions) {
            try {
                LOGGER.info("Handling orphan BEGIN transaction in Phase 1: {}", session.getXid());
                
                // 对于超过一定时间的事务，设置为超时回滚
                if (System.currentTimeMillis() - session.getBeginTime() > 
                        ConfigurationFactory.getInstance().getInt(ConfigurationKeys.HANDLE_PHASE1_TIMEOUT_THRESHOLD, 10000)) {
                    session.changeGlobalStatus(GlobalStatus.TimeoutRollbacking);
                    SessionHelper.endRollbackSession(session);
                    LOGGER.info("Changed orphan transaction to TimeoutRollbacking: {}", session.getXid());
                }
            } catch (Exception e) {
                LOGGER.error("Error handling orphan transaction {}", session.getXid(), e);
            }
        }
    }
    
    /**
     * 等待新Leader选举完成
     */
    private static void waitForNewLeaderElection(String group) {
        int retries = 10;
        while (retries > 0) {
            try {
                RouteTable.getInstance().refreshLeader(
                    MultiRaftServerManager.getCliClientServiceInstance(), 
                    group, 
                    3000);
                PeerId leaderId = RouteTable.getInstance().selectLeader(group);
                
                if (leaderId != null) {
                    return;
                }
            } catch (Exception e) {
                LOGGER.warn("Waiting for leader election in group {}, retries left: {}", 
                    group, retries);
            }
            
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            retries--;
        }
    }
}
```

### 1.3 客户端重试机制

对于TM和RM客户端，需要增强重试机制以处理TC Leader宕机：

```java
public class TMClientRetryStrategy {
    
    /**
     * 全局事务开始请求重试处理
     */
    public boolean retryBeginTransaction(String applicationId, String transactionServiceGroup, 
                                         String name, int timeout) {
        int retryCount = ConfigurationFactory.getInstance()
            .getInt(ConfigurationKeys.CLIENT_TM_BEGIN_RETRY_COUNT, 3);
        long retryInterval = ConfigurationFactory.getInstance()
            .getLong(ConfigurationKeys.CLIENT_TM_BEGIN_RETRY_INTERVAL, 1000);
        
        GlobalBeginRequest request = new GlobalBeginRequest();
        request.setApplicationId(applicationId);
        request.setTransactionServiceGroup(transactionServiceGroup);
        request.setTransactionName(name);
        request.setTimeout(timeout);
        
        GlobalBeginResponse response = null;
        for (int i = 0; i < retryCount; i++) {
            try {
                response = (GlobalBeginResponse) tmRpcClient.sendMsgWithResponse(request);
                if (response.getResultCode() == ResultCode.Success) {
                    return true;
                }
                
                // 处理NotLeader错误
                if (response.getResultCode() == ResultCode.Failed && 
                    response.getTransactionExceptionCode() == TransactionExceptionCode.NotRaftLeader) {
                    
                    LOGGER.warn("TC server is not leader, will retry after {} ms", retryInterval);
                    // 刷新客户端的Leader缓存
                    tmRpcClient.refreshLeaderInfo();
                    
                    Thread.sleep(retryInterval);
                    continue;
                }
                
                // 其他错误直接返回失败
                return false;
            } catch (TimeoutException e) {
                LOGGER.warn("Request timeout, may due to server leader changing, will retry", e);
                try {
                    Thread.sleep(retryInterval);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            } catch (Exception e) {
                LOGGER.error("Begin transaction error", e);
                break;
            }
        }
        
        return false;
    }
}
```

## 2. RM客户端节点在分支事务注册时宕机

当资源管理器(RM)在分支事务注册过程中宕机：

### 2.1 影响分析

1. **分支事务状态不确定**：
   - 如果宕机发生在发送请求前，分支事务未被TC感知
   - 如果宕机发生在TC确认后但RM未收到响应，TC可能已注册了分支事务

2. **全局事务悬挂风险**：全局事务可能缺少重要的分支事务，无法正确完成

### 2.2 TC端处理策略

```java
public class RMFailureInPhase1Handler {
    
    /**
     * 定期检查并处理可能的RM注册失败
     */
    public static void startOrphanBranchChecker() {
        ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(1, 
            new NamedThreadFactory("rm-failure-checker"));
        
        long checkInterval = ConfigurationFactory.getInstance()
            .getLong(ConfigurationKeys.RM_FAILURE_CHECK_INTERVAL, 60000);
        
        scheduler.scheduleWithFixedDelay(() -> {
            try {
                checkAndHandleOrphanBranches();
            } catch (Exception e) {
                LOGGER.error("Error checking orphan branches", e);
            }
        }, checkInterval, checkInterval, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 检查并处理孤立的全局事务
     */
    private static void checkAndHandleOrphanBranches() {
        ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        List<String> groups = strategy.getAllGroups();
        
        for (String group : groups) {
            SessionManager sessionManager = SessionHolder.getRootSessionManager(group);
            if (sessionManager == null) {
                continue;
            }
            
            Collection<GlobalSession> sessions = sessionManager.allSessions();
            for (GlobalSession session : sessions) {
                // 关注长时间处于Begin状态的事务
                if (session.getStatus() == GlobalStatus.Begin) {
                    long beginTime = session.getBeginTime();
                    long now = System.currentTimeMillis();
                    
                    // 如果事务开始时间超过设定阈值，但没有任何分支或者分支长时间未更新
                    if (now - beginTime > ConfigurationFactory.getInstance()
                            .getLong(ConfigurationKeys.MAX_BRANCH_REGISTER_TIMEOUT, 30000)) {
                        
                        // 检查是否有活跃的RM心跳
                        boolean hasActiveRMs = hasActiveResourceManagers(session);
                        
                        // 如果没有活跃的RM且事务已超时，则回滚
                        if (!hasActiveRMs) {
                            LOGGER.info("Rolling back orphan global transaction with no active RMs: {}", 
                                session.getXid());
                            try {
                                sessionManager.globalStatus(session.getXid(), GlobalStatus.TimeoutRollbacking);
                                // 触发异步回滚流程
                                AsyncWorker.getInstance().addTimeoutRollbackTask(session);
                            } catch (Exception e) {
                                LOGGER.error("Failed to rollback orphan transaction {}", session.getXid(), e);
                            }
                        }
                    }
                }
            }
        }
    }
    
    /**
     * 检查是否有活跃的资源管理器
     */
    private static boolean hasActiveResourceManagers(GlobalSession session) {
        // 获取参与该事务的资源管理器
        Set<String> resourceIds = new HashSet<>();
        session.getBranchSessions().forEach(branchSession -> 
            resourceIds.add(branchSession.getResourceId()));
        
        // 检查这些资源管理器是否有活跃心跳
        for (String resourceId : resourceIds) {
            if (!RMHeartbeatManager.getInstance().isResourceActive(resourceId)) {
                return false;
            }
        }
        
        return !resourceIds.isEmpty();
    }
}
```

### 2.3 RM端恢复策略

当RM恢复后，需要重新获取和处理未完成的事务：

```java
public class RMRecoveryHandler {
    
    /**
     * RM恢复后处理未完成的事务
     */
    public static void handleRMRecovery() {
        LOGGER.info("Handling RM recovery after crash in Phase 1");
        
        // 1. 从本地存储获取进行中的事务XID
        Set<String> localXids = UndoLogManager.findInProgressTransactions();
        
        // 2. 对每个本地事务查询其全局状态
        for (String xid : localXids) {
            try {
                GlobalStatus globalStatus = rmRpcClient.queryGlobalStatus(xid);
                
                // 3. 根据全局状态决定如何处理本地事务
                switch (globalStatus) {
                    case Begin:
                        // 全局事务仍在进行中，重新注册分支
                        tryReregisterBranch(xid);
                        break;
                        
                    case Committed:
                        // 全局事务已提交，提交本地事务
                        UndoLogManager.commitLocalTransaction(xid);
                        break;
                        
                    case Rollbacked:
                    case RollbackRetrying:
                    case TimeoutRollbacking:
                    case TimeoutRollbackRetrying:
                        // 全局事务正在回滚，回滚本地事务
                        UndoLogManager.rollbackLocalTransaction(xid);
                        break;
                        
                    case RollbackFailed:
                        // 全局事务回滚失败，记录异常并尝试回滚
                        LOGGER.error("Global transaction rollback failed: {}", xid);
                        UndoLogManager.rollbackLocalTransaction(xid);
                        break;
                        
                    case CommitFailed:
                        // 全局事务提交失败，记录异常
                        LOGGER.error("Global transaction commit failed: {}", xid);
                        break;
                        
                    default:
                        LOGGER.warn("Unknown global status {} for xid {}", globalStatus, xid);
                }
            } catch (Exception e) {
                LOGGER.error("Failed to handle recovery for xid {}", xid, e);
            }
        }
    }
    
    /**
     * 尝试重新注册分支事务
     */
    private static void tryReregisterBranch(String xid) {
        try {
            // 获取本地事务信息
            BranchTransactionInfo branchInfo = UndoLogManager.getBranchTransactionInfo(xid);
            if (branchInfo == null) {
                LOGGER.warn("Cannot find branch info for xid {}", xid);
                return;
            }
            
            // 重新注册分支
            Long branchId = rmRpcClient.branchRegister(
                BranchType.AT,
                branchInfo.getResourceId(),
                branchInfo.getClientId(),
                xid,
                branchInfo.getApplicationData(),
                branchInfo.getLockKeys()
            );
            
            if (branchId != null) {
                // 更新本地的分支ID
                UndoLogManager.updateBranchId(xid, branchId);
                LOGGER.info("Successfully re-registered branch for xid {}, new branchId: {}", 
                    xid, branchId);
            } else {
                LOGGER.warn("Failed to re-register branch for xid {}", xid);
            }
        } catch (Exception e) {
            LOGGER.error("Error re-registering branch for xid {}", xid, e);
        }
    }
}
```

## 3. TM客户端在开启全局事务后宕机

当事务管理器(TM)在开启全局事务后宕机：

### 3.1 影响分析

1. **全局事务悬挂**：TM未能继续提交或回滚全局事务，导致TC中事务一直处于Begin状态
2. **资源锁定**：相关资源可能被锁定，无法被其他事务使用

### 3.2 TC端超时处理

```java
public class TMFailureInPhase1Handler {
    
    /**
     * 启动TM失败监测器
     */
    public static void startTMFailureDetector() {
        ScheduledExecutorService scheduler = new ScheduledThreadPoolExecutor(1, 
            new NamedThreadFactory("tm-failure-detector"));
        
        long detectionInterval = ConfigurationFactory.getInstance()
            .getLong(ConfigurationKeys.TM_FAILURE_DETECTION_INTERVAL, 60000);
        
        scheduler.scheduleWithFixedDelay(() -> {
            try {
                detectAndHandleTMFailures();
            } catch (Exception e) {
                LOGGER.error("Error detecting TM failures", e);
            }
        }, detectionInterval, detectionInterval, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 检测和处理TM失败
     */
    private static void detectAndHandleTMFailures() {
        ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        List<String> groups = strategy.getAllGroups();
        
        long timeout = ConfigurationFactory.getInstance()
            .getLong(ConfigurationKeys.GLOBAL_TRANSACTION_TIMEOUT, 60000);
        
        for (String group : groups) {
            SessionManager sessionManager = SessionHolder.getRootSessionManager(group);
            if (sessionManager == null) {
                continue;
            }
            
            Collection<GlobalSession> sessions = sessionManager.allSessions();
            
            for (GlobalSession session : sessions) {
                if (session.getStatus() == GlobalStatus.Begin) {
                    String applicationId = session.getApplicationId();
                    
                    // 检查TM是否仍然活跃
                    boolean isTMActive = TMHeartbeatManager.getInstance().isApplicationActive(applicationId);
                    
                    // 检查事务是否超时
                    boolean isTransactionTimeout = System.currentTimeMillis() - session.getBeginTime() > timeout;
                    
                    // 如果TM不活跃且事务已超时，执行超时回滚
                    if (!isTMActive && isTransactionTimeout) {
                        LOGGER.info("Rolling back transaction due to TM failure: {}, applicationId: {}", 
                            session.getXid(), applicationId);
                        
                        try {
                            SessionHelper.endRollbackSession(session);
                        } catch (Exception e) {
                            LOGGER.error("Failed to rollback transaction after TM failure: {}", 
                                session.getXid(), e);
                        }
                    }
                }
            }
        }
    }
}
```

### 3.3 TM端恢复策略

当TM客户端恢复后，需要处理之前未完成的事务：

```java
public class TMRecoveryAfterCrash {
    
    /**
     * TM启动时恢复处理
     */
    public static void recoverAfterStart() {
        LOGGER.info("Recovering TM after crash in Phase 1");
        
        // 1. 从本地存储获取未完成的事务列表
        Set<String> unfinishedXids = getUnfinishedTransactions();
        
        if (unfinishedXids.isEmpty()) {
            LOGGER.info("No unfinished transactions found");
            return;
        }
        
        LOGGER.info("Found {} unfinished transactions", unfinishedXids.size());
        
        // 2. 对每个未完成的事务查询全局状态
        for (String xid : unfinishedXids) {
            try {
                GlobalStatus globalStatus = tmRpcClient.queryGlobalStatus(xid);
                
                // 3. 根据全局状态决定操作
                switch (globalStatus) {
                    case Begin:
                        // 事务仍在进行中，发起全局回滚
                        LOGGER.info("Rolling back unfinished transaction: {}", xid);
                        tmRpcClient.globalRollback(xid);
                        break;
                        
                    case Committed:
                    case Rollbacked:
                    case RollbackRetrying:
                    case CommitRetrying:
                    case TimeoutRollbacking:
                    case TimeoutRollbackRetrying:
                    case Finished:
                        // 事务已经在处理中或已完成，不需要额外操作
                        LOGGER.info("Transaction {} is already in status: {}", xid, globalStatus);
                        break;
                        
                    default:
                        LOGGER.warn("Unexpected status for transaction {}: {}", xid, globalStatus);
                }
            } catch (Exception e) {
                LOGGER.error("Failed to recover transaction {}", xid, e);
            } finally {
                // 4. 从本地记录中移除
                removeLocalTransactionRecord(xid);
            }
        }
    }
    
    /**
     * 获取本地未完成的事务
     */
    private static Set<String> getUnfinishedTransactions() {
        // 实现根据具体存储方式获取未完成事务
        // 可以是文件、数据库或其他持久化存储
        return new HashSet<>();
    }
    
    /**
     * 移除本地事务记录
     */
    private static void removeLocalTransactionRecord(String xid) {
        // 实现移除本地事务记录的逻辑
    }
}
```

## 4. Follower节点宕机

当参与同步的Follower节点在阶段1宕机：

### 4.1 影响分析

1. 在Multi-Raft模式下，只要多数节点存活，Raft组仍能正常工作
2. 对客户端几乎无感知，Leader继续处理事务请求

### 4.2 处理策略

```java
public class FollowerFailureHandler {
    
    /**
     * 处理Follower节点宕机
     */
    public static void handleFollowerFailure(String group, PeerId failedPeerId) {
        LOGGER.info("Handling follower failure in group {}: {}", group, failedPeerId);
        
        // 1. 检查该组状态
        try {
            RouteTable.getInstance().refreshLeader(
                MultiRaftServerManager.getCliClientServiceInstance(), 
                group, 
                3000);
            
            PeerId leaderId = RouteTable.getInstance().selectLeader(group);
            if (leaderId == null) {
                LOGGER.warn("No leader found in group {} after follower failure", group);
                return;
            }
            
            // 2. 获取配置信息
            Configuration conf = RouteTable.getInstance().getConfiguration(group);
            if (conf == null) {
                LOGGER.warn("Cannot get configuration for group {}", group);
                return;
            }
            
            // 3. 检查是否仍有法定人数
            List<PeerId> peers = conf.getPeers();
            int liveCount = 0;
            
            for (PeerId peer : peers) {
                if (!peer.equals(failedPeerId) && isPeerAlive(peer)) {
                    liveCount++;
                }
            }
            
            // 4. 如果活跃节点不足多数，触发告警
            if (liveCount < peers.size() / 2 + 1) {
                LOGGER.error("Critical: Group {} may lose quorum after follower {} failure. " +
                    "Only {} out of {} nodes alive!", 
                    group, failedPeerId, liveCount, peers.size());
                
                // 发送告警通知
                NotificationManager.sendAlert(
                    "Raft Group Quorum Alert",
                    String.format("Group %s may lose quorum! Only %d out of %d nodes alive.", 
                        group, liveCount, peers.size()));
            } else {
                LOGGER.info("Group {} still has quorum after follower {} failure: {} alive nodes", 
                    group, failedPeerId, liveCount);
            }
        } catch (Exception e) {
            LOGGER.error("Error handling follower failure in group {}", group, e);
        }
    }
    
    /**
     * 检查节点是否存活
     */
    private static boolean isPeerAlive(PeerId peer) {
        try {
            CliClientServiceImpl clientService = 
                (CliClientServiceImpl)MultiRaftServerManager.getCliClientServiceInstance();
            PingRequest ping = new PingRequest();
            CompletableFuture<PingResponse> future = new CompletableFuture<>();
            
            clientService.getRpcClient().invokeAsync(peer.getEndpoint(), ping, 
                new InvokeCallback() {
                    @Override
                    public void onResponse(Object result) {
                        future.complete((PingResponse)result);
                    }
                    
                    @Override
                    public void onException(Throwable e) {
                        future.completeExceptionally(e);
                    }
                }, 3000);
            
            future.get(3, TimeUnit.SECONDS);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

## 5. 多故障点组合场景处理

实际生产环境中，可能会遇到多个故障同时发生的复杂场景：

```java
public class MultiFailureScenarioHandler {
    
    /**
     * 处理复杂的多故障点场景
     */
    public static void handleComplexFailureScenario() {
        // 1. 识别故障类型和影响
        Map<String, FailureState> groupFailureStates = identifyFailureStates();
        
        // 2. 确定处理优先级 - 先恢复关键组
        List<String> prioritizedGroups = prioritizeRecovery(groupFailureStates);
        
        // 3. 按优先级处理各组故障
        for (String group : prioritizedGroups) {
            FailureState state = groupFailureStates.get(group);
            
            switch (state) {
                case LEADER_DOWN:
                    // 处理Leader宕机
                    Phase1FailureHandler.handleLeaderFailureInPhase1(group);
                    break;
                    
                case QUORUM_LOST:
                    // 法定人数丢失，需要紧急干预
                    handleQuorumLostEmergency(group);
                    break;
                    
                case FOLLOWER_DOWN:
                    // Follower宕机，但仍有法定人数
                    // 通常无需特殊处理
                    break;
                    
                case NETWORK_PARTITION:
                    // 处理网络分区
                    handleNetworkPartition(group);
                    break;
            }
        }
        
        // 4. 尝试恢复全局事务处理能力
        restoreTransactionProcessingCapability();
    }
    
    /**
     * 识别各Raft组的故障状态
     */
    private static Map<String, FailureState> identifyFailureStates() {
        Map<String, FailureState> result = new HashMap<>();
        ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
        
        for (String group : strategy.getAllGroups()) {
            try {
                RouteTable.getInstance().refreshLeader(
                    MultiRaftServerManager.getCliClientServiceInstance(), 
                    group, 
                    3000);
                
                PeerId leaderId = RouteTable.getInstance().selectLeader(group);
                Configuration conf = RouteTable.getInstance().getConfiguration(group);
                
                if (leaderId == null) {
                    // 无法识别Leader
                    if (conf != null) {
                        List<PeerId> peers = conf.getPeers();
                        int aliveCount = countAlivePeers(peers);
                        
                        if (aliveCount < peers.size() / 2 + 1) {
                            result.put(group, FailureState.QUORUM_LOST);
                        } else {
                            result.put(group, FailureState.LEADER_DOWN);
                        }
                    } else {
                        result.put(group, FailureState.UNKNOWN);
                    }
                } else {
                    // 有Leader，检查Follower状态
                    if (conf != null) {
                        List<PeerId> peers = conf.getPeers();
                        int aliveCount = countAlivePeers(peers);
                        
                        if (aliveCount == peers.size()) {
                            result.put(group, FailureState.HEALTHY);
                        } else if (aliveCount >= peers.size() / 2 + 1) {
                            result.put(group, FailureState.FOLLOWER_DOWN);
                        } else {
                            // 虽然有Leader，但活跃节点不足法定人数
                            // 可能是网络分区导致
                            result.put(group, FailureState.NETWORK_PARTITION);
                        }
                    } else {
                        result.put(group, FailureState.UNKNOWN);
                    }
                }
            } catch (Exception e) {
                LOGGER.error("Error identifying failure state for group {}", group, e);
                result.put(group, FailureState.UNKNOWN);
            }
        }
        
        return result;
    }
    
    /**
     * 确定恢复优先级
     */
    private static List<String> prioritizeRecovery(Map<String, FailureState> groupFailureStates) {
        // 按故障严重程度排序：QUORUM_LOST > LEADER_DOWN > NETWORK_PARTITION > FOLLOWER_DOWN > HEALTHY
        // 同时考虑组的事务负载和重要性
        
        List<String> result = new ArrayList<>(groupFailureStates.keySet());
        result.sort((g1, g2) -> {
            FailureState s1 = groupFailureStates.get(g1);
            FailureState s2 = groupFailureStates.get(g2);
            
            int priority1 = getFailureStatePriority(s1);
            int priority2 = getFailureStatePriority(s2);
            
            if (priority1 != priority2) {
                return priority1 - priority2;
            }
            
            // 同等优先级下，可以考虑事务数量等因素
            return getGroupTransactionCount(g2) - getGroupTransactionCount(g1);
        });
        
        return result;
    }
    
    /**
     * 处理法定人数丢失的紧急情况
     */
    private static void handleQuorumLostEmergency(String group) {
        LOGGER.error("Emergency: Group {} has lost quorum!", group);
        
        // 这种情况通常需要人工干预，但可以尝试一些自动恢复措施
        
        // 1. 发送高优先级告警
        NotificationManager.sendCriticalAlert(
            "CRITICAL: Raft Group Quorum Lost",
            String.format("Group %s has lost quorum and cannot elect a leader. Manual intervention required!", group));
        
        // 2. 尝试通过故障转移将该组的事务路由到健康的组
        try {
            LOGGER.info("Attempting to route transactions from group {} to healthy groups", group);
            ShardingStrategy strategy = EnhancedServiceLoader.load(ShardingStrategy.class);
            List<String> allGroups = strategy.getAllGroups();
            
            // 找出健康的组
            List<String> healthyGroups = allGroups.stream()
                .filter(g -> !g.equals(group) && isGroupHealthy(g))
                .collect(Collectors.toList());
            
            if (!healthyGroups.isEmpty()) {
                // 创建并注册故障转移策略
                EmergencyFailoverStrategy failoverStrategy = 
                    new EmergencyFailoverStrategy(group, healthyGroups);
                RoutingStrategyRegistry.registerEmergencyStrategy(failoverStrategy);
                
                LOGGER.info("Emergency failover activated: {} -> {}", group, healthyGroups);
            } else {
                LOGGER.error("No healthy groups available for emergency failover!");
            }
        } catch (Exception e) {
            LOGGER.error("Failed to setup emergency failover for group {}", group, e);
        }
    }
    
    private static int getFailureStatePriority(FailureState state) {
        switch (state) {
            case QUORUM_LOST: return 0;
            case LEADER_DOWN: return 1;
            case NETWORK_PARTITION: return 2;
            case FOLLOWER_DOWN: return 3;
            case HEALTHY: return 4;
            case UNKNOWN: return 5;
            default: return 6;
        }
    }
    
    /**
     * 故障状态枚举
     */
    public enum FailureState {
        HEALTHY,           // 组健康
        FOLLOWER_DOWN,     // Follower宕机但法定人数正常
        LEADER_DOWN,       // Leader宕机但可以选举新Leader
        QUORUM_LOST,       // 法定人数丢失
        NETWORK_PARTITION, // 网络分区
        UNKNOWN            // 未知状态
    }
}
```

## 总结

在Seata Multi-Raft模式下，事务阶段1节点宕机的处理策略充分利用了Raft协议的容错特性和Multi-Raft的分组隔离优势：

1. **Leader节点宕机**：通过新Leader选举和客户端重试机制确保事务能继续处理
2. **RM节点宕机**：通过事务状态检查和恢复机制处理可能的孤立事务
3. **TM节点宕机**：通过超时检测和回滚机制处理未完成的事务
4. **Follower节点宕机**：只要保持法定人数，系统正常运行
5. **多故障点组合**：根据故障严重程度优先级处理，必要时实现故障转移

这些故障处理策略确保了Seata在各种复杂故障场景下依然能保持事务一致性和系统可用性，是构建可靠分布式事务系统的关键设计。
