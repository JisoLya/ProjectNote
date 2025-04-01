# TC处理全局事务与分支事务的原理
## 全局事务开启流程

全局事务开启的请求类型为`MessageType.TYPE_GLOBAL_BEGIN`, 请求的对象为`GlobalBeginRequest`，由TM发送。

全局事务开启请求的处理，总的来说分为两部分内容：一是创建GlobalSession对象，该对象的创建意味着全局事务的开启，GlobalSession记录了应用名，XID，事务id，全局事务状态等内容，二是生成XID，并设置GlobalSession在事务状态为开启，将XID返回至TM。
下面来看具体实现，首先是协调器中处理该请求的handle方法：

```java
public GlobalBeginResponse handle(GlobalBeginRequest request, final RpcContext rpcContext) {
        GlobalBeginResponse response = new GlobalBeginResponse();
        //使用模板方法
        exceptionHandleTemplate(new AbstractCallback<GlobalBeginRequest, GlobalBeginResponse>() {
            @Override
            public void execute(GlobalBeginRequest request, GlobalBeginResponse response) throws TransactionException {
                try {
                    doGlobalBegin(request, response, rpcContext);
                } catch (StoreException e) {
                    throw new TransactionException(TransactionExceptionCode.FailedStore,
                        String.format("begin global request failed. xid=%s, msg=%s", response.getXid(), e.getMessage()),
                        e);
                }
            }
        }, request, response);
        return response;
    }
    public <T extends AbstractTransactionRequest, S extends AbstractTransactionResponse> void exceptionHandleTemplate(Callback<T, S> callback, T request, S response) {
        try {
            callback.execute(request, response);
            callback.onSuccess(request, response);
        } catch (TransactionException tex) {
            LOGGER.error("Catch TransactionException while do RPC, request: {}", request, tex);
            callback.onTransactionException(request, response, tex);
        } catch (RuntimeException rex) {
            LOGGER.error("Catch RuntimeException while do RPC, request: {}", request, rex);
            callback.onException(request, response, rex);
        }
    }
```

我们看到execute又调用了doGlobalBegin方法：
```java
protected void doGlobalBegin(GlobalBeginRequest request, GlobalBeginResponse response, RpcContext rpcContext)
        throws TransactionException {
        //这里的core是DefaultCore对象，DefaultCore的begin方法返回事务XID
        //之后response对象携带XID返回给TM，TM收到XID后表示全局事务开启成功
        response.setXid(core.begin(rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(),
            request.getTransactionName(), request.getTimeout()));
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Begin new global transaction applicationId: {},transactionServiceGroup: {}, transactionName: {},timeout:{},xid:{}",
                rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(), request.getTransactionName(), request.getTimeout(), response.getXid());
        }
    }
```
下面是DefaultCore对象的begin方法：
```java
//name：默认是应用发起全局事务的方法名，可以通过@GlobalTransactional(name="beginTransaction")修改该值
    //applicationId：发起全局事务的应用名
    //transactionServiceGroup：发起全局事务的事务分组名
    //timeout：事务超时时间，默认300s，可以通过@GlobalTransactional(timeoutMills = 3000)设置
    public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
        throws TransactionException {
		//创建全局会话，在seata中事务称为会话
        GlobalSession session = GlobalSession.createGlobalSession(applicationId, transactionServiceGroup, name,
            timeout);
        //设置监听器，监听器的作用主要是写事务日志，关于监听器后续单独文章介绍
        session.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        //开启事务
        session.begin();
        // transaction start event
        // 发布全局事务开启事件，这里用到了Guava里面的EventBus工具类，这个后面单独文章介绍
        eventBus.post(new GlobalTransactionEvent(session.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            session.getTransactionName(), session.getBeginTime(), null, session.getStatus()));
        //返回XID
        return session.getXid();
    }
```
上面代码用到了GlobalSession里面的两个方法：createGlobalSession和begin，下面分别来看一下：
```java
public static GlobalSession createGlobalSession(String applicationId, String txServiceGroup, String txName,
                                                    int timeout) {
        //GlobalSession的构造方法在下面
        GlobalSession session = new GlobalSession(applicationId, txServiceGroup, txName, timeout);
        return session;
    }
    public GlobalSession(String applicationId, String transactionServiceGroup, String transactionName, int timeout) {
        this.transactionId = UUIDGenerator.generateUUID();
        this.status = GlobalStatus.Begin;
        this.applicationId = applicationId;
        this.transactionServiceGroup = transactionServiceGroup;
        this.transactionName = transactionName;
        this.timeout = timeout;
        //XID的组成=IP:port:transactionId
        this.xid = XID.generateXID(transactionId);
    }
    public void begin() throws TransactionException {
        this.status = GlobalStatus.Begin;
        //设置事务开启时间
        this.beginTime = System.currentTimeMillis();
        //事务为激活状态，直到GlobalSession对象关闭
        this.active = true;
        //调用监听器，后续单独文章分析
        for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
            lifecycleListener.onBegin(this);
        }
    }
```
note： XID的组成： TC的IP：TC的监听端口：UUID。

## 分支事务注册请求
分支事务注册请求的消息类型是`MessageType.TYPE_BRANCH_REGISTER`,请求对象为`BranchRegisterRequest`,请求由RM发出。用于开启分支事务。

处理该请求也是在模板方法中进行的，在模板方法里面直接调用了DefaultCoordinator.doBranchRegister方法，下面直接来看doBranchRegister方法方法：

```java
protected void doBranchRegister(BranchRegisterRequest request, BranchRegisterResponse response,
                                    RpcContext rpcContext) throws TransactionException {
        //分支事务注册，TC生成一个分支ID，并返回到RM
        //下面要调用DefaultCore的branchRegister方法
        response.setBranchId(
            core.branchRegister(request.getBranchType(), request.getResourceId(), rpcContext.getClientId(),
                request.getXid(), request.getApplicationData(), request.getLockKey()));
    }
    //下面是DefaultCore的branchRegister方法
    /** 
     * 
     * @param branchType 分支类型，我们现在使用的是AT模式，branchType为AT
     * @param resourceId RM连接数据库的URL 
     * @param clientId 客户端ID，组成是应用名:IP:端口 
     * @param xid the xid 
     * @param applicationData 未知 
     * @param lockKeys 需要锁定记录的主键值和表，这个非常关键。RM在操作数据库前会解析SQL语句 
     * 分析出SQL语句要操作的表和主键值，并将这两个值拼成字符串发送到TC
     **/
    @Override
    public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                               String applicationData, String lockKeys) throws TransactionException {
       	//AT模式下branchType为AT，getCore方法返回的是ATCore对象
        return getCore(branchType).branchRegister(branchType, resourceId, clientId, xid,
            applicationData, lockKeys);
    }
```

