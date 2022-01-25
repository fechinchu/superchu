# Zookeeper01

# 1.Zookeeper简介

Apache Zookeeper是一种用于分布式应用程序的**高性能协调服务**,提供一种集中式信息存储服务;

特点:数据存在内存中,类型文件系统的树形结构(文件和目录),高吞吐和低延迟,集群高可靠;

作用:基于Zookeeper可以实现分布式统一配置中心,服务注册中心,分布式锁等; 

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210401150611173.png" alt="image-20210401150611173" style="zoom: 40%;" />

Zookeeper的应用案例

* Hbase:使用Zookeeper进行Master选举,服务间协调;
* Solr:使用Zookeeper进行集群管理,Leader选举,配置管理;
* Dubbo:使用Zookeeper服务注册与发现;
* MyCat:集群管理,配置管理;

# 2.Zookeeper核心概念

## 2.1.session

* 一个客户端连接一个会话,由zk分配唯一sessionId;
* 客户端以特定的时间间隔发送心跳以保持会话有效;
* 超过会话超时时间未收到客户端的心跳,则判定客户端死了;
* 会话中的请求按FIFO顺序执行;

![image-20210402142022504](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402142022504.png)

## 2.2.数据模型

* 层次名称空间
  * 类似unit文件系统,以'/'为根;
  * 节点可以包含与之关联的数据以及子节点(既是文件也是文件夹);
  * 节点的路径总是表示为规范的,绝对的,斜杠分隔的路径;
* znode
  * 名称唯一,命名规范;
  * 节点类型:持久,顺序,临时,临时顺序;
  * 节点数据构成;

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402142452026.png" alt="image-20210402142452026" style="zoom: 90%;" />

### 2.2.1.znode节点类型

* 持久节点`create /app1 xxx`;
* 临时节点 `create -e /app2 xxx`,当会话关闭,临时节点就会被删除;
* 顺序节点`create -s /app1/a xxx`,`create -s /app1/ xxx`;分别得到两个节点`a0000000000`,`00000000001`;
* 临时顺序节点:`create -s -e /app1/xxx`

### 2.2.2.znode数据构成;

* 节点元数据:stat结构;
* 节点数据:储存的协调数据;

![image-20210402145728017](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402145728017.png)

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402150301081.png" alt="image-20210402150301081" style="zoom: 33%;" />

注意:其中的aclVersion是访问控制列表的版本;

Zookeeper uses ACLs to control access to its znode;

![image-20210402151752905](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402151752905.png)

## 2.3.Zookeeper中的时间

Zookeeper中有多种方式跟踪时间;

* Zxid:Zookeeper中的每次更改操作都对应一个唯一的事务id,称为Zxid,它是一个全局有序的戳记,如果zxid1小于zxid2,那么zxid1发生在zxid2之前;
* Version numbers:版本号,对节点的每次更改都会导致该节点的版本号之一增加;
* Ticks:当使用多服务器Zookeeper时,服务器使用'滴答'来定义事件的时间,如状态上传,会话超时,对等点之间的连接超时等;
* Real time:Zookeeper除了在znode创建和修改时将时间戳放入stat结构之外,根本不使用Real time或时钟时间;

## 2.4.Watch监听机制

客户端可以在znodes上设置watch,监听znode的变化;

当我们使用`-w`来开启app1 的watch监听机制的时候,我们修改app1的数据.就可以监听到watch;

![image-20210402154820441](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210402154820441.png)

两类watch:

* data watch监听数据变更;
* child watch监听子节点变化;

触发watch事件(getData,getChildren,exists)

* created event:
  * enabled with a call to exists;
* deleted event:
  * enabled with a call to exists,getData,and getChildren;
* changed event:
  * enabled with a call to exists and getData;
* child event:
  * enabled with a call to getChildren;

watch重要特性:

* 一次性触发:watch触发后即被删除,要持续监控变化,则需要持续设置watch;
* 有序性:客户端先得到watch通知,后才会看到变化结果;

watch注意事项:

