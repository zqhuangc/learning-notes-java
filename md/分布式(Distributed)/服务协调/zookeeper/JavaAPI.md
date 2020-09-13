# Zookeeper 的编程

## 三种方式区别

Zookeeper客户端提供了基本的操作，比如，创建会话、创建节点、读取节点、更新数据、删除节点和检查节点是否存在等。但对于开发人员来说，Zookeeper提供的基本操纵还是有一些不足之处。本篇博客就聊聊这些不足之处和两款开源框架ZKClient和Curator。

* Zookeeper API不足之处

（1）Zookeeper的Watcher是一次性的，每次触发之后都需要重新进行注册； 
（2）Session超时之后没有实现重连机制； 
（3）异常处理繁琐，Zookeeper提供了很多异常，对于开发人员来说可能根本不知道该如何处理这些异常信息； 
（4）只提供了简单的byte[]数组的接口，没有提供针对对象级别的序列化； 
（5）创建节点时如果节点存在抛出异常，需要自行检查节点是否存在； 
（6）删除节点无法实现级联删除；

* ZkClient简介

ZkClient是一个开源客户端，在Zookeeper原生API接口的基础上进行了包装，更便于开发人员使用。内部实现了Session超时重连，Watcher反复注册等功能。像dubbo等框架对其也进行了集成使用。

虽然ZkClient对原生API进行了封装，但也有它自身的不足之处：

几乎没有参考文档；
异常处理简化（抛出RuntimeException）；
重试机制比较难用；
没有提供各种使用场景的实现；

* Curator简介

Curator是Netflix公司开源的一套Zookeeper客户端框架，和ZkClient一样，解决了非常底层的细节开发工作，包括连接重连、反复注册Watcher和NodeExistsException异常等。目前已经成为Apache的顶级项目。另外还提供了一套易用性和可读性更强的Fluent风格的客户端API框架。

除此之外，Curator中还提供了Zookeeper各种应用场景（Recipe，如共享锁服务、Master选举机制和分布式计算器等）的抽象封装。

## Java 原生 API

#### Operation

* WatchedEvent
  - Event.KeeperState
  - Event.EventType
* ZooDefs.Ids ACL
* CreateMode 节点类型

```java
        /*ZooKeeper zookeeper = new ZooKeeper(CONNECTSTRING, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                
            }
        });*/
        // 连接无法保证实时
        countDownLatch.await();

        // 访问控制
        ACL acl=new ACL(ZooDefs.Perms.ALL,new Id("ip","192.168.11.129"));
        List<ACL> acls=new ArrayList<>();
        acls.add(acl);
        zooKeeper.create("/authTest","111".getBytes(),acls,CreateMode.PERSISTENT);
        zooKeeper.getData("/authTest",true,new Stat());

        //创建节点
        String result = zooKeeper.create("/test1", "test".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        //增加一个
        zooKeeper.getData("/test1", new WatcherApi(), stat);
        System.out.println("创建成功："+result);

        //修改数据
        zooKeeper.setData("/test1","testUpdate".getBytes(),-1);
        Thread.sleep(2000);
        byte[] data = zooKeeper.getData("/test1", new WatcherApi(), stat);
        System.out.println("数据为："+ new String(data));

        //修改数据
        zooKeeper.setData("/test1","testUpdate2".getBytes(),-1);
        Thread.sleep(2000);

        //删除节点
        zooKeeper.delete("/test1",-1);
        Thread.sleep(2000);

        //创建节点和子节点
        String path="/test11";

        zooKeeper.create(path,"123".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
        TimeUnit.SECONDS.sleep(1);

        Stat stat= zooKeeper.exists(path+"/test",true);
        if(stat==null){//表示节点不存在
            zooKeeper.create(path+"/test","123".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,CreateMode.PERSISTENT);
            TimeUnit.SECONDS.sleep(1);
        }
        //修改子路径
        zooKeeper.setData(path+"/test","test123".getBytes(),-1);
        TimeUnit.SECONDS.sleep(1);
        //获取指定节点下的子节点
        List<String> childrens= zooKeeper.getChildren("/test",true);
        System.out.println(childrens);
```