branchRegister方法调用了ATCore的branchRegister方法去注册分支事务：
```java
public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                               String applicationData, String lockKeys) throws TransactionException {
        //根据XID找到GlobalSession对象，这个对象是全局事务开启时创建的
        GlobalSession globalSession = assertGlobalSessionNotNull(xid, false);
        //1、lockAndExecute方法中要对GlobalSession对象上锁
        return SessionHolder.lockAndExecute(globalSession, () -> {
            //检查事务状态是否为开启和激活状态
            globalSessionStatusCheck(globalSession);
            //添加监听器，后续专门文章解析
            globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
            //创建分支会话BranchSession对象
            BranchSession branchSession = SessionHelper.newBranchByGlobal(globalSession, branchType, resourceId,
                    applicationData, lockKeys, clientId);
            //2、对记录进行上锁
            branchSessionLock(globalSession, branchSession);
            try {
                //设置分支事务状态为BranchStatus.Registered，并且将该分支事务添加到GlobalSession中
                globalSession.addBranch(branchSession);
            } catch (RuntimeException ex) {
                branchSessionUnlock(branchSession);
                throw new BranchTransactionException(FailedToAddBranch, String
                        .format("Failed to store branch xid = %s branchId = %s", globalSession.getXid(),
                                branchSession.getBranchId()), ex);
            }
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("Register branch successfully, xid = {}, branchId = {}, resourceId = {} ,lockKeys = {}",
                    globalSession.getXid(), branchSession.getBranchId(), resourceId, lockKeys);
            }
            //返回分支事务ID
            return branchSession.getBranchId();
        });
    }
```
这里出现了两个上锁，分别是 `SessionHolder.lockAndExecute()`和`branchSessionLock()`分别看看源码
```java
public static <T> T lockAndExecute(GlobalSession globalSession, GlobalSession.LockCallable<T> lockCallable)
            throws TransactionException {
        //这里根会话管理器使用的是FileSessionManager，其lockAndExecute方法见下面
        return getRootSessionManager().lockAndExecute(globalSession, lockCallable);
    }
    public <T> T lockAndExecute(GlobalSession globalSession, GlobalSession.LockCallable<T> lockCallable)
            throws TransactionException {
        //全局事务上锁，这个锁定的是事务自身，如果在全局事务中存在并发分支事务，这里会被拦截
        globalSession.lock();
        try {
        	//调用回调方法
            return lockCallable.call();
        } finally {
            globalSession.unlock();
        }
    }
```
第一个加锁相对比较简单，就是对全局会话GlobalSession直接加锁，底层使用的是ReentrantLock，如果当前已经被加锁，最长锁等待时间是2000ms。代码如下：
```java
private Lock globalSessionLock = new ReentrantLock();
    private static final int GLOBAL_SESSION_LOCK_TIME_OUT_MILLS = 2 * 1000;
    public void lock() throws TransactionException {
        try {
           if (globalSessionLock.tryLock(GLOBAL_SESSION_LOCK_TIME_OUT_MILLS, TimeUnit.MILLISECONDS)) {
               return;
            }
        } catch (InterruptedException e) {
            LOGGER.error("Interrupted error", e);
    	}
    	throw new GlobalTransactionException(TransactionExceptionCode.FailedLockGlobalTranscation, "Lock global session failed");
	}
```

下面来看第二个加锁的地方，也就是方法branchSessionLock:

```java
protected void branchSessionLock(GlobalSession globalSession, BranchSession branchSession) throws TransactionException {
		//调用分支会话加锁，如果加锁失败，则抛出异常
        if (!branchSession.lock()) {
            throw new BranchTransactionException(LockKeyConflict, String
                    .format("Global lock acquire failed xid = %s branchId = %s", globalSession.getXid(),
                            branchSession.getBranchId()));
        }
    }
    //下面是BranchSession的lock方法，从这里可以看到首先获取锁管理器，然后使用锁管理器上锁
    public boolean lock() throws TransactionException {
        if (this.getBranchType().equals(BranchType.AT)) {
            return LockerManagerFactory.getLockManager().acquireLock(this);
        }
        return true;
    }
```
seata锁管理器与存储模式有关，存储模式可以是db、file、redis，这里使用的是file，对应的类是FileLockManager。下面是FileLockManager的acquireLock方法：
```java
public boolean acquireLock(BranchSession branchSession) throws TransactionException {
        if (branchSession == null) {
            throw new IllegalArgumentException("branchSession can't be null for memory/file locker.");
        }
        //得到本分支事务需要加锁的记录和表名，RM将要加锁的记录主键值和表名组装成字符串发送到TC
        String lockKey = branchSession.getLockKey();
        if (StringUtils.isNullOrEmpty(lockKey)) {
            // no lock
            return true;
        }
        // 创建RowLock集合，RowLock对象表示要加锁的表及主键中字段的值，一条加锁记录一个RowLock对象
        List<RowLock> locks = collectRowLocks(branchSession);
        if (CollectionUtils.isEmpty(locks)) {
            // no lock
            return true;
        }
        //当前设置存储模式是file，getLocker方法返回的是FileLocker
        return getLocker(branchSession).acquireLock(locks);
    }
    protected List<RowLock> collectRowLocks(BranchSession branchSession) {
        List<RowLock> locks = new ArrayList<>();
        if (branchSession == null || StringUtils.isBlank(branchSession.getLockKey())) {
            return locks;
        }
        String xid = branchSession.getXid();
        //得到资源id，也就是数据库连接URL
        String resourceId = branchSession.getResourceId();
        //得到事务ID，分支事务的transactionId与GlobalSession的transactionId是相同的
        long transactionId = branchSession.getTransactionId();
        String lockKey = branchSession.getLockKey();
        //遍历主键中的每个字段值，对每个字段值创建RowLock对象
        return collectRowLocks(lockKey, resourceId, xid, transactionId, branchSession.getBranchId());
    }
    protected List<RowLock> collectRowLocks(String lockKey, String resourceId, String xid, Long transactionId,
                                            Long branchID) {
        List<RowLock> locks = new ArrayList<RowLock>();
        //当要对多个记录加锁时，中间使用";"分隔
        String[] tableGroupedLockKeys = lockKey.split(";");
        for (String tableGroupedLockKey : tableGroupedLockKeys) {
            //表名和记录主键值之间使用":"分隔
            int idx = tableGroupedLockKey.indexOf(":");
            if (idx < 0) {
                return locks;
            }
            //tableName为要加锁的表名
            String tableName = tableGroupedLockKey.substring(0, idx);
            //mergedPKs为要加锁的记录主键值，如果主键有多个字段，则使用"_"分隔
            //如果需要一次加锁多条记录，则记录之间使用","分隔
            String mergedPKs = tableGroupedLockKey.substring(idx + 1);
            if (StringUtils.isBlank(mergedPKs)) {
                return locks;
            }
            String[] pks = mergedPKs.split(",");
            if (pks == null || pks.length == 0) {
                return locks;
            }
            //遍历主键中的每个字段，创建RowLock对象，每个主键值都使用字符串表示
            for (String pk : pks) {
                if (StringUtils.isNotBlank(pk)) {
                    RowLock rowLock = new RowLock();
                    rowLock.setXid(xid);
                    rowLock.setTransactionId(transactionId);
                    rowLock.setBranchId(branchID);
                    rowLock.setTableName(tableName);
                    rowLock.setPk(pk);
                    rowLock.setResourceId(resourceId);
                    locks.add(rowLock);
                }
            }
        }
        return locks;
    }
```