* watch是一次性触发器;如果您获得了一个watch事件,并且希望得到关于未来更改的通知,则必须设置另一个watch;
* 因为watch是一次性触发器,并且在获取事件和发送获取watch的新请求之间存在延时,所以不能可靠地得到节点发生的每个更改;
* 一个watch对象只会被特定的通知触发一次.如果一个watch对象同时注册了exists,getData,当节点被删除时,删除事件对exists,getData都有效,但只会调用watch一次;

## 2.5.Zookeeper特性

1. 顺序一致性(Sequential Consistency):保证客户端操作是按顺序生效的;
2. 原子性(Atomicity):更新成功或失败,没有部分结果;
3. 单个系统印象:无论连接到哪个服务器,客户端都将看到相同的内容;
4. 可靠性:数据的变更不会丢失,除非被客户端覆盖修改;
5. 及时性:保证系统的客户端当时读取到的数据是最新的;

# 3.Zookeeper应用场景

## 3.1.Zookeeper典型应用场景

* 数据发布订阅(配置中心);
* 命名服务;
* Master选举;
* 集群管理;
* 分布式队列;
* 分布式锁;

## 3.2.Zookeeper实现配置中心

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720234310847.png" alt="image-20210720234310847" style="zoom: 20%;" />

  通过zookeeper的znode能够存储数据和watcher监听机制来实现分布式配置中心;

## 3.3.Zookeeper实现命名服务

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210405182837501.png" alt="image-20210405182837501" style="zoom: 33%;" />

## 3.4.Zookeeper实现Master选举

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210405183324616.png" alt="image-20210405183324616" style="zoom: 33%;" />

~~~java
public class MasterElectionDemo {

	static class Server {

		private String cluster, name, address;

		private final String path, value;

		private String master;

		public Server(String cluster, String name, String address) {
			super();
			this.cluster = cluster;
			this.name = name;
			this.address = address;
			path = "/" + this.cluster + "Master";
			value = "name:" + name + " address:" + address;

			ZkClient client = new ZkClient("localhost:2181");
			client.setZkSerializer(new MyZkSerializer());

			//该线程,要改为守护线程
			new Thread(new Runnable() {

				@Override
				public void run() {
					electionMaster(client);
				}

			}).start();
		}

		public void electionMaster(ZkClient client) {
			try {
				client.createEphemeral(path, value);
				master = client.readData(path);
				System.out.println(value + "创建节点成功，成为Master");
			} catch (ZkNodeExistsException e) {
				master = client.readData(path);
				System.out.println("Master为：" + master);
			}

			// 为阻塞自己等待而用
			CountDownLatch cdl = new CountDownLatch(1);

			// 注册watcher
			IZkDataListener listener = new IZkDataListener() {

				@Override
				public void handleDataDeleted(String dataPath) throws Exception {
					System.out.println("-----监听到节点被删除");
					cdl.countDown();
				}

				@Override
				public void handleDataChange(String dataPath, Object data) throws Exception {

				}
			};

			client.subscribeDataChanges(path, listener);

			// 让自己阻塞
			if (client.exists(path)) {
				try {
					cdl.await();
				} catch (InterruptedException e1) {
					e1.printStackTrace();
				}
			}
			// 醒来后，取消watcher
			client.unsubscribeDataChanges(path, listener);
			// 递归调自己（下一次选举）
			electionMaster(client);

		}
	}
}
~~~



## 3.5.Zookeeper实现分布式队列

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210405185108317.png" alt="image-20210405185108317" style="zoom:33%;" />

## 3.6.Zookeeper实现分布式锁

### 3.6.1.分布式锁的实现方式一

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210405192341844.png" alt="image-20210405192341844" style="zoom:33%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210405192441451.png" alt="image-20210405192441451" style="zoom:33%;" />

缺点:惊群效应;

疑惑:为什么不通过zookeeper的watch机制进行阻塞等待,后来发现注册节点的watcher并不会阻塞;

~~~java
public class ZKDistributeLock implements Lock {

	private String lockPath;

	private ZkClient client;

	// 锁重入计数
	private ThreadLocal<Integer> reentrantCount = new ThreadLocal<>();

	public ZKDistributeLock(String lockPath) {
		super();
		this.lockPath = lockPath;

		client = new ZkClient("47.117.141.206:2181");
		client.setZkSerializer(new MyZkSerializer());
	}