#### Watcher and WatchedEvent

```java
        @Override
        public void process(WatchedEvent event) {
            //如果当前的连接状态是连接成功的，那么通过计数器去控制
            if(event.getState() == Event.KeeperState.SyncConnected){
                if(Event.EventType.None == event.getType()){
                    countDownLatch.countDown();
                    System.out.println(event.getState() + "-->" + event.getType());
                } else if(event.getType() == Event.EventType.NodeDataChanged){
                    try {
                        System.out.println("数据变更触发路径："+ event.getPath()+"->改变后的值：" +
                                zooKeeper.getData(event.getPath(),true,stat));
                    } catch (KeeperException e) {
                        e.printStackTrace();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else if(event.getType() == Event.EventType.NodeChildrenChanged){
                    //子节点的数据变化会触发
                    try {
                        System.out.println("子节点数据变更路径："+ event.getPath()+"->节点的值："+
                                zooKeeper.getData(event.getPath(),true,stat));
                    } catch (KeeperException e) {
                        e.printStackTrace();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                } else if(event.getType() == Event.EventType.NodeCreated){
                    //创建子节点的时候会触发
                    try {
                        System.out.println("节点创建路径："+event.getPath()+"->节点的值："+
                                zooKeeper.getData(event.getPath(),true,stat));
                    } catch (KeeperException e) {
                        e.printStackTrace();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }else if(event.getType()== Event.EventType.NodeDeleted){
                    //子节点删除会触发
                    System.out.println("节点删除路径："+ event.getPath());
                }
                System.out.println(event.getType());
            }
        }
```



## ZkClient

### 依赖

```xml
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>RELEASE</version>
        </dependency>
```



### 应用

```java
        ZkClient zkClient=new ZkClient(CONNECTSTRING,4000);

        //zkclient createPersistent 提供递归创建父节点的功能，临时节点暂无此动能
        // 递归判断“/”截取
        zkClient.createPersistent("/zkclient/zkclient1/zkclient1-1/zkclient1-1-1",true);
        System.out.println("success");

        //删除节点
        zkClient.deleteRecursive("/zkclient");


        //获取子节点
        List<String> list = zkClient.getChildren("/node");
        System.out.println(list);

        //watcher
        zkClient.subscribeDataChanges("/node", new IZkDataListener() {
            @Override
            public void handleDataChange(String s, Object o) throws Exception {
                System.out.println("节点名称："+s+"->节点修改后的值"+o);
            }

            @Override
            public void handleDataDeleted(String s) throws Exception {

            }
        });
        // 设置值
        zkClient.writeData("/node","node");
        TimeUnit.SECONDS.sleep(2);

        zkClient.subscribeChildChanges("/node", new IZkChildListener() {
            @Override
            public void handleChildChange(String s, List<String> list) throws Exception {

            }
        });
```





## [Curator](http://curator.apache.org/getting-started.html)（丰富）

### [官方例子](https://github.com/apache/curator/tree/master/curator-examples/src/main/java)

可参考官方提供实现，以及如何操作的

### 依赖

```xml

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>RELEASE</version>
        </dependency>
       <!-- zookeeepr的功能实现 -->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>RELEASE</version>
        </dependency>
```

### 应用

#### 创建连接

```java
public class CuratorCreateSessionDemo {
    private final static String CONNECTSTRING="192.168.11.129:2181,192.168.11.134:2181," +
            "192.168.11.135:2181,192.168.11.136:2181";
    public static void main(String[] args) {
        //创建会话的两种方式 normal
        CuratorFramework curatorFramework= CuratorFrameworkFactory.
                newClient(CONNECTSTRING,5000,5000,
                        new ExponentialBackoffRetry(1000,3));
        curatorFramework.start(); //start方法启动连接

        //fluent风格
        CuratorFramework curatorFramework1=CuratorFrameworkFactory.builder().connectString(CONNECTSTRING).sessionTimeoutMs(5000).
                retryPolicy(new ExponentialBackoffRetry(1000,3)).
                namespace("/curator").build();

        curatorFramework1.start();
        System.out.println("success");
    }
}



```

#### 事件（CuratorEvent）