上面的`collectRowLocks`方法主要是遍历需要加锁的记录，加锁的记录存储在`BranchSession`的`lockKey`属性中，如果需要对一个表的两条记录加锁，而且该表主键有两个字段，则`lockKey`的组成是：

表名:主键值_主键值;表名:主键值_主键值

`collectRowLocks`方法遍历`lockKey`的每条记录，并且一条记录生成一个`RowLock`对象。然后调用`FileLocker.acquireLock`方法对`RowLock`对象加锁。

`acquireLock`方法遍历所有的`RowLock`对象，对记录加锁时，首先查看当前记录是否已经加锁，如果没有加过，则直接记录主键值和事务id，表示对该记录上锁，如果已经加过，则比较本次加锁的事务id与上次的事务id是否一致，如果一致，则直接跳过，如果不一致，意味着加锁失败，接下来会对该分支事务已经加过的锁解锁，然后返回RM加锁失败。

`acquireLock`里面创建`BucketLockMap`对象记录了加锁的主键值和加锁的事务id的对应关系，表示该记录是由该事务加锁，`BucketLockMap`可以简单认为是Map对象，然后`BucketLockMap`会被存储到两个地方：`BranchSession`的`lockHolder`属性和`LOCK_MAP`。

`lockHolder`属性是一个`Map`，`key`是`BucketLockMap`对象，`value`是一个集合，存储了数据库的主键值，凡是记录到该集合中就表示本分支事务对这些记录加了锁，释放锁的时候就根据`value`集合从`BucketLockMap`对象中删除记录。下面重点介绍`LOCK_MAP`。

`LOCK_MAP`是一个全局的锁集合，记录了目前所有事务正在加的锁，判断记录是否已经加锁以及加锁的事务id就是通过`LOCK_MAP`完成的，它的类型如下：

```java
ConcurrentMap<String, ConcurrentMap<String, ConcurrentMap<Integer, BucketLockMap>>>
```
结构是：数据库资源->表名->桶->BucketLockMap。
`LOCK_MAP`将每个数据库表下面分了128个桶，每个桶对应一个`BucketLockMap`，一条加锁记录分在哪个桶，是根据主键值的hash值模128得到的，如果分到某个桶下，就将主键值和事务id添加到桶对应的`BucketLockMap`中，所以`BucketLockMap`里面其实存储了不同事务的加锁记录，下面来看一下具体实现：

```java
public boolean acquireLock(List<RowLock> rowLocks) {
        //如果集合为空，表示没有要加锁的记录
        if (CollectionUtils.isEmpty(rowLocks)) {
            //no lock
            return true;
        }
        String resourceId = branchSession.getResourceId();
        long transactionId = branchSession.getTransactionId();
        //bucketHolder：BucketLockMap->主键值集合，表示当前分支事务有哪些记录已经加锁了
        //在branchSession中记录lockHolder是为了在释放锁的时候使用
        //BucketLockMap是一个Map对象，是记录主键值与transactionId的对应关系
        ConcurrentMap<BucketLockMap, Set<String>> bucketHolder = branchSession.getLockHolder();
        //dbLockMap：表名->桶->BucketLockMap，
        //LOCK_MAP：数据库资源->表名->桶->BucketLockMap，
        //LOCK_MAP记录了全局所有事务的所有加锁记录
        ConcurrentMap<String, ConcurrentMap<Integer, BucketLockMap>> dbLockMap = LOCK_MAP.get(resourceId);
        if (dbLockMap == null) {
            LOCK_MAP.putIfAbsent(resourceId, new ConcurrentHashMap<>());
            dbLockMap = LOCK_MAP.get(resourceId);
        }
        //遍历每个需要加锁的记录
        for (RowLock lock : rowLocks) {
            String tableName = lock.getTableName();
            //主键值
            String pk = lock.getPk();
            ConcurrentMap<Integer, BucketLockMap> tableLockMap = dbLockMap.get(tableName);
            if (tableLockMap == null) {
                dbLockMap.putIfAbsent(tableName, new ConcurrentHashMap<>());
                tableLockMap = dbLockMap.get(tableName);
            }
            //得到桶的值，桶的个数是128个
            int bucketId = pk.hashCode() % BUCKET_PER_TABLE;
            BucketLockMap bucketLockMap = tableLockMap.get(bucketId);
            if (bucketLockMap == null) {
                tableLockMap.putIfAbsent(bucketId, new BucketLockMap());
                bucketLockMap = tableLockMap.get(bucketId);
            }
            //previousLockTransactionId为之前的事务id
            Long previousLockTransactionId = bucketLockMap.get().putIfAbsent(pk, transactionId);
            //下面分为三种情况，
            // 一是当前加锁的事务与之前已经加过锁的事务是同一个，
            // 二是两个事务不是同一个，
            // 三是之前还没有加过锁
            if (previousLockTransactionId == null) {
                //No existing lock, and now locked by myself
                Set<String> keysInHolder = bucketHolder.get(bucketLockMap);
                if (keysInHolder == null) {
                    bucketHolder.putIfAbsent(bucketLockMap, new ConcurrentSet<>());
                    keysInHolder = bucketHolder.get(bucketLockMap);
                }
                //如果该记录没有加过锁，则直接将记录保存到集合中
                keysInHolder.add(pk);
            } else if (previousLockTransactionId == transactionId) {
                //如果加锁的事务是同一个，则跳过加锁
                // Locked by me before
                continue;
            } else {
                LOGGER.info("Global lock on [" + tableName + ":" + pk + "] is holding by " + previousLockTransactionId);
                try {
                    //如果要加锁的记录已经被其他事务加过锁，则释放当前分支事务已经加过的锁，并返回加锁失败
                    // Release all acquired locks.
                    branchSession.unlock();
                } catch (TransactionException e) {
                    throw new FrameworkException(e);
                }
                return false;
            }
        }
        return true;
    }
```