	@Override
	public boolean tryLock() { // 不会阻塞

		if (this.reentrantCount.get() != null) {
			int count = this.reentrantCount.get();
			if (count > 0) {
				this.reentrantCount.set(++count);
				return true;
			}
		}
		// 创建节点
		try {
			client.createEphemeral(lockPath);
			this.reentrantCount.set(1);
		} catch (ZkNodeExistsException e) {
			return false;
		}
		return true;
	}

	@Override
	public void unlock() {
		// 重入的释放锁处理
		if (this.reentrantCount.get() != null) {
			int count = this.reentrantCount.get();
			if (count > 1) {
				this.reentrantCount.set(--count);
				return;
			} else {
				this.reentrantCount.set(null);
			}
		}
		client.delete(lockPath);
	}

	@Override
	public void lock() { // 如果获取不到锁，阻塞等待
		if (!tryLock()) {
			// 没获得锁，阻塞自己
			waitForLock();
			// 再次尝试
			lock();
		}

	}

	private void waitForLock() {

		CountDownLatch cdl = new CountDownLatch(1);

		IZkDataListener listener = new IZkDataListener() {

			@Override
			public void handleDataDeleted(String dataPath) throws Exception {
				System.out.println("----收到节点被删除了-------------");
				cdl.countDown();
			}

			@Override
			public void handleDataChange(String dataPath, Object data) throws Exception {
			}
		};

		client.subscribeDataChanges(lockPath, listener);

		// 阻塞自己
		if (this.client.exists(lockPath)) {
			try {
				cdl.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		// 取消注册
		client.unsubscribeDataChanges(lockPath, listener);
	}
}
~~~

### 3.6.2.分布式锁的实现方式二

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210406111116854.png" alt="image-20210406111116854" style="zoom:33%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20220125101135058.png" alt="image-20220125101135058" style="zoom: 33%;" /> 

~~~java
public class ZKDistributeImproveLock implements Lock {

	/*
	 * 利用临时顺序节点来实现分布式锁
	 * 获取锁：取排队号（创建自己的临时顺序节点），然后判断自己是否是最小号，如是，则获得锁；不是，则注册前一节点的watcher,阻塞等待
	 * 释放锁：删除自己创建的临时顺序节点
	 */
	private String lockPath;

	private ZkClient client;

	private ThreadLocal<String> currentPath = new ThreadLocal<>();

	private ThreadLocal<String> beforePath = new ThreadLocal<>();

	// 锁重入计数
	private ThreadLocal<Integer> reentrantCount = new ThreadLocal<>();

	public ZKDistributeImproveLock(String lockPath) {
		super();
		this.lockPath = lockPath;
		client = new ZkClient("47.117.141.206:2181");
		client.setZkSerializer(new MyZkSerializer());
		if (!this.client.exists(lockPath)) {
			try {
				this.client.createPersistent(lockPath);
			} catch (ZkNodeExistsException e) {

			}
		}
	}

	@Override
	public boolean tryLock() {
		if (this.reentrantCount.get() != null) {
			int count = this.reentrantCount.get();
			if (count > 0) {
				this.reentrantCount.set(++count);
				return true;
			}
		}

		if (this.currentPath.get() == null) {
			currentPath.set(this.client.createEphemeralSequential(lockPath + "/", "aaa"));
		}
		// 获得所有的子
		List<String> children = this.client.getChildren(lockPath);

		// 排序list
		Collections.sort(children);

		// 判断当前节点是否是最小的
		if (currentPath.get().equals(lockPath + "/" + children.get(0))) {
			this.reentrantCount.set(1);
			return true;
		} else {
			// 取到前一个
			// 得到字节的索引号
			int curIndex = children.indexOf(currentPath.get().substring(lockPath.length() + 1));
			beforePath.set(lockPath + "/" + children.get(curIndex - 1));
		}
		return false;
	}

	@Override
	public void lock() {
		if (!tryLock()) {
			// 阻塞等待
			waitForLock();
			// 再次尝试加锁
			lock();
		}
	}

	private void waitForLock() {

		CountDownLatch cdl = new CountDownLatch(1);

		// 注册watcher
		IZkDataListener listener = new IZkDataListener() {

			@Override
			public void handleDataDeleted(String dataPath) throws Exception {
				System.out.println("-----监听到节点被删除");
				cdl.countDown();
			}

			@Override
			public void handleDataChange(String dataPath, Object data) throws Exception {

			}
		};

		client.subscribeDataChanges(this.beforePath.get(), listener);

		// 怎么让自己阻塞
		if (this.client.exists(this.beforePath.get())) {
			try {
				cdl.await();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
		// 醒来后，取消watcher
		client.unsubscribeDataChanges(this.beforePath.get(), listener);
	}

	@Override
	public void unlock() {
		// 重入的释放锁处理
		if (this.reentrantCount.get() != null) {
			int count = this.reentrantCount.get();
			if (count > 1) {
				this.reentrantCount.set(--count);
				return;
			} else {
				this.reentrantCount.set(null);
			}
		}
		// 删除节点
		this.client.delete(this.currentPath.get());
	}
}
~~~

### 3.6.3.Redis实现分布式锁

* 普通的redis分布式锁
  * 加锁:`SET my:lock <随机值> NX PX 3000`,
  * 释放锁就是删除key,但是一般可以用Lua脚本删除,判断value一样才能删除

  ~~~lua
  if redis.call("get",KEYS[1]) == ARGV[1] then
  return redis.call("del",KEYS[1])
  else
      return 0
  end
  ~~~

  * 为什么要用随机值?因为如果某个客户端获取到了锁,但是阻塞了很长时间才执行完,此时可能已经自动释放锁了,此时可能别的客户端已经获取到了这个锁,要是这个时候直接删除key的话会有问题,所以得用随机值加上Lua脚本来释放锁;

  <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210721210120104.png" alt="image-20210721210120104" style="zoom:33%;" />

  <img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210721211952203.png" alt="image-20210721211952203" style="zoom: 33%;" />

  上图实现的是可重入锁

* RedLock算法

  * 场景有一个redis cluster,有5个redis master实例,然后执行如下步骤获取一把锁;
  * 获取当前时间戳,单位是毫秒;
  * 轮流尝试在每个master节点上创建锁,过期时间较短,一般就几十毫秒;
  * 尝试在大多数节点上建立一个锁,比如5个节点就要求3个节点(n/2+1);
  * 客户端计算建立好锁的时间,如果建立锁的时间小于超时时间,就算建立成功了;
  * 要是锁建立失败了,那么就依次删除这个锁;
  * 只要别人建立了一把分布式锁,就得不断轮询去尝试获取锁;

### 3.6.4.常用的分布式锁实现技术

|               | 实现思路                                      | 优点                                     | 缺点                                                         |
| ------------- | --------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| 数据库实现    | 利用数据库自身提供的锁机制,要求数据库支持行锁 | 实现简单,稳定可靠                        | 性能差,无法适应高并发场景;容易出现死锁情况;无法优雅的实现阻塞式锁; |
| 缓存实现      | 使用setnx和lua脚本机制,保证对缓存操作的原子性 | 性能非常好                               | 实现相对复杂,有出现死锁的可能性,无法优雅的实现阻塞式锁;      |
| zookeeper实现 | 基于zk的节点特性以及watch机制实现             | 性能好,稳定可靠性高,能较好的实现阻塞式锁 | 实现相对复杂;                                                |

## 3.7.Zookeeper分布式协调场景

这个其实是zk很经典的一个用法，简单来说，就好比，你A系统发送个请求到mq，然后B消息消费之后处理了。那A系统如何知道B系统的处理结果？用zk就可以实现分布式系统之间的协调工作。A系统发送请求之后可以在zk上对某个节点的值注册个监听器，一旦B系统处理完了就修改zk那个节点的值，A立马就可以收到通知，完美解决。

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720233037718.png" alt="image-20210720233037718" style="zoom:33%;" /> 

## 3.8.Zookeeper HA高可用场景

比如hadoop、hdfs、yarn等很多大数据系统，都选择基于zk来开发HA高可用机制，就是一个重要进程一般会做主备两个，主进程挂了立马通过zk感知到切换到备用进程.

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210720233516312.png" alt="image-20210720233516312" style="zoom:33%;" />