```java
public class CuratorEventDemo {

    /**
     * 三种watcher来做节点的监听
     * pathcache   监视一个路径下子节点的创建、删除、节点数据更新
     * NodeCache   监视一个节点的创建、更新、删除
     * TreeCache   pathcaceh+nodecache 的合体（监视路径下的创建、更新、删除事件），
     * 缓存路径下的所有子节点的数据
     */

    public static void main(String[] args) throws Exception {
        CuratorFramework curatorFramework=CuratorClientUtils.getInstance();

        /**
         * 节点变化NodeCache
         */
       /* NodeCache cache=new NodeCache(curatorFramework,"/curator",false);
        cache.start(true);

        cache.getListenable().addListener(()-> System.out.println("节点数据发生变化,变化后的结果" +
                "："+new String(cache.getCurrentData().getData())));

        curatorFramework.setData().forPath("/curator","菲菲".getBytes());*/


        /**
         * PatchChildrenCache
         */

        PathChildrenCache cache=new PathChildrenCache(curatorFramework,"/event",true);
        cache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);
        // Normal / BUILD_INITIAL_CACHE /POST_INITIALIZED_EVENT

        cache.getListenable().addListener((curatorFramework1,pathChildrenCacheEvent)->{
            switch (pathChildrenCacheEvent.getType()){
                case CHILD_ADDED:
                    System.out.println("增加子节点");
                    break;
                case CHILD_REMOVED:
                    System.out.println("删除子节点");
                    break;
                case CHILD_UPDATED:
                    System.out.println("更新子节点");
                    break;
                default:break;
            }
        });

        curatorFramework.create().withMode(CreateMode.PERSISTENT).forPath("/event","event".getBytes());
        TimeUnit.SECONDS.sleep(1);
        System.out.println("1");
        curatorFramework.create().withMode(CreateMode.EPHEMERAL).forPath("/event/event1","1".getBytes());
        TimeUnit.SECONDS.sleep(1);
        System.out.println("2");

        curatorFramework.setData().forPath("/event/event1","222".getBytes());
        TimeUnit.SECONDS.sleep(1);
        System.out.println("3");

        curatorFramework.delete().forPath("/event/event1");
        System.out.println("4");

        System.in.read();

    }
}
```

#### 各项操作（CRUD/Async/Transaction）

```java
public class CuratorOperatorDemo {

    public static void main(String[] args) throws InterruptedException {
        CuratorFramework curatorFramework=CuratorClientUtils.getInstance();
        System.out.println("连接成功.........");

        //fluent风格

        /**
         * 创建节点
         */

       /* try {
            String result=curatorFramework.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).
                    forPath("/curator/curator1/curator11","123".getBytes());

            System.out.println(result);
        } catch (Exception e) {
            e.printStackTrace();
        }*/

        /**
         * 删除节点
         */
        /*try {
            //默认情况下，version为-1
            curatorFramework.delete().deletingChildrenIfNeeded().forPath("/node11");

        } catch (Exception e) {
            e.printStackTrace();
        }*/

        /**
         * 查询
         */
        /*Stat stat=new Stat();
        try {
            byte[] bytes=curatorFramework.getData().storingStatIn(stat).forPath("/curator");
            System.out.println(new String(bytes)+"-->stat:"+stat);
        } catch (Exception e) {
            e.printStackTrace();
        }*/

        /**
         * 更新
         */

       /* try {
            Stat stat=curatorFramework.setData().forPath("/curator","123".getBytes());
            System.out.println(stat);
        } catch (Exception e) {
            e.printStackTrace();
        }*/


        /**
         * 异步操作
         */
        /*ExecutorService service= Executors.newFixedThreadPool(1);
        CountDownLatch countDownLatch=new CountDownLatch(1);
        try {
            curatorFramework.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).
                    inBackground(new BackgroundCallback() {
                        @Override
                        public void processResult(CuratorFramework curatorFramework, CuratorEvent curatorEvent) throws Exception {
                            System.out.println(Thread.currentThread().getName()+"->resultCode:"+curatorEvent.getResultCode()+"->"
                            +curatorEvent.getType());
                            countDownLatch.countDown();
                        }
                    },service).forPath("/mic","123".getBytes());
        } catch (Exception e) {
            e.printStackTrace();
        }
        countDownLatch.await();
        service.shutdown();*/

        /**
         * 事务操作（curator独有的）
         */

        try {
            Collection<CuratorTransactionResult> resultCollections=curatorFramework.inTransaction().create().forPath("/trans","111".getBytes()).and().
                    setData().forPath("/curator","111".getBytes()).and().commit();
            for (CuratorTransactionResult result:resultCollections){
                System.out.println(result.getForPath()+"->"+result.getType());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 选举（内有封装实现）

```java
public class ExampleClient extends LeaderSelectorListenerAdapter implements Closeable{