## 分支报告状态请求
分支状态报告请求的消息类型是`MessageType.TYPE_BRANCH_STATUS_REPORT`，请求对象为`BranchReportRequest`，该请求由RM发起。该请求的作用是报告某个分支事务的状态，TC收到后更改对应`BranchSession`对象的状态。

处理该请求通过模板方法调用了`DefaultCoordinator.doBranchReport`：
```java
protected void doBranchReport(BranchReportRequest request, BranchReportResponse response, RpcContext rpcContext)
        throws TransactionException {
        //调用DefaultCore的branchReport方法
        core.branchReport(request.getBranchType(), request.getXid(), request.getBranchId(), request.getStatus(),
            request.getApplicationData());
    }
```

在`doBranchReport`方法中又调用了`DefaultCore`的`branchReport`方法，在`branchReport`方法中又调用了其他方法，最终调用到`ATCore`的`branchReport`方法：
```java
public void branchReport(BranchType branchType, String xid, long branchId, BranchStatus status,
                             String applicationData) throws TransactionException {
        //根据XID得到全局事务对象GlobalSession
        GlobalSession globalSession = assertGlobalSessionNotNull(xid, true);
        //根据branchId得到分支事务对象branchSession
        BranchSession branchSession = globalSession.getBranch(branchId);
        if (branchSession == null) {
            throw new BranchTransactionException(BranchTransactionNotExist,
                    String.format("Could not found branch session xid = %s branchId = %s", xid, branchId));
        }
        //添加监听器
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        //更改分支事务状态，本请求的处理核心在下面这个方法中
        globalSession.changeBranchStatus(branchSession, status);
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Report branch status successfully, xid = {}, branchId = {}", globalSession.getXid(),
                branchSession.getBranchId());
        }
    }
```
`branchReport`方法在最后调用了`GlobalSession`的`changeBranchStatus`方法，这个方法也非常简单，就是直接更改了分支事务的状态：
```java
public void changeBranchStatus(BranchSession branchSession, BranchStatus status)
        throws TransactionException {
        //设置分支事务对象的状态
        branchSession.setStatus(status);
        //通知监听器，监听器的处理后续文章介绍
        for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
            lifecycleListener.onBranchStatusChange(this, branchSession, status);
        }
    }
```
## 全局事务报告请求
全局事务报告请求的消息类型是`MessageType.TYPE_GLOBAL_REPORT`，请求对象为`GlobalReportRequest`，该请求由TM发起。该请求的作用是报告某个全局事务的状态，TC收到后更改该全局事务的状态。
全局事务报告请求的处理通过模板方法调用了协调器对象`DefaultCoordinator`的`doGlobalReport`方法，之后在`doGlobalReport`方法里面进一步调用到`DefaultCore`的`globalReport`方法：
```java
public GlobalStatus globalReport(String xid, GlobalStatus globalStatus) throws TransactionException {
        //根据XID找到全局事务
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
        if (globalSession == null) {
            return globalStatus;
        }
        //添加监听器
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        doGlobalReport(globalSession, xid, globalStatus);
        return globalSession.getStatus();
    }

    @Override
    public void doGlobalReport(GlobalSession globalSession, String xid, GlobalStatus globalStatus) throws TransactionException {
        if (globalSession.isSaga()) {
        	//getCore(BranchType.SAGA)方法返回的对象是SagaCore，现在使用的是AT模式，不知道为什么调用SagaCore对象
            getCore(BranchType.SAGA).doGlobalReport(globalSession, xid, globalStatus);
        }
    }
```

`globalReport`和`doGlobalReport`两个方法也非常简单，就是找到全局事务对象，然后调用`SagaCore`的`doGlobalReport`方法：
```java
public void doGlobalReport(GlobalSession globalSession, String xid, GlobalStatus globalStatus) throws TransactionException {
        //全局事务提交
        if (GlobalStatus.Committed.equals(globalStatus)) {
            removeAllBranches(globalSession);
            SessionHelper.endCommitted(globalSession);
            LOGGER.info("Global[{}] committed", globalSession.getXid());
        }
        //全局事务回滚，Finished未知，等分析TM的时候在介绍该状态
        else if (GlobalStatus.Rollbacked.equals(globalStatus)
                || GlobalStatus.Finished.equals(globalStatus)) {
            removeAllBranches(globalSession);
            SessionHelper.endRollbacked(globalSession);
            LOGGER.info("Global[{}] rollbacked", globalSession.getXid());
        }
        //其他状态
        else {
            globalSession.changeStatus(globalStatus);
            LOGGER.info("Global[{}] reporting is successfully done. status[{}]", globalSession.getXid(), globalSession.getStatus());
            if (GlobalStatus.RollbackRetrying.equals(globalStatus)
                    || GlobalStatus.TimeoutRollbackRetrying.equals(globalStatus)
                    || GlobalStatus.UnKnown.equals(globalStatus)) {
                globalSession.queueToRetryRollback();
                LOGGER.info("Global[{}] will retry rollback", globalSession.getXid());
            } else if (GlobalStatus.CommitRetrying.equals(globalStatus)) {
                globalSession.queueToRetryCommit();
                LOGGER.info("Global[{}] will retry commit", globalSession.getXid());
            }
        }
    }
```
从代码中可以看到，三种情况来处理全局事务状态
#### 全局事务提交

提交全局事务调用了两个方法：`removeAllBranches`和`SessionHelper.endCommitted`。
`removeAllBranches`方法将`GlobalSession`对象中的分支事务集合清空，同时释放分支事务对数据库记录加的锁，释放锁其实就是将上一篇文章介绍的`BucketLockMap`对象中的数据库记录和加锁的事务id删除。`SessionHelper.endCommitted`将修改`GlobalSession`对象中的状态为已提交（`GlobalStatus.Committed`）：

```java
private void removeAllBranches(GlobalSession globalSession) throws TransactionException {
        //得到所有的分支事务
        ArrayList<BranchSession> branchSessions = globalSession.getSortedBranches();
        for (BranchSession branchSession : branchSessions) {
            globalSession.removeBranch(branchSession);
        }
    }
    //下面是GlobalSession中的方法
    public void removeBranch(BranchSession branchSession) throws TransactionException {
        for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
            lifecycleListener.onRemoveBranch(this, branchSession);
        }
        //释放分支事务加的锁
        branchSession.unlock();
        //删除分支事务
        remove(branchSession);
    }
    public boolean remove(BranchSession branchSession) {
        return branchSessions.remove(branchSession);
    }
```
下面是`SessionHelper.endCommitted`方法：
```java
public static void endCommitted(GlobalSession globalSession) throws TransactionException {
        globalSession.changeStatus(GlobalStatus.Committed);
        globalSession.end();
    }
    //下面是GlobalSession的方法
    public void end() throws TransactionException {
        // Clean locks first
        clean();
        for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
            lifecycleListener.onEnd(this);
        }
    }
    public void clean() throws TransactionException {
    	//下面的代码也是释放分支事务的锁，因为removeAllBranches方法已经释放过，所以下面的方法是空执行
        LockerManagerFactory.getLockManager().releaseGlobalSessionLock(this);
    }
```
#### 全局事务回滚
回滚全局事务的首先也是执行`removeAllBranches`方法，所以第一步也是释放分支事务加的锁。下面我们重点看一下`SessionHelper.endRollbacked`：
```java
public static void endRollbacked(GlobalSession globalSession) throws TransactionException {
        GlobalStatus currentStatus = globalSession.getStatus();
        //isTimeoutGlobalStatus检查当前状态是否是超时的状态
        if (isTimeoutGlobalStatus(currentStatus)) {
            globalSession.changeStatus(GlobalStatus.TimeoutRollbacked);
        } else {
            globalSession.changeStatus(GlobalStatus.Rollbacked);
        }
        globalSession.end();
    }
    public static boolean isTimeoutGlobalStatus(GlobalStatus status) {
        return status == GlobalStatus.TimeoutRollbacked
                || status == GlobalStatus.TimeoutRollbackFailed
                || status == GlobalStatus.TimeoutRollbacking
                || status == GlobalStatus.TimeoutRollbackRetrying;
    }
```
`endRollbacked`方法首先设置事务的状态，然后调用了`globalSession.end`方法，`globalSession.end`方法与提交全局事务中调用的是同一个，都是用于释放分支事务锁的。

`endRollbacked`方法在修改全局事务状态前，会比较当前事务的状态，如果当前事务状态是超时，则设置事务状态为超时回滚成功，如果不是则设置事务状态为回滚成功。

至于全局事务的状态的流转，则在分析完RM和TM之后在介绍。

#### 其他事务状态
如果请求中的事务状态不是前面两种，则根据请求直接修改`GlobalSession`的状态，之后是一个分支判断，下面重点看一下分支判断：
```java
//事务回滚中、超时回滚重试中、未知状态
			if (GlobalStatus.RollbackRetrying.equals(globalStatus)
                    || GlobalStatus.TimeoutRollbackRetrying.equals(globalStatus)
                    || GlobalStatus.UnKnown.equals(globalStatus)) {
                globalSession.queueToRetryRollback();
                LOGGER.info("Global[{}] will retry rollback", globalSession.getXid());
            } 
			//提交重试
			else if (GlobalStatus.CommitRetrying.equals(globalStatus)) {
                globalSession.queueToRetryCommit();
                LOGGER.info("Global[{}] will retry commit", globalSession.getXid());
            }
```
分支判断中分别调用了`queueToRetryRollback`和`queueToRetryCommit`。
`queueToRetryRollback`：
```java
public void queueToRetryRollback() throws TransactionException {
        //添加监听器
        this.addSessionLifecycleListener(SessionHolder.getRetryRollbackingSessionManager());
        //seata提供了回滚重试管理器，下面这行代码将本全局事务对象添加到回滚重试管理器中
        //回滚重试管理器后面文章介绍
        SessionHolder.getRetryRollbackingSessionManager().addGlobalSession(this);
        GlobalStatus currentStatus = this.getStatus();
        //设置事务状态
        if (SessionHelper.isTimeoutGlobalStatus(currentStatus)) {
            this.changeStatus(GlobalStatus.TimeoutRollbackRetrying);
        } else {
            this.changeStatus(GlobalStatus.RollbackRetrying);
        }
    }
```
`queueToRetryCommit`：
```java
public void queueToRetryCommit() throws TransactionException {
        //设置监听器
        this.addSessionLifecycleListener(SessionHolder.getRetryCommittingSessionManager());
        //将本全局事务对象添加到重试提交管理器中
        //重试提交管理器后面文章介绍
        SessionHolder.getRetryCommittingSessionManager().addGlobalSession(this);
        //设置事务状态
        this.changeStatus(GlobalStatus.CommitRetrying);
    }
```
总结，报告全局事务状态的时候主要分为一下几个状态：
1. 全局事务提交，删除分支事务在数据库加的记录锁，修改全局事务的状态为已提交;
2. 事务回滚，释放分支事务加的数据库记录锁，更改事务状态为回滚成功或者超时回滚成功；
3. 事务回滚中、超时回滚重试中、未知状态，更改事务状态，将GlobalSession对象添加到回滚重试管理器；
4. 提交重试，更改事务状态，将GlobalSession对象添加到重试提交管理器；
5. 其他状态，更改事务状态。

## 全局事务提交请求
全局事务提交请求的消息类型是`MessageType.TYPE_GLOBAL_COMMIT`，请求对象为`GlobalCommitRequest`，该请求由TM发起。该请求的作用是报告某个全局事务执行完毕请求提交。TC收到后要做出以下更改：
- 对全局事务对象GlobalSession加锁
- 释放分支事务锁，修改GlobalSession的状态为提交中；
- 释放GlobalSession锁；
- 当前处于AT模式，事务可以异步提交，将当前全局事务添加到异步提交管理器；
- 修改全局事务状态为异步提交中；
- 返回TM当前全局事务状态。

这里简单介绍一下异步提交管理器后续对全局事务做的处理是：通知各个分支事务，要求分支事务提交，如果分支事务提交都成功了，则修改全局事务状态为提交成功，如果有失败的，则重试通知。异步提交管理器后面有文章专门做介绍。