    private final String name;
    private final LeaderSelector leaderSelector;
    private final AtomicInteger leaderCount=new AtomicInteger();

    public ExampleClient(CuratorFramework client,String path,String name) {
        this.name = name;
        this.leaderSelector = new LeaderSelector(client,path,this);
        leaderSelector.autoRequeue(); //自动抢
    }

    public void start(){
        leaderSelector.start();
    }

    @Override
    public void close() throws IOException {
        leaderSelector.close();
    }
   
    @Override
    public void takeLeadership(CuratorFramework client) throws Exception {
        final int waitSeconds=new Random().nextInt(50);
        System.out.println(name+"->我现在是leader，等待时间："+waitSeconds+", 抢到领导的次数:"+leaderCount.getAndIncrement());

        TimeUnit.SECONDS.toMillis(1000);
    }
}


public class TreeCacheDemo {
    private final static String CONNECTSTRING="192.168.11.129:2181,192.168.11.134:2181," +
            "192.168.11.135:2181,192.168.11.136:2181";

    private static final String MASTER_PATH="/curator_master_path1";

    private static final int CLIENT_QTY=10; //客户端数量

    public static void main(String[] args) throws Exception {
        System.out.println("创建"+CLIENT_QTY+"个客户端，");
        List<CuratorFramework> clients = Lists.newArrayList();
        List<ExampleClient>     examples = Lists.newArrayList();
        try {
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client = CuratorFrameworkFactory.newClient(CONNECTSTRING,
                        new ExponentialBackoffRetry(1000, 3));
                clients.add(client);
                ExampleClient exampleClient = new ExampleClient(client, MASTER_PATH, "Client:" + i);
                examples.add(exampleClient);
                client.start();
                exampleClient.start();
            }
            System.in.read();
        }finally {
            for ( ExampleClient exampleClient : examples ){
                CloseableUtils.closeQuietly(exampleClient);
            }
            for ( CuratorFramework client : clients ){
                CloseableUtils.closeQuietly(client);
            }
        }
    }
}
```



## 选举

### Curator 

#### 核心类/接口

* LeaderSelector

* LeaderSelectorListenerAdapter（因接口不会抛异常，抽象实现来抛出异常）

#### 实现

```java
public class MasterSelector {

    private final static String CONNECTSTRING="192.168.11.129:2181,192.168.11.134:2181," +
            "192.168.11.135:2181,192.168.11.136:2181";

    private final static String MASTER_PATH="/curator_master_path";
    public static void main(String[] args) {

        CuratorFramework curatorFramework= CuratorFrameworkFactory.builder().connectString(CONNECTSTRING).
                retryPolicy(new ExponentialBackoffRetry(1000,3)).build();

        LeaderSelector leaderSelector=new LeaderSelector(curatorFramework, MASTER_PATH, new LeaderSelectorListenerAdapter() {
            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                System.out.println("获得leader成功");
                TimeUnit.SECONDS.sleep(2);
            }
        });
        leaderSelector.autoRequeue();
        leaderSelector.start();//开始选举
    }
}
```

### ZkClient 实现

```java
public class MasterChooseTest {

    private final static String CONNECTSTRING="192.168.11.129:2181,192.168.11.134:2181," +
            "192.168.11.135:2181,192.168.11.136:2181";