下面来看一下代码逻辑。下面直接从DefaultCore的commit方法开始介绍。
```java
public GlobalStatus commit(String xid) throws TransactionException {
        //根据XID找到全局事务对象GlobalSession
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
        if (globalSession == null) {
            //如果GlobalSession没有找到，说明当前事务是非seata管理的
            return GlobalStatus.Finished;
        }
        //添加监听器
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        // just lock changeStatus
        //shouldCommit表示当前事务是否可以提交，如果事务状态为GlobalStatus.Begin，则可以提交，否则不可以
        boolean shouldCommit = SessionHolder.lockAndExecute(globalSession, () -> {
            // the lock should release after branch commit
            // Highlight: Firstly, close the session, then no more branch can be registered.
            //closeAndClean：将关闭消息通知上面注册的监听器，然后释放分支事务锁
            globalSession.closeAndClean();
            //如果事务状态为开始，那么将状态直接改为提交中
            if (globalSession.getStatus() == GlobalStatus.Begin) {
                globalSession.changeStatus(GlobalStatus.Committing);
                return true;
            }
            return false;
        });
        if (!shouldCommit) {
            //返回当前事务状态，表示禁止全局事务提交，事务提交失败
            return globalSession.getStatus();
        }
        //遍历分支事务是否可以异步提交，如果分支事务有TCC或者XA的，则不能异步提交
        //本文介绍的场景都是AT模式的，因此globalSession.canBeCommittedAsync()返回true
        //下面只介绍if分支
        if (globalSession.canBeCommittedAsync()) {
            //执行异步提交
            globalSession.asyncCommit();
            //返回事务状态为提交成功
            return GlobalStatus.Committed;
        } else {
            //执行同步提交
            doGlobalCommit(globalSession, false);
        }
        //返回最终事务状态
        return globalSession.getStatus();
    }
```
上面代码使用到了`globalSession.asyncCommit`方法，下面来看一下这个方法：
```java
public void asyncCommit() throws TransactionException {
        //添加监听器
        this.addSessionLifecycleListener(SessionHolder.getAsyncCommittingSessionManager());
        //将当前全局事务添加到异步提交管理器，异步提交管理器后续文章介绍
        SessionHolder.getAsyncCommittingSessionManager().addGlobalSession(this);
        //修改事务状态为异步提交中
        this.changeStatus(GlobalStatus.AsyncCommitting);
    }
```
Q:为什么在AT模式下，不先通知分支事务提交，再去释放任务分支锁？

这里我理解是，既然TM通知全局事务提交了，那么说明当前事务的执行过程是成功的，没有错误，而且分支事务在执行结束后就已经提交了，也就是在全局事务提交前，分支事务对数据的改变就已经写入数据库了，TC通知分支事务提交是为了处理回滚日志等，这些处理与业务处理关系不大，即使分支事务提交失败也不会有影响，但是分支事务加的锁就不同了，如果锁一直加着，就会影响其他事务的执行，严重可能造成大量事务执行失败，所以先释放锁，让其他需要锁的事务可以正常执行，至于分支事务提交可以异步进行，即使失败也没有影响。

## 全局事务回滚请求
全局事务回滚请求的消息类型是`MessageType.TYPE_GLOBAL_ROLLBACK`，请求对象为`GlobalRollbackRequest`，该请求由TM发起。该请求的作用是回滚全局事务。TC收到后要做出以下更改：
- 全局事务对象加锁；
- 修改全局事务状态为回滚中；
- 全局事务对象解锁；
- 通知各个分支事务回滚；
- 如果有分支事务回顾失败的，则通知TM回滚失败，如果分支事务全部回滚成功，则更改全局事务状态为回滚成功，并且释放分支事务锁。

对于上述流程，有个问题就是对于回滚失败的分支事务怎么处理，因为一个全局事务包含多个分支事务，可能有的回滚成功了，有的回滚失败，这样会造成事务的不一致，这个问题等到介绍TM的时候在分析。

下面看一下具体的代码实现，从`DefaultCore`的`rollback`方法看起：
```java
public GlobalStatus rollback(String xid) throws TransactionException {
        //获得XID对应的GlobalSession
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
        if (globalSession == null) {
            //如果没有找到，说明当前事务不是seat管理的
            return GlobalStatus.Finished;
        }
        //添加监听器
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        // just lock changeStatus
        //lockAndExecute方法会对全局事务对象GlobalSession加锁，然后执行里面的回调方法，
        //回调成功后，再解锁
        boolean shouldRollBack = SessionHolder.lockAndExecute(globalSession, () -> {
            //close方法是将关闭事件通知给之前注册的监听器
            globalSession.close(); // Highlight: Firstly, close the session, then no more branch can be registered.
            if (globalSession.getStatus() == GlobalStatus.Begin) {
                //修改状态为回滚中
                globalSession.changeStatus(GlobalStatus.Rollbacking);
                return true;
            }
            return false;
        });
        if (!shouldRollBack) {
            return globalSession.getStatus();
        }
	        doGlobalRollback(globalSession, false);
        return globalSession.getStatus();
    }
    public boolean doGlobalRollback(GlobalSession globalSession, boolean retrying) throws TransactionException {
        boolean success = true;
        // start rollback event
        //发布事务回滚中的事件
        eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            globalSession.getTransactionName(), globalSession.getBeginTime(), null, globalSession.getStatus()));
        //当前使用的模式是AT模式，因此只执行else分支
        if (globalSession.isSaga()) {
            success = getCore(BranchType.SAGA).doGlobalRollback(globalSession, retrying);
        } else {
            //遍历全局事务下的每个分支事务
            for (BranchSession branchSession : globalSession.getReverseSortedBranches()) {
                BranchStatus currentBranchStatus = branchSession.getStatus();
                if (currentBranchStatus == BranchStatus.PhaseOne_Failed) {
                    //如果分支事务在一阶段失败了，说明事务更改没有写入数据库，
                    //则分支事务无需做操作，将该分支事务从全局事务中删除即可
                    globalSession.removeBranch(branchSession);
                    continue;
                }
                try {
                    //branchRollback：构建BranchRollbackRequest请求对象，通知分支事务做回滚
                    BranchStatus branchStatus = branchRollback(globalSession, branchSession);
                    switch (branchStatus) {
                        case PhaseTwo_Rollbacked:
                            //如果二阶段回退成功，则直接将分支事务从全局事务中删除
                            globalSession.removeBranch(branchSession);
                            LOGGER.info("Rollback branch transaction successfully, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            continue;
                        case PhaseTwo_RollbackFailed_Unretryable:
                            //事务回滚失败，则修改全局事务状态为回滚失败，并且释放所有的分支事务锁
                            //然后给TM返回回滚失败
                            SessionHelper.endRollbackFailed(globalSession);
                            LOGGER.info("Rollback branch transaction fail and stop retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            return false;
                        default:
                            LOGGER.info("Rollback branch transaction fail and will retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            if (!retrying) {
                                //将全局事务加入到回滚重试管理器
                                globalSession.queueToRetryRollback();
                            }
                            return false;
                    }
                } catch (Exception ex) {
                    StackTraceLogger.error(LOGGER, ex,
                        "Rollback branch transaction exception, xid = {} branchId = {} exception = {}",
                        new String[] {globalSession.getXid(), String.valueOf(branchSession.getBranchId()), ex.getMessage()});
                    if (!retrying) {
                    	//如果异常了，则将全局事务加入到回滚重试管理器 
                        globalSession.queueToRetryRollback();
                    }
                    throw new TransactionException(ex);
                }
            }
            // In db mode, there is a problem of inconsistent data in multiple copies, resulting in new branch
            // transaction registration when rolling back.
            // 1. New branch transaction and rollback branch transaction have no data association
            // 2. New branch transaction has data association with rollback branch transaction
            // The second query can solve the first problem, and if it is the second problem, it may cause a rollback
            // failure due to data changes.
            //能进行到这个位置，说明分支事务全部回滚成功了，GlobalSession中已经没有分支事务了
            //根据注释，增加下面的代码应该是在db模式下出过问题，但是新分支注册会对全局事务对象加锁，回滚也会加锁，
            //所以回滚时，不会发生新的分支事务注册情况，对于上面的注释大家如果知道原因烦请指导
            GlobalSession globalSessionTwice = SessionHolder.findGlobalSession(globalSession.getXid());
            if (globalSessionTwice != null && globalSessionTwice.hasBranch()) {
                LOGGER.info("Rollbacking global transaction is NOT done, xid = {}.", globalSession.getXid());
                return false;
            }
        }
        //success永远都是true
        if (success) {
            //endRollbacked更改全局事务状态为回滚成功，并且释放分支事务锁
            SessionHelper.endRollbacked(globalSession);
            // rollbacked event
            //发布回滚成功事件
            eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
                globalSession.getTransactionName(), globalSession.getBeginTime(), System.currentTimeMillis(),
                globalSession.getStatus()));
            LOGGER.info("Rollback global transaction successfully, xid = {}.", globalSession.getXid());
        }
        return success;
    }
```