    public static void main(String[] args) throws IOException {
        List<MasterSelector> selectorLists=new ArrayList<>();
        try {
            for(int i=0;i<10;i++) {
                ZkClient zkClient = new ZkClient(CONNECTSTRING, 5000,
                        5000,
                        new SerializableSerializer());
                UserCenter userCenter = new UserCenter();
                userCenter.setMc_id(i);
                userCenter.setMc_name("客户端：" + i);

                MasterSelector selector = new MasterSelector(userCenter,zkClient);
                selectorLists.add(selector);
                selector.start();//触发选举操作
                TimeUnit.SECONDS.sleep(1);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            for(MasterSelector selector:selectorLists){
                selector.stop();
            }
        }
    }
}


public class MasterSelector {

    private ZkClient zkClient;

    private final static String MASTER_PATH="/master"; //需要争抢的节点

    private IZkDataListener dataListener; //注册节点内容变化

    private UserCenter server;  //其他服务器

    private UserCenter master;  //master节点

    private boolean isRunning=false;

    ScheduledExecutorService scheduledExecutorService= Executors.newScheduledThreadPool(1);

    public MasterSelector(UserCenter server,ZkClient zkClient) {
        System.out.println("["+server+"] 去争抢master权限");
        this.server = server;
        this.zkClient=zkClient;

        this.dataListener= new IZkDataListener() {
            @Override
            public void handleDataChange(String s, Object o) throws Exception {

            }

            @Override
            public void handleDataDeleted(String s) throws Exception {
                //节点如果被删除, 发起选主操作
                chooseMaster();
            }
        };
    }

    public void start(){
        //开始选举
        if(!isRunning){
            isRunning=true;
            zkClient.subscribeDataChanges(MASTER_PATH,dataListener); //注册节点事件
            chooseMaster();
        }
    }


    public void stop(){
        //停止
        if(isRunning){
            isRunning=false;
            scheduledExecutorService.shutdown();
            zkClient.unsubscribeDataChanges(MASTER_PATH,dataListener);
            releaseMaster();
        }
    }


    //具体选master的实现逻辑
    private void chooseMaster(){
        if(!isRunning){
            System.out.println("当前服务没有启动");
            return ;
        }
        try {
            zkClient.createEphemeral(MASTER_PATH, server);
            master=server; //把server节点赋值给master
            System.out.println(master+"->我现在已经是master，你们要听我的");

            //定时器
            //master释放(master 出现故障）,没5秒钟释放一次
            scheduledExecutorService.schedule(()->{
                releaseMaster();//释放锁
            },2, TimeUnit.SECONDS);
        }catch (ZkNodeExistsException e){
            //表示master已经存在
            UserCenter userCenter=zkClient.readData(MASTER_PATH,true);
            if(userCenter==null) {
                System.out.println("启动操作：");
                chooseMaster(); //再次获取master
            }else{
                master=userCenter;
            }
        }
    }

    private void releaseMaster(){
        //释放锁(故障模拟过程)
        //判断当前是不是master，只有master才需要释放
        if(checkIsMaster()){
            zkClient.delete(MASTER_PATH); //删除
        }
    }


    private boolean checkIsMaster(){
        //判断当前的server是不是master
        UserCenter userCenter=zkClient.readData(MASTER_PATH);
        if(userCenter.getMc_name().equals(server.getMc_name())){
            master=userCenter;
            return true;
        }
        return false;
    }

}


public class UserCenter implements Serializable{

    private static final long serialVersionUID = -1776114173857775665L;
    private int mc_id; //机器信息

    private String mc_name;//机器名称

    public int getMc_id() {
        return mc_id;
    }

    public void setMc_id(int mc_id) {
        this.mc_id = mc_id;
    }

    public String getMc_name() {
        return mc_name;
    }

    public void setMc_name(String mc_name) {
        this.mc_name = mc_name;
    }

    @Override
    public String toString() {
        return "UserCenter{" +
                "mc_id=" + mc_id +
                ", mc_name='" + mc_name + '\'' +
                '}';
    }
}


```





## 分布式锁

### 原理

利用 Zookeeper 中节点路径的唯一性，通过竞争创建相同路径节点，成功即获取锁，删除节点则为释放锁，由 通过第三方的 Zookeeper ，可以实现分布式锁

### 原生 API 实现

```java
public class DistributeLock {