全局事务回滚请求的代码逻辑和全局事务提交请求的逻辑比较类似。
全局事务提交请求和全局事务回滚请求的代码里面都在GlobalSession对象中添加了监听器，然后在执行下面这行代码时，会通知监听器：
```java
globalSession.close();
	//globalSession的close方法
	public void close() throws TransactionException {
        if (active) {
            for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
                lifecycleListener.onClose(this);
            }
        }
    }
```
下面我们看一下监听器的处理逻辑：
```java
public void onClose(GlobalSession globalSession) throws TransactionException {
        globalSession.setActive(false);
    }
```
监听器的处理还是非常简单的，只是将globalSession对象有激活状态改为非激活状态，改为非激活状态后，新分支事务就不能在注册了。

## 全局事务锁请求
全局事务锁状态查询请求的消息类型是`MessageType.TYPE_GLOBAL_LOCK_QUERY`，请求对象为`GlobalLockQueryRequest`，该请求由RM发起。该请求的作用是查询一条或多条记录是否上锁。
RM会将表名、需要查询是否上锁的主键值组装成一个字符串发送过来，TC解析字符串，每条记录创建一个RowLock对象，然后从`LOCK_MAP`中查看记录加锁情况，并将是否上锁的结果返回至RM。其中RM组装字符串的规则和分支事务注册请求中记录要加锁的`lockKey`组装规则是一致的，规则如下：
**表名:主键值_主键值;表名:主键值_主键值**

自然根据字符串创建RowLock对象的处理逻辑也是和分支事务注册请求的处理是一致的。
下面具体看一下代码逻辑，下面代码是`DefaultCoordinator.doLockCheck`方法：
```java
protected void doLockCheck(GlobalLockQueryRequest request, GlobalLockQueryResponse response, RpcContext rpcContext)
        throws TransactionException {
        response.setLockable(
            core.lockQuery(request.getBranchType(), request.getResourceId(), request.getXid(), request.getLockKey()));
    }
```
`doLockCheck`调用了`DefaultCore`的`lockQuery`方法：
```java
public boolean lockQuery(BranchType branchType, String resourceId, String xid, String lockKeys)
        throws TransactionException {
        //branchType=AT，因为现在使用的是AT模式，getCore(branchType)返回的对象是ATCore
        return getCore(branchType).lockQuery(branchType, resourceId, xid, lockKeys);
    }
    //下面代码是ATCore的lockQuery方法：
    public boolean lockQuery(BranchType branchType, String resourceId, String xid, String lockKeys)
            throws TransactionException {
        return lockManager.isLockable(xid, resourceId, lockKeys);
    }
```
`lockQuery`方法中调用了`LockManager`的方法，`LockManager`根据配置文件中配置的`store.mode`参数的不同，而使用不同的对象，这里配置的是`file`，`LockManager`的实现类是`FileLockManager`，store.mode还可以配置db，redis，至于他们之间的区别，后面文章在做介绍。
下面看一下`FileLockManager`的`isLockable`方法：
```java
//入参lockKey就是根据上面介绍的字符串组装规则形成的字符串，lockKey是请求报文里面的，可以参见DefaultCoordinator.doLockCheck方法
	//isLockable方法返回true表示没有加锁，返回false表示已经加锁
	public boolean isLockable(String xid, String resourceId, String lockKey) throws TransactionException {
        if (StringUtils.isBlank(lockKey)) {
            return true;
        }
        //collectRowLocks方法的处理过程与分支事务注册请求中是一样的，这里不再做介绍
        //可以参见[《Seata解析-TC处理全局事务和分支事务原理详解之全局事务开启和分支事务注册》](https://blog.csdn.net/weixin_38308374/article/details/108457173)
        List<RowLock> locks = collectRowLocks(lockKey, resourceId, xid);
        try {
        	//LockManager的getLocker()方法用于创建FileLocker对象
            return getLocker().isLockable(locks);
        } catch (Exception t) {
            LOGGER.error("isLockable error, xid:{} resourceId:{}, lockKey:{}", xid, resourceId, lockKey, t);
            return false;
        }
    }
```
`isLockable`方法最后是调用`FileLocker`的`isLockable`方法：
```java
public boolean isLockable(List<RowLock> rowLocks) {
        if (CollectionUtils.isEmpty(rowLocks)) {
            //no lock
            //没有加锁
            return true;
        }
        //全局事务ID
        Long transactionId = rowLocks.get(0).getTransactionId();
        //资源ID，也就是RM访问的数据库URL
        String resourceId = rowLocks.get(0).getResourceId();
        //LOCK_MAP里面存储了所有加锁的数据库数据
        ConcurrentMap<String, ConcurrentMap<Integer, BucketLockMap>> dbLockMap = LOCK_MAP.get(resourceId);
        if (dbLockMap == null) {
            //如果dbLockMap为null，表示尚未有事务对该数据库加锁
            return true;
        }
        //遍历每个数据库记录
        for (RowLock rowLock : rowLocks) {
            String xid = rowLock.getXid();
            String tableName = rowLock.getTableName();
            String pk = rowLock.getPk();//pk是主键值
            ConcurrentMap<Integer, BucketLockMap> tableLockMap = dbLockMap.get(tableName);
            if (tableLockMap == null) {
                //tableLockMap为null，表示该表中没有记录被加锁
                continue;
            }
            //根据主键值计算桶位
            int bucketId = pk.hashCode() % BUCKET_PER_TABLE;
            BucketLockMap bucketLockMap = tableLockMap.get(bucketId);
            if (bucketLockMap == null) {
                //bucketLockMap为null，表示该主键值中没有被加锁
                continue;
            }
            //根据主键值从bucketLockMap中取出已经对该主键值加锁的事务ID
            Long lockingTransactionId = bucketLockMap.get().get(pk);
            if (lockingTransactionId == null || lockingTransactionId.longValue() == transactionId) {
                // 如果lockingTransactionId为null，表示没有事务对该主键值加锁
                // 如果lockingTransactionId与当前事务ID相同，表示两个事务是同一个事务，可以认为是没有加锁的
                continue;
            } else {
                //如果主键值已经被加锁，则返回false
                LOGGER.info("Global lock on [" + tableName + ":" + pk + "] is holding by " + lockingTransactionId);
                return false;
            }
        }
        return true;
    }
```
## 全局事务状态查询请求
全局事务提交请求的消息类型是`MessageType.TYPE_GLOBAL_STATUS`，请求对象为`GlobalStatusRequest`，该请求由TM发起。该请求的作用是查询全局事务的状态，TC直接将事务状态返回。
处理全局事务状态查询请求是通过`DefaultCoordinator`的`doGlobalStatus`调用了`DefaultCore`的`getStatus`方法，下面看一下这个方法：
```java
public GlobalStatus getStatus(String xid) throws TransactionException {
		//第二个入参false表示不查询分支事务
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid, false);
        if (globalSession == null) {
        	//为null，表示当前事务不是本seata服务器管理的事务
            return GlobalStatus.Finished;
        } else {
            return globalSession.getStatus();
        }
    }
```
## 分支事务提交响应
TC会给各个分支事务发送消息通知分支事务提交（提交是在异步提交管理器中进行的）或者回滚，那这里就涉及到分支事务的响应问题，本文将分析分支事务的两个响应：分支事务提交结果响应和分支事务回滚结果响应。

这两个响应都是由`ServerOnResponseProcessor`处理器处理的。下面看一下该类的`process`方法：
```java
public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        MessageFuture messageFuture = futures.remove(rpcMessage.getId());
        //发送的消息都会创建MessageFuture对象并存储到futures中，
        //这样当响应消息到达时，可以通过futures找到MessageFuture对象
        //当执行messageFuture.setResultMessage设置返回值，则会触发发送请求的线程运行
        //MessageFuture下面会介绍
        if (messageFuture != null) {
        	//本文涉及的两个响应都会进入到该分支中，除非响应超时
            messageFuture.setResultMessage(rpcMessage.getBody());
        } else {
            if (ChannelManager.isRegistered(ctx.channel())) {
            	//onResponseMessage仅仅打印了日志，比较简单，不再做介绍
                onResponseMessage(ctx, rpcMessage);
            } else {
                //如果当前channel没有注册过，seata任务是非法连接，直接关闭该连接。
                try {
                    if (LOGGER.isInfoEnabled()) {
                        LOGGER.info("closeChannelHandlerContext channel:" + ctx.channel());
                    }
                    ctx.disconnect();
                    ctx.close();
                } catch (Exception exx) {
                    LOGGER.error(exx.getMessage());
                }
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info(String.format("close a unhandled connection! [%s]", ctx.channel().toString()));
                }
            }
        }
    }
```
当给分支事务发送提交或者回滚请求时，都会在`futures`属性中添加`MessageFuture`对象，下面是该对象包含的属性：
```java
private RpcMessage requestMessage;//请求报文
    private long timeout;//等待响应的超时时间
    private long start = System.currentTimeMillis();//请求的发送时间
    private transient CompletableFuture<Object> origin = new CompletableFuture<>();
```
这些属性中最重要的是origin属性，文章下面会看到该属性如何使用。
下面再来看一下请求发送逻辑（删除了部分代码）：
```java
protected Object sendSync(Channel channel, RpcMessage rpcMessage, long timeoutMillis) throws TimeoutException {
		//创建MessageFuture对象
        MessageFuture messageFuture = new MessageFuture();
        //rpcMessage是要发送的请求对象
        messageFuture.setRequestMessage(rpcMessage);
        //设置超时时间
        messageFuture.setTimeout(timeoutMillis);
        //将messageFuture保存到futures集合中
        futures.put(rpcMessage.getId(), messageFuture);
		//发送请求
        channel.writeAndFlush(rpcMessage).addListener((ChannelFutureListener) future -> {
            if (!future.isSuccess()) {
            	//如果请求发送失败，则从futures集合中删除MessageFuture，并且销毁channel通道
                MessageFuture messageFuture1 = futures.remove(rpcMessage.getId());
                if (messageFuture1 != null) {
                    messageFuture1.setResultMessage(future.cause());
                }
                destroyChannel(future.channel());
            }
        });
        try {
        	//从messageFuture中获取远程返回的数据，如果远程没有返回，这里会一直等待
            return messageFuture.get(timeoutMillis, TimeUnit.MILLISECONDS);
        } catch (Exception exx) {
            ...
        }
    }
```
上面的`messageFuture.get()`调用了下边的方法：
```java
try {
        result = origin.get(timeout, unit);
} catch (ExecutionException e) {
            ...
} catch (TimeoutException e) {
            ...
}
```

这里可以看到，请求发送成功后，会调用`CompletableFuture`的`get`方法，当响应没有返回时，线程会一直阻塞在`origin.get`位置。当远程返回了响应，会在`ServerOnResponseProcessor.process`方法里面设置：
```java
messageFuture.setResultMessage(rpcMessage.getBody());
```

`ServerOnResponseProcessor`处理分支事务提交结果响应和分支事务回滚结果响应时，并没有特殊的处理，仅仅是设置了响应结果，然后触发发送请求的线程由阻塞转为运行状态。