    private static final String ROOT_LOCKS="/LOCKS";//根节点

    private ZooKeeper zooKeeper;

    private int sessionTimeout; //会话超时时间

    private String lockID; //记录锁节点id

    private final static byte[] data={1,2}; //节点的数据

    private CountDownLatch countDownLatch=new CountDownLatch(1);

    public DistributeLock() throws IOException, InterruptedException {
        this.zooKeeper=ZookeeperClient.getInstance();
        this.sessionTimeout=ZookeeperClient.getSessionTimeout();
    }

    //获取锁的方法
    public boolean lock(){
        try {
            //LOCKS/00000001
            lockID=zooKeeper.create(ROOT_LOCKS+"/",data, ZooDefs.Ids.
                    OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

            System.out.println(Thread.currentThread().getName()+"->成功创建了lock节点["+lockID+"], 开始去竞争锁");

            List<String> childrenNodes=zooKeeper.getChildren(ROOT_LOCKS,true);//获取根节点下的所有子节点
            //排序，从小到大
            SortedSet<String> sortedSet=new TreeSet<String>();
            for(String children:childrenNodes){
                sortedSet.add(ROOT_LOCKS+"/"+children);
            }
            String first=sortedSet.first(); //拿到最小的节点
            if(lockID.equals(first)){
                //表示当前就是最小的节点
                System.out.println(Thread.currentThread().getName()+"->成功获得锁，lock节点为:["+lockID+"]");
                return true;
            }
            SortedSet<String> lessThanLockId=sortedSet.headSet(lockID);
            if(!lessThanLockId.isEmpty()){
                String prevLockID=lessThanLockId.last();//拿到比当前LOCKID这个几点更小的上一个节点
                zooKeeper.exists(prevLockID,new LockWatcher(countDownLatch));
                countDownLatch.await(sessionTimeout, TimeUnit.MILLISECONDS);
                //上面这段代码意味着如果会话超时或者节点被删除（释放）了
                System.out.println(Thread.currentThread().getName()+" 成功获取锁：["+lockID+"]");
            }
            return true;
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    public boolean unlock(){
        System.out.println(Thread.currentThread().getName()+"->开始释放锁:["+lockID+"]");
        try {
            zooKeeper.delete(lockID,-1);
            System.out.println("节点["+lockID+"]成功被删除");
            return true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
        return false;
    }


    public static void main(String[] args) {
        final CountDownLatch latch=new CountDownLatch(10);
        Random random=new Random();
        for(int i=0;i<10;i++){
            new Thread(()->{
                DistributeLock lock=null;
                try {
                    lock=new DistributeLock();
                    latch.countDown();
                    latch.await();
                    lock.lock();
                    Thread.sleep(random.nextInt(500));
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    if(lock!=null){
                        lock.unlock();
                    }
                }
            }).start();
        }
    }
}

```



```java
public class LockWatcher implements Watcher{

    private CountDownLatch latch;

    public LockWatcher(CountDownLatch latch) {
        this.latch = latch;
    }

    public void process(WatchedEvent event) {
        if(event.getType()== Event.EventType.NodeDeleted){
            latch.countDown();
        }
    }
}
```



```java
public class ZookeeperClient {

    private final static String CONNECTSTRING="192.168.11.129:2181," + "192.168.11.134:2181," + "192.168.11.135:2181," + "192.168.11.136:2181";

    private static int sessionTimeout=5000;

    //获取连接
    public static ZooKeeper getInstance() throws IOException, InterruptedException {
        final CountDownLatch conectStatus=new CountDownLatch(1);
        ZooKeeper zooKeeper=new ZooKeeper(CONNECTSTRING, sessionTimeout, new Watcher() {
            public void process(WatchedEvent event) {
                if(event.getState()== Event.KeeperState.SyncConnected){
                    conectStatus.countDown();
                }
            }
        });
        conectStatus.await();
        return zooKeeper;
    }

    public static int getSessionTimeout() {
        return sessionTimeout;
    }
}

```



### Curator （参考官方例子）

* InterProcessMutex

