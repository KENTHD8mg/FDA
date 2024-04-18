# 《分布式应用技术基础之归终》

## 秒杀

### 主要问题

- 超卖

	- 乐观锁

		- update 库存表 set
   已售数=已售数+1,版本号=版本号+1
where 秒杀id =#{id} and 版本号 = #{version}

		- CAS（Compare And Swap）的ABA问题

			- ABA问题解决就是没吃Swap后给数据变更版本号，如上

- 并发

	- 限流

		- 令牌桶算法

			-  

			- Guava的实现

				- //创建令牌桶实例
private RateLimiter rateLimiter = RateLimiter.create(20);
// 阻塞式获得令牌才继续往下执行
rateLimiter.acquire();
// 就等3秒看是否可以获得令牌，返回Boolean值。
rateLimiter.tryAcquire(3, TimeUnit.SECONDS) 

	- 缓存

		- redis

	- 流量削峰

		- kafka

- 其他

	- 重复提交

		- token

	- 限制购买次数

		- redis：set userid buycount

## 分布式锁

### redis

- setNx key value PX timeout

	- 问题

		- 过期时间，A进程临界区还没出来，锁就过期了

			- 临界区退出时先检查锁是否还继续持有

				- 如果仍然持有则解锁退出临界区

				- 如果已经失效则回滚相关操作后退出临界区

			- 额外起一个线程，定期检查线程是否还持有锁，如果有则延长过期时间

				- Redisson 里面就实现了这个方案，使用“看门狗”定期检查（每1/3的锁时间检查1次），如果线程还持有锁，则刷新过期时间

		- A进程获得锁后执行临界区代码，master记录好锁信息，如果slave尚未同步完成master的锁，master宕机，slave被选为master，这个时候锁会失效，B进程依然可以获取锁

- redLock

	- 原理

		- 为了解决setNx master故障问题

		- 1、向奇数个实例一起请求加锁 2、获取半数以上加锁成功即为加锁成功 

	- 问题

		- 需要保证过期时间一致就需要严格依赖各个参与机器的时钟，例如5个实例，进程A获取了3个实例的锁，加锁成功后，实例1的的时钟走的比较快，导致锁信息提前过期释放，此时持有锁信息的实例只有两台，B进程仍有加锁成功的可能

		- 例如5个实例，进程A获取了3个实例的锁，加锁成功后，实例1的master宕机刚好又是slave尚未同步锁信息，导致持有锁信息的实例只有两台，此时B进程仍有加锁成功的可能

### Zookeeper

## 分布式事务

### 2PC

-  

-  

- 承诺

	- 系统承诺
初看与单节点的提交没啥两样，但是我们看下细节就明白了：
1、当应用要开始一个事务的时候，它向协调者申请一个全剧唯一的ID。
2、这个应用在每个节点上开始一个绑定ID的事务操作。如果有节点这个时候操作失败，协调者放弃这次事务。
3、当应用准备好了提交时，协调者发送一个绑定ID的prepare请求到每个参与者。如果任何节点失败或者超时，协调者放弃这次事务。
4、当参与者收到了prepare请求，它需要做的是确保无论如何都能成功提交，这包含将事务数据写到磁盘，检查数据内容是否合理。一旦回复yes，这个节点承诺一定能完成提交。
5、当协调者收到所有节点的回复时，它会做出决定提交还是放弃。协调者必须将这个决定持久化到磁盘，防止失败恢复时能知道之前的决定。
6、一旦协调者将决定持久化了，决定会下发到每个节点。如果下发失败或者超时，必须重试直到成功。如果决定是提交，那么必须被执行。如果一个节点宕机了，当它恢复时会被继续执行之前的决定。

- 故障分析

	- 当在准备阶段，协调者和个别参与者同时故障，新协调者可以执行回滚来保持数据一致；
当在提交阶段，协调者向第一个参与者发送了提交请求之后，协调者和该参与者同时故障，新的协调者既不知道原来的协调者的状态，也不知道故障参与者的状态，无法后续指令，确保数据一致，因为他不知道老协调者发的是回滚还是提交，后面他要怎么继续，即使知道老的协调者最后是发送提交命令，也不知道故障参与者提交是否执行成功，不知道给剩余其他参与者发送回滚还是提交来确保数据一致

		- 问题其实就出在每个参与者自身的状态只有自己和协调者知道

		- 不能保证一致性

### 3PC

-  

- 两点改进

	- 1、 引入超时机制。同时在协调者和参与者中都引入超时机制。
2、在第一阶段和第二阶段中插入一个准备阶段。保证了在最后提交阶段之前各参与节点的状态是一致的。

- 故障分析

	- 超时机制也会带来数据不一致的问题，比如在等待提交命令时候超时了，参与者默认执行的是提交事务操作，但是有可能执行的是回滚操作，这样一来数据就不一致了

	- 3PC对2PC中协调者和参与者同时故障的问题，采取简化的处理方法，如果新的协调者发现剩余协调者都处于预提交或者提交状态，则继续执行剩余操作，3PC协议基于一个假设，认为准备阶段如果成功执行，大概率后面的操作都能成功

### TCC

-  

- 特点

	- Try 指的是预留，即资源的预留和锁定，注意是预留，这个跟2PC准备阶段锁定资源有本质区别，比如TCC try阶段会库存冻结后commit，不占用这条记录的锁，2PC 准备阶段会会减库存，但不提交事务，持有事务锁

		- 如何预留资源
而有的人可能会好奇，什么冻结库存之类的玩意如何实现？

这个其实也很简单，比如你库存表本来只有一个库存数字段。但你现在要用这种TCC方案了，那么就给他再加上一个冻结库存数字段。

本来来说，你某个商品库存数是10 ， 那么用户再下单完成支付的这个流程中，假设买了1个商品，那么原来的逻辑就得改一下。

正常你买1个商品，那库存就是10-1=9 咯，所以将库存数改为9。而在TCC下，你就不将他改为9，而是在冻结库存数上+1，即数据库状态变为 库存数10，冻结库存数1

那么在其他的用户购买时，你返回给前端的库存数应该就是 库存数10 减去 冻结库存数1 等于现在只有9个库存能够被使用。

用户他不知道冻结不冻结，但是你的程序里边是知道的。用这种逻辑那是一点没有问题。

在你支付接口里边所有的预留操作成功之后，那么就如上所述，将库存字段真正减掉，也就是将10库存真正减1，将冻结库存还原，这是第二阶段该做的事情。



说到这里，总没人问第二阶段要是某环节错了怎么办吧？ 要是有这疑问的那肯定是没看我上一篇分布式事务文章。

这里也简单的讲一句：本身失败的概率就极小，要是操作失败了肯定是会重试、并且记录事务日志的。状态一个不对劲就由人肉团队、定时任务来进行补偿

- 注意点

	- 因此 TCC 对业务的侵入较大和业务紧耦合，需要根据特定的场景和业务逻辑来设计相应的操作。

还有一点要注意，撤销和确认操作的执行可能需要重试，因此还需要保证操作的幂等

并没有强调需要一个独立的协调者和事务管理器，协调者和事务管理器可以在应用中实现也可以独立实现

- 故障分析

	- 2PC的问题它都有

	- 空回滚

		- 首先发起方在调用参与者之前，会向TC申请开始一笔分布式事务。然后发起方调用参与者的一阶段方法，在调用实际发生之前，一般会有切面拦截器感知到此次Try调用，然后写入一条分支事务记录。紧接着，在实际调用参与者的Try方法时发生了异常。异常原因可以是发起方宕机，网络抖动等。

总而言之，就是Try方法没有执行成功，然而此时这笔分布式事务和分支事务已经落库。有两种情况会触发分布式事务的回滚：

发起方认为当前分布式事务无法成功，主动通知TC回滚
TC发现分布式事务超时，被动触发回滚
触发回滚操作后，TC会对该分布式事务关联的分支事务调用其二阶段Cancel。在执行Cancel时，Try还未执行成功，触发空回滚。如果不对空回滚加以防范的话，可能会造成资源的无效释放。即在没有预留资源的情况下就释放资源，造成故障。

	- 资源悬挂

	- 幂等

- Hmily 框架、TXC

### 消息

-  

## jVM基础

### classloader

- 双亲委派

	- java8及之前

		-  

	- java9及之后

		-  

			- 当平台以及应用程序类加载器收到类加载的请求的时候，在委派给父类加载器之前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责该模块的加载器完成加载

- 主要类加载器

	- java8及之前

		- 启动类加载器（Bootstrap Class Loader）

			- ① java语言编写

			- ② 父类加载器为扩展类加载器

			- ③ 负责加载环境变量classpanth或系统属性;java.class.path 指定路径下的类库

			- ④ 该类加载器是程序中默认的类加载器，一般来说，java应用的类都是由它来完成的加载

			- ⑤ 通过ClassLoader的getSystemClassLoader()可以获得该类加载器

		- 扩展类加载器（Extension Class Loader）

			- ① java语言编写获取

			- ② 父类加载器为启动类加载器

			- ③ 派生于ClassLoader类

		- 应用类加载器（Application Class Loader)

			- ① C++语言实现，嵌套在JVM内部

			- ② 用来加载Java核心库

			- ③ 加载扩展类和应用程序类加载器，并指定为他们的父类加载器

			- ④ 并不继承自java.lang.ClassLoader;没有父加载器

			- ⑤ 处于安全考虑，Bootstrap启动类加载器值加载包名为java，javax，sun等开头的类

	- java9及之后

		- 启动类加载器（Bootstrap Class Loader）

		- 平台类加载器（Platform ClassLoader）

		- 应用类加载器（Application Class Loader)

### JVM

- hashCode方法的作用

	- HashCode的存在主要是用于查找的快捷性，如Hashtable，HashMap等，HashCode经常用于确定对象的存储地址；

	- 如果两个对象相同， equals方法一定返回true，并且这两个对象的HashCode一定相同；

	- 两个对象的HashCode相同，并不一定表示两个对象就相同，即equals()不一定为true，只能够说明这两个对象在一个散列存储结构中。

	- 如果对象的equals方法被重写，那么对象的HashCode也尽量重写。

- 内存

	- 内存分区模型

		- 程序计数器

		- 栈

			- java虚拟机栈

				- 线程私有，生命周期与线程一样，描述的是Java方法执行的区域：每个方法被执行就会生成一个栈帧（Stack Frame）用于存储局部变量表，操作栈，动态链接，方法出口等信息。

				- 局部变量表存储编译器可知的各种基本数据类型（boolean byte char short int float long double)对象引用（reference）和returnAddress类型，其中float和double占用两个局部变量空间Slot，其余占用一个。

				- 两种异常

					- 线程请求的深度大于虚拟机允许深度，StackOverFlowError，

					- 无法申请到足够的空间：OutOfMemoryError

			- 本地方法栈

				- 与虚拟机栈的作用相同，只不过执行的是本地方法Native，HotSpot把java虚拟机栈和本地方法栈合二为一。

		- 堆

			- java堆（java Heap）

				- 是内存中占用最大的一块，被所有线程共享。所有的对象实例和数组都在这上面分配，但是随着JIT编译器的发展和逃逸技术的成熟等，也不是那么绝对了。

				- Java堆是内存回收的主要区域，也成为“GC堆”（Garbage Collected Heap）

		- 方法区

			- 方法区

				- 用来存储虚拟机加载的类信息，常量，静态常量，即时编译器编译后的代码等。也就是平时大家说的永久代，本质上并不等价，或者说使用永久代实现方法区而已。一般方法区内存回收成绩不令人满意。

			- 运行时常量池

				- 是方法区的一部分，Class文件除了有类的版本信息、字段方法接口外，还有一项信息是常量池，用于存放编译器生成的各种字面量和符号引用，这部分信息在类加载后存放到方法区的运行时常量池中。
具有动态性，可以使用String类的intern（）方法加入。

		- 直接内存

			- 并不是虚拟机运行时数据区的一部分。JDK1.4中引入了NIO类，引入了一种基于通道与缓冲区的I/O方式，可以使用Native直接分配堆外内存，避免了再Java堆和Native堆中来回复制数据。

	- hotspot实现中的GC回收机制

		- GC优化的目的

			- GC调优中，GC导致的应用暂停时间影响系统响应速度，GC处理线程的CPU使用率影响系统吞吐量

				- 响应速度(Responsiveness) 响应速度指程序或系统对一个请求的响应有多迅速。比如，用户订单查询响应时间，对响应速度要求很高的系统，较大的停顿时间是不可接受的。调优的重点是在短的时间内快速响应

				- 吞吐量(Throughput) 吞吐量关注在一个特定时间段内应用系统的最大工作量，例如每小时批处理系统能完成的任务数量，在吞吐量方面优化的系统，较长的GC停顿时间也是可以接受的，因为高吞吐量应用更关心的是如何尽可能快地完成整个任务，不考虑快速响应用户请求

		- GC分代收集算法

			-  

				- 新生代

					- 新生代又叫年轻代，大多数对象在新生代中被创建，很多对象的生命周期很短。每次新生代的垃圾回收（又称Young GC、Minor GC、YGC）后只有少量对象存活，所以使用复制算法，只需少量的复制操作成本就可以完成回收

						- 新生代内又分三个区：一个Eden区，两个Survivor区(S0、S1，又称From Survivor、To Survivor)，大部分对象在Eden区中生成。当Eden区满时，还存活的对象将被复制到两个Survivor区（中的一个）。当这个Survivor区满时，此区的存活且不满足晋升到老年代条件的对象将被复制到另外一个Survivor区。对象每经历一次复制，年龄加1，达到晋升年龄阈值后(默认是15)，转移到老年代

				- 老年代

					- 在新生代中经历了N次垃圾回收后仍然存活的对象，就会被放到老年代，该区域中对象存活率高。老年代的垃圾回收通常使用“标记-整理”算法

				- jvm8之前永久代、jvm8之后的元数据区

			- 内存分配策略

				- 对象优先在Eden区分配

					-  大多数情况下，对象在先新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Young GC

				- 大对象直接进入老年代

					-  JVM提供了一个对象大小阈值参数(-XX:PretenureSizeThreshold，默认值为0，代表不管多大都是先在Eden中分配内存)，大于参数设置的阈值值的对象直接在老年代分配，这样可以避免对象在Eden及两个Survivor直接发生大内存复制

						- 有个特殊情况（动态年龄判定：新生代对象的年龄可能没达到阈值(MaxTenuringThreshold参数指定)就晋升老年代，如果Young GC之后，新生代存活对象达到相同年龄所有对象大小的总和大于任一Survivor空间(S0 或 S1总空间)的一半，此时S0或者S1区即将容纳不了存活的新生代对象，年龄大于或等于该年龄的对象就可以直接进入老年代，无须等到MaxTenuringThreshold中要求的年龄）

				- 长期存活的对象将进入老年代

					-  对象每经历一次垃圾回收，且没被回收掉，它的年龄就增加1，大于年龄阈值参数(-XX:MaxTenuringThreshold，默认15)的对象，将晋升到老年代中

				- 空间分配担保

					- 当进行Young GC之前，JVM需要预估：老年代是否能够容纳Young GC后新生代晋升到老年代的存活对象，以确定是否需要提前触发GC回收老年代空间，基于空间分配担保策略来计算（continueSize老年代最大可用连续空间）

						-  

		- GC事件分类

			- Young GC、Minor GC

				- 当JVM无法为新对象分配在新生代内存空间时总会触发 Young GC，比如 Eden 区占满时。新对象分配频率越高, Young GC 的频率就越高

					- Young GC 每次都会引起全线停顿(Stop-The-World)，暂停所有的应用线程，停顿时间相对老年代GC的造成的停顿，几乎可以忽略不计

						- 日志

							-  

			- Old GC

				- 只清理老年代空间的GC事件

			-  Full GC

				- CMS的并发收集是这个模式

					- 清理整个堆的GC事件，包括新生代、老年代、元空间等

						- 日志

							-  

			- Mixed GC

				- 清理整个新生代以及部分老年代的GC，只有G1有这个模式

		- 垃圾回收器

			- CMS

				- Concurrent Mark and Sweep 并发-标记-清除

					- 名词解释

						- 可达性分析算法：用于判断对象是否存活，基本思想是通过一系列称为“GC Root”的对象作为起点（常见的GC Root有系统类加载器、栈中的对象、处于激活状态的线程等），基于对象引用关系，从GC Roots开始向下搜索，所走过的路径称为引用链，当一个对象到GC Root没有任何引用链相连，证明对象不再存活

						- Stop The World：GC过程中分析对象引用关系，为了保证分析结果的准确性，需要通过停顿所有Java执行线程，保证引用关系不再动态变化，该停顿事件称为Stop The World(STW)

						- Safepoint：代码执行过程中的一些特殊位置，当线程执行到这些位置的时候，说明虚拟机当前的状态是安全的，如果有需要GC，线程可以在这个位置暂停。HotSpot采用主动中断的方式，让执行线程在运行期轮询是否需要暂停的标志，若需要则中断挂起

					- 新生代垃圾回收

						- 能与CMS搭配使用的新生代垃圾收集器有Serial收集器和ParNew收集器

							-  

					- 老年代垃圾回收

						- CMS GC以获取最小停顿时间为目的，尽可能减少STW时间，可以分为7个阶段

							-  

								- 阶段 1: 初始标记(Initial Mark)

									- 此阶段的目标是标记老年代中所有存活的对象, 包括 GC Root 的直接引用, 以及由新生代中存活对象所引用的对象，触发第一次STW事件

这个过程是支持多线程的（JDK7之前单线程，JDK8之后并行，可通过参数CMSParallelInitialMarkEnabled调整）

									-  

								- 阶段 2: 并发标记(Concurrent Mark)

									- 此阶段GC线程和应用线程并发执行，遍历阶段1初始标记出来的存活对象，然后继续递归标记这些对象可达的对象

									-  

								- 阶段 3: 并发预清理(Concurrent Preclean)

									- 此阶段GC线程和应用线程也是并发执行，因为阶段2是与应用线程并发执行，可能有些引用关系已经发生改变。 通过卡片标记(Card Marking)，提前把老年代空间逻辑划分为相等大小的区域(Card)，如果引用关系发生改变，JVM会将发生改变的区域标记位“脏区”(Dirty Card)，然后在本阶段，这些脏区会被找出来，刷新引用关系，清除“脏区”标记

									-  

								- 阶段 4: 并发可取消的预清理(Concurrent Abortable Preclean)

									- 此阶段也不停止应用线程. 本阶段尝试在 STW的 最终标记阶段(Final Remark)之前尽可能地多做一些工作，以减少应用暂停时间，在该阶段不断循环处理：标记老年代的可达对象、扫描处理Dirty Card区域中的对象，循环的终止条件有： 1 达到循环次数 2 达到循环执行时间阈值 3 新生代内存使用率达到阈值

								- 阶段 5: 最终标记(Final Remark)

									- 这是GC事件中第二次(也是最后一次)STW阶段，目标是完成老年代中所有存活对象的标记。在此阶段执行： 1 遍历新生代对象，重新标记 2 根据GC Roots，重新标记 3 遍历老年代的Dirty Card，重新标记

									- 二次标记的必要性

										- 三色标记法

								- 阶段 6: 并发清除(Concurrent Sweep)

									- 此阶段与应用程序并发执行，不需要STW停顿，根据标记结果清除垃圾对象

									-  

								- 阶段 7: 并发重置(Concurrent Reset)

									- 此阶段与应用程序并发执行，重置CMS算法相关的内部数据, 为下一次GC循环做准备

					- CMS常见问题

						- 空间担保

							-  

						- 最终标记阶段停顿时间过长问题

							- CMS的GC停顿时间约80%都在最终标记阶段(Final Remark)

								- 若该阶段停顿时间过长，常见原因是新生代对老年代的无效引用，在上一阶段的并发可取消预清理阶段中，来不及触发Young GC，清理这些无效引用

									- 通过添加参数：-XX:+CMSScavengeBeforeRemark。在执行最终操作之前先触发Young GC，从而减少新生代对老年代的无效引用

										- 但如果在上个阶段(并发可取消的预清理)已触发Young GC，也会重复触发Young GC

						- 并发模式失败(concurrent mode failure) & 晋升失败(promotion failed)问题

							- 并发模式失败

								-  

								- 并发模式失败：当CMS在执行回收时，新生代发生垃圾回收，同时老年代又没有足够的空间容纳晋升的对象时，CMS 垃圾回收就会退化成单线程的Full GC。所有的应用线程都会被暂停，老年代中所有的无效对象都被回收

									- Full GC == Major GC指的是对老年代/永久代的stop the world的GC

									- Full GC的次数 = 老年代GC时 stop the world的次数

									- Full GC的时间 = 老年代GC时 stop the world的总时间

									- CMS 不等于Full GC，我们可以看到CMS分为多个阶段，只有stop the world的阶段被计算到了Full GC的次数和时间，而和业务线程并发的GC的次数和时间则不被认为是Full GC。CMS主要可以分为initial mark(stop the world), concurrent mark, remark(stop the world), concurrent sweep几个阶段，其中initial mark和remark会stop the world。

									- Full GC本身不会先进行Minor GC，我们可以配置，让Full GC之前先进行一次Minor GC，因为老年代很多对象都会引用到新生代的对象，先进行一次Minor GC可以提高老年代GC的速度。比如老年代使用CMS时，设置CMSScavengeBeforeRemark优化，让CMS remark之前先进行一次Minor GC。

							- 晋升失败

								-  

								- 晋升失败：当新生代发生垃圾回收，老年代有足够的空间可以容纳晋升的对象，但是由于空闲空间的碎片化，导致晋升失败，此时会触发单线程且带压缩动作的Full GC

							- 并发模式失败和晋升失败都会导致长时间的停顿，常见解决思路

								- 降低触发CMS GC的老年代空间阈值，即参数-XX:CMSInitiatingOccupancyFraction的值，让CMS GC尽早执行，以保证有足够的空间

								- 增加CMS线程数，即参数-XX:ConcGCThreads

								- 增大老年代空间

								- 让对象尽量在新生代回收，避免进入老年代

						- 内存碎片问题

							- 通常CMS的GC过程基于标记清除算法，不带压缩动作，导致越来越多的内存碎片需要压缩，常见以下场景会触发内存碎片压缩

								- 新生代Young GC出现新生代晋升担保失败(promotion failed)

								- 程序主动执行System.gc()

							- 可通过参数CMSFullGCsBeforeCompaction的值，设置多少次Full GC触发一次压缩，默认值为0，代表每次进入Full GC都会触发压缩，带压缩动作的算法为上面提到的单线程Serial Old算法，暂停时间(STW)时间非常长，需要尽可能减少压缩时间

			- G1

				- G1(Garbage-First）是一款面向服务器的垃圾收集器，支持新生代和老年代空间的垃圾收集，主要针对配备多核处理器及大容量内存的机器，G1最主要的设计目标是: 实现可预期及可配置的STW停顿时间

					- G1堆空间划分

						-  

							- Region

								- 为实现大内存空间的低停顿时间的回收，将划分为多个大小相等的Region。每个小堆区都可能是 Eden区，Survivor区或者Old区，但是在同一时刻只能属于某个代

									- 在逻辑上, 所有的Eden区和Survivor区合起来就是新生代，所有的Old区合起来就是老年代，且新生代和老年代各自的内存Region区域由G1自动控制，不断变动

							- 巨型对象

								- 当对象大小超过Region的一半，则认为是巨型对象(Humongous Object)，直接被分配到老年代的巨型对象区(Humongous regions)，这些巨型区域是一个连续的区域集，每一个Region中最多有一个巨型对象，巨型对象可以占多个Region

							- 空间划分的意义

								- 每次GC不必都去处理整个堆空间，而是每次只处理一部分Region，实现大容量内存的GC

								- 通过计算每个Region的回收价值，包括回收所需时间、可回收空间，在有限时间内尽可能回收更多的内存，把垃圾回收造成的停顿时间控制在预期配置的时间范围内，这也是G1名称的由来: garbage-first

					- G1工作模式

						- 针对新生代和老年代，G1提供2种GC模式，Young GC和Mixed GC，两种都会导致Stop The World

							- Young GC 当新生代的空间不足时，G1触发Young GC回收新生代空间，Young GC主要是对Eden区进行GC，它在Eden空间耗尽时触发，基于分代回收思想和复制算法，每次Young GC都会选定所有新生代的Region，同时计算下次Young GC所需的Eden区和Survivor区的空间，动态调整新生代所占Region个数来控制Young GC开销

							- Mixed GC 当老年代空间达到阈值会触发Mixed GC，选定所有新生代里的Region，根据全局并发标记阶段统计得出收集收益高的若干老年代 Region。在用户指定的开销目标范围内，尽可能选择收益高的老年代Region进行GC，通过选择哪些老年代Region和选择多少Region来控制Mixed GC开销

								- 并发标记

									-  

										- 阶段 1: 初始标记(Initial Mark) 暂停所有应用线程（STW），并发地进行标记从 GC Root 开始直接可达的对象（原生栈对象、全局对象、JNI 对象），当达到触发条件时，G1 并不会立即发起并发标记周期，而是等待下一次新生代收集，利用新生代收集的 STW 时间段，完成初始标记，这种方式称为借道（Piggybacking），其实这一步并不在mix gc中，是mix gc的前置步骤

										- 阶段 2: 根区域扫描（Root Region Scan） 在初始标记暂停结束后，新生代收集也完成了对象复制到 Survivor 的工作，应用线程开始活跃起来； 此时为了保证标记算法的正确性，所有新复制到 Survivor 分区的对象，需要找出哪些对象存在对老年代对象的引用，把这些对象标记成根(Root)； 这个过程称为根分区扫描（Root Region Scanning），同时扫描的 Suvivor 分区也被称为根分区（Root Region）； 根分区扫描必须在下一次新生代垃圾收集启动前完成（接下来并发标记的过程中，可能会被若干次新生代垃圾收集打断），因为每次 GC 会产生新的存活对象集合，因为第一步是借道的，所以部分被标记的新生代对象可能被复制到其他区域，所以需要再扫描一遍对老年代有引用的新生代对象，为下一步用新生代对象和GC ROOT对象去标记老年代做准备

										- 阶段 3: 并发标记（Concurrent Marking） 标记线程与应用程序线程并行执行，标记各个堆中Region的存活对象信息，这个步骤可能被新的 Young GC 打断 所有的标记任务必须在堆满前就完成扫描，如果并发标记耗时很长，那么有可能在并发标记过程中，又经历了几次新生代收集

										- 阶段 4: 再次标记(Remark) 和CMS类似暂停所有应用线程（STW），以完成标记过程短暂地停止应用线程, 标记在并发标记阶段发生变化的对象，和所有未被标记的存活对象，同时完成存活数据计算

										- 阶段 5: 清理(Cleanup) 为即将到来的转移阶段做准备, 此阶段也为下一次标记执行所有必需的整理计算工作

											- 阶段 5: 清理(Cleanup) 为即将到来的转移阶段做准备, 此阶段也为下一次标记执行所有必需的整理计算工作

											- 回收不包含存活对象的Region

											- 统计计算回收收益高（基于释放空间和暂停目标）的老年代分区集合

									- G1的运行过程是这样的，会在Young GC和Mix GC之间不断的切换运行，同时定期的做全局并发标记，在实在赶不上回收速度的情况下使用Full GC(Serial GC)。初始标记是搭在YoungGC上执行的，在进行全局并发标记的时候不会做Mix GC，在做Mix GC的时候也不会启动初始标记阶段。当MixGC赶不上对象产生的速度的时候就退化成Full GC，这一点是需要重点调优的地方。

						- 调优

							- G1的正常处理流程中没有Full GC，只有在垃圾回收处理不过来(或者主动触发)时才会出现， G1的Full GC就是单线程执行的Serial old gc，会导致非常长的STW

								- 常见原因

									- 程序主动执行System.gc()

									- 全局并发标记期间老年代空间被填满（并发模式失败）

									- Mixed GC期间老年代空间被填满（晋升失败）

									- Young GC时Survivor空间和老年代没有足够空间容纳存活对象

								- 常见的解决

									- 增大-XX:ConcGCThreads=n 选项增加并发标记线程的数量，或者STW期间并行线程的数量：-XX:ParallelGCThreads=n

									- 减小-XX:InitiatingHeapOccupancyPercent 提前启动标记周期

									- 增大预留内存 -XX:G1ReservePercent=n ，默认值是10，代表使用10%的堆内存为预留内存，当Survivor区域没有足够空间容纳新晋升对象时会尝试使用预留内存

							- 巨型对象分配问题

								- 巨型对象区中的每个Region中包含一个巨型对象，剩余空间不再利用，导致空间碎片化，当G1没有合适空间分配巨型对象时，G1会启动串行Full GC来释放空间。可以通过增加 -XX:G1HeapRegionSize来增大Region大小，这样一来，相当一部分的巨型对象就不再是巨型对象了，而是采用普通的分配方式

							- 不要设置Young区的大小

								- 原因是为了尽量满足目标停顿时间，逻辑上的Young区会进行动态调整。如果设置了大小，则会覆盖掉并且会禁用掉对停顿时间的控制

							- 平均响应时间设置

								- 使用应用的平均响应时间作为参考来设置MaxGCPauseMillis，JVM会尽量去满足该条件，可能是90%的请求或者更多的响应时间在这之内， 但是并不代表是所有的请求都能满足，平均响应时间设置过小会导致频繁GC

							- GC优化的核心思路在于：尽可能让对象在新生代中分配和回收，尽量避免过多对象进入老年代，导致对老年代频繁进行垃圾回收，同时给系统足够的内存减少新生代垃圾回收次数

							- 分析系统的运行状况

								- 指标

									- 系统每秒请求数、每个请求创建多少对象，占用多少内存

									- Young GC触发频率、对象进入老年代的速率

									- 老年代占用内存、Full GC触发频率、Full GC触发的原因、长时间Full GC的原因

								- 常用工具

									- jstat -gc <pid> <统计间隔时间>  <统计次数>

										-  

									- // 命令行输出类名、类数量数量，类占用内存大小，
// 按照类占用内存大小降序排列
jmap -histo <pid>
 
// 生成堆内存转储快照，在当前目录下导出dump.hrpof的二进制文件，
// 可以用eclipse的MAT图形化工具分析
jmap -dump:live,format=b,file=dump.hprof <pid>

									- jinfo <pid> 

										- 用来查看正在运行的 Java 应用程序的扩展参数，包括Java System属性和JVM命令行参数

									- 其他

										- 监控告警系统：Zabbix、Prometheus、Open-Falcon

										- jdk自动实时内存监控工具：VisualVM

										- 堆外内存监控： Java VisualVM安装Buffer Pools 插件、google perf工具、Java NMT(Native Memory Tracking)工具

										- GC日志分析：GCViewer、gceasy

										- GC参数检查和优化：xxfox.perfma.com/

			- GC优化案例

				- 数据分析平台系统频繁Full GC
平台主要对用户在APP中行为进行定时分析统计，并支持报表导出，使用CMS GC算法。数据分析师在使用中发现系统页面打开经常卡顿，通过jstat命令发现系统每次Young GC后大约有10%的存活对象进入老年代。

原来是因为Survivor区空间设置过小，每次Young GC后存活对象在Survivor区域放不下，提前进入老年代，通过调大Survivor区，使得Survivor区可以容纳Young GC后存活对象，对象在Survivor区经历多次Young GC达到年龄阈值才进入老年代，调整之后每次Young GC后进入老年代的存活对象稳定运行时仅几百Kb，Full GC频率大大降低

				- 业务对接网关OOM
网关主要消费Kafka数据，进行数据处理计算然后转发到另外的Kafka队列，系统运行几个小时候出现OOM，重启系统几个小时之后又OOM，通过jmap导出堆内存，在eclipse MAT工具分析才找出原因：代码中将某个业务Kafka的topic数据进行日志异步打印，该业务数据量较大，大量对象堆积在内存中等待被打印，导致OOM

				- 账号权限管理系统频繁长时间Full GC
系统对外提供各种账号鉴权服务，使用时发现系统经常服务不可用，通过Zabbix的监控平台监控发现系统频繁发生长时间Full GC，且触发时老年代的堆内存通常并没有占满，发现原来是业务代码中调用了System.gc()

	- jvm内存模型跟GC内存模型的关系

		-  

		- 永久代和方法区的联系

			- 永久代是Hotspot虚拟机特有的概念，是方法区的一种实现，别的JVM都没有这个东西。在Java
8中，永久代被彻底移除，取而代之的是另一块与堆不相连的本地内存——元空间。 永久代或者“Perm
Gen”包含了JVM需要的应用元数据，这些元数据描述了在应用里使用的类和方法。注意，永久代不是Java堆内存的一部分。永久代存放JVM运行时使用的类。永久代同样包含了Java
SE库的类和方法。永久代的对象在full GC时进行垃圾收集。

				- 《Java虚拟机规范》只是规定了有方法区这么个概念和它的作用，并没有规定如何去实现它。那么，在不同的 JVM 上方法区的实现肯定是不同的了。 同时大多数用的JVM都是Sun公司的HotSpot。在HotSpot上把GC分代收集扩展至方法区，或者说使用永久代来实现方法区。换句话说：方法区是一种规范，永久代是Hotspot针对这一规范的一种实现。而永久代本身也在迭代中：

在Java 6中，方法区中包含的数据，除了JIT编译生成的代码存放在native memory的CodeCache区域，其他都存放在永久代；
在Java 7中，Symbol的存储从PermGen移动到了native memory，并且把静态变量从instanceKlass末尾（位于PermGen内）移动到了java.lang.Class对象的末尾（位于普通Java heap内）；
在Java 8中，永久代被彻底移除，取而代之的是另一块与堆不相连的本地内存——元空间（Metaspace）,‑XX:MaxPermSize 参数失去了意义，取而代之的是-XX:MaxMetaspaceSize。

对于Java8， HotSpots取消了永久代，那么是不是也就没有方法区了呢？当然不是，方法区是一个规范，规范没变，它就一直在。那么取代永久代的就是元空间。它与永久代有什么不同的？

存储位置不同，永久代是堆的一部分，和新生代，老年代地址是连续的，而元空间属于本地内存；

存储内容不同，元空间存储类的元信息，静态变量和常量池等并入堆中。相当于永久代的数据被分到了堆和元空间中。

二、方法区里存着什么？
既然永久代是方法区的一种实现，那么在Hotspot下，方法区就等于永久代，也被称为非堆。那方法区里都存着什么呢？先抛结论：

静态变量 + 常量 + 类信息(构造方法/接口定义) + 运行时常量池存在方法区中 。

				- 上面说过，HotSpot虚拟机在1.8之后已经取消了永久代，改为元空间，类的元信息被存储在元空间中。元空间没有使用堆内存，而是与堆不相连的本地内存区域。所以，理论上系统可以使用的内存有多大，元空间就有多大，所以不会出现永久代存在时的内存溢出问题。这项改造也是有必要的，永久代的调优是很困难的，虽然可以设置永久代的大小，但是很难确定一个合适的大小，因为其中的影响因素很多，比如类数量的多少、常量数量的多少等。永久代中的元数据的位置也会随着一次full GC发生移动，比较消耗虚拟机性能。同时，HotSpot虚拟机的每种类型的垃圾回收器都需要特殊处理永久代中的元数据。将元数据从永久代剥离出来，不仅实现了对元空间的无缝管理，还可以简化Full GC以及对以后的并发隔离类元数据等方面进行优化。

	- Java 内存模型(简称 JMM)

		- JMM 是共享内存的并发模型，线程之间主要通过读-写共享变量(堆内存中的实例域，静态域和数组元素)来完成隐式通信。JMM 控制 Java 线程之间的通信，决定一个线程对共享变量的写入何时对另一个线程可见。Java 内存模型中规定了所有的变量都存储在主内存中，每条线程有自己的工作内存。线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量。

## 分布式理论

### CAP

### BASE

- Basically Available（基本可用）

- Soft-state（ 软状态）

- Eventual Consistency（最终一致性）

### 一致性协议

- 2PC、3PC

- 共识协议

	- BFT（Byzantine fault）

	- CFT（Crash fault）

		- Paxos

			-  

			-  

			- 原理

				- paxos的角色

					- 提案者：负责提出提案，这里提案可以是任何可以达成共识的东西，比如某个要写入DB的值，一条日志等等；
接受者：接收提案，确定接受还是拒绝提案；
学习者：获取并同步最终选择的提案；

				- paxos可以分为两个阶段

					- 第一阶段：准备阶段，提案者选择一个提案号，发送提案给接受者，试探能否得到半数以上接受者的响应；
第二阶段：如果第一阶段收到超过半数的接受者的响应，则提交提案，如果能够得到半数以上接受者的响应，则共识完成；

				- 故事

					- axos算法的论证过程虽然比较难理解，但是实际的操作过程却比较简单，网上有人用一个形象的例子来说明paxos达成共识的过程：

    假设有2个商人P1和P2想从政府买块地，商人想要最终拿下这块地，需要经过3个政府官员A1，A2和A3的审批，只有经过2个以上的官员同意，才能最终拿下地皮，现在的目标是：最终只能有一个商人拿到地。另外假设商人要想通过官员的审批，必须给一定数量的贿赂费。

    我们看看这样一个场景下，如何达成共识（选定一个商人把地批给他）。

    拿地是一件争分夺秒的事情，于是商人P1和P2开始准备了：

    (1) 假设P1的行动快一些，他很快找到了官员A1和A2，告诉两人只要批了地就给100W的感谢费，两个官员很高兴，告诉P1回去拿钱吧；

    注：这一步实际上是P1在进行paxos算法中的准备阶段；

    (2) 商人P2在P1之前先找了官员A3，告诉他只要批了地就给他200W，A3愉快的接受了；

    (3) P2又跑到官员A1和A2那，承诺只要批地，就给200W，因为这份费用比此前P1承诺的高，于是贪财的官员A1和A2变卦了；

    注：以上两步是P2在进行paxos的准备阶段；

    (4) 商人P1此前已经初步贿赂A1和A2成功，于是满心欢喜的拿着准备好的钱找到了A1和A2，结果却是A1和A2都告诉他：对不起，Mr XX，就在刚刚有人承诺了200W，你是个聪明人，你懂得该怎么做的。商人P1很是郁闷，告诉A1和A2：容我想想再给答复；

    注：以上P1拿钱给A1和A2，对应与paxos算法的提交阶段，因为此前P1已经得到了3位官员中2位的同意；

    (5) 就在P1还在犹豫要不要提高贿赂费的时候，商人P2按之前承诺的向A1和A2的账户分别打入200W，于是A1，A2拿钱办事，批准通过，因为超过半数的官员审批通过，于是在政府网站上向大众公布P2最终拿地成功，所有人都知道了这个结果。

    注意上面的过程中的一个问题：假设上面第（4）步中P1被拒绝以后，立刻向官员承诺一个更高的费用，那么当商人P2拿着钱到A1和A2时，同样也会被拒绝，于是P2又可能会抬价，这样交替下去就可能造成死循环，这就是paxos算法中的活锁问题

			- 问题

				- 活死锁

					- proposer通过accpter返回的消息知道此时有更高编号的提案被提出时，该proposer静默一段时间

						-  

				- 脑裂

			- 变种

				- ZAB（Zookeeper Atomic Broadcast）

					- 选举协议

						-  

					- 数据广播协议

						-  

				- Raft

					- 选举leader

						-  

					- 日志同步

						-  

					- Raft协议一共包含如下3类角色：

Leader（领袖）：领袖由群众投票选举得出，每次选举，只能选出一名领袖；
Candidate（候选人）：当没有领袖时，某些群众可以成为候选人，然后去竞争领袖的位置；
Follower（群众）：这个很好理解，就不解释了。
然后在进行选举过程中，还有几个重要的概念：

Leader Election（领导人选举）：简称选举，就是从候选人中选出领袖；
Term（任期）：它其实是个单独递增的连续数字，每一次任期就会重新发起一次领导人选举；
Election Timeout（选举超时）：就是一个超时时间，当群众超时未收到领袖的心跳时，会重新进行选举。


			- libpaxos

		- 工作量证明(POW)的算法

			- 比特币

		- Gossip

			- Gossip protocol 也叫 Epidemic Protocol （流行病协议）

				- 流程

					- 种子节点周期性的散播消息 【假定把周期限定为 1 秒】。
被感染节点随机选择N个邻接节点散播消息【假定fan-out(扇出)设置为6，每次最多往6个节点散播】。
节点只接收消息不反馈结果。
每次散播消息都选择尚未发送过的节点进行散播。
收到消息的节点不再往发送节点散播：A -> B，那么B进行散播的时候，不再发给 A。

				- 通讯

					- Push: 节点 A 将数据 (key,value,version) 及对应的版本号推送给 B 节点，B 节点更新 A 中比自己新的数据
Pull：A 仅将数据 key, version 推送给 B，B 将本地比 A 新的数据（Key, value, version）推送给 A，A 更新本地
Push/Pull：与 Pull 类似，只是多了一步，A 再将本地比 B 新的数据推送给 B，B 则更新本地

				- 状态

					- Gossip 协议的消息传播方式有两种：Anti-Entropy(反熵传播)和Rumor-Mongering(谣言传播)

						- 反熵传播所有参与节点只有两种状态：Suspective(病原)、Infective(感染)

							- 这种节点状态又叫做simple epidemics(SI model)。过程是种子节点会把所有的数据都跟其他节点共享，以便消除节点之间数据的任何不一致，它可以保证最终、完全的一致。缺点是消息数量非常庞大，且无限制；通常只用于新加入节点的数据初始化。

						- 谣言传播是以固定的概率仅传播新到达的数据。所有参与节点有三种状态：Suspective(病原)、Infective(感染)、Removed(愈除)

							- 这种节点状态又叫做complex epidemics(SIR model)。过程是消息只包含最新 update，谣言消息在某个时间点之后会被标记为 removed，并且不再被传播。缺点是系统有一定的概率会不一致，通常用于节点间数据增量同步

	- FLP不可能原理

		- 异步分布式系统的共识问题的通用解决方法是：无解

- NWR

	- Amazon Dynamo的NWR模型。NWR模型把CAP的选择权交给了用户，让用户自己的选择你的CAP中的哪两个。

		- N — 数据复制的份数
W — 更新数据是需要保证写完成的节点数
R — 读取数据的时候需要读取的节点数

			-     1.如果W+R>N：则是强一致性，写的节点和读的节点重叠。例如对于典型的一主一备同步复制的关系型数据库。N=2,W=2,R=1，则不管读的是主库还是备库的数据，都是一致的。

    2.如果W+R<=N：则是弱一致性。例如对于一主一备异步复制的关系型数据库，N=2,W=1,R=1，则如果读的是备库，就可能无法读取主库已经更新过的数据，所以是弱一致性。

			- 对于分布式系统，为了保证高可用性，一般设置N>=3。不同的N,W,R组合，是在可用性和一致性之间取一个平衡，以适应不同的应用场景。

如果N=W,R=1，任何一个写节点失效，都会导致写失败，因此可用性会降低，但是由于数据分布的N个节点是同步写入的，因此可以保证强一致性。
如果N=R,W=1，只需要一个节点写入成功即可，写性能和可用性都比较高。但是读取其他节点的进程可能不能获取更新后的数据，因此是弱一致性。这种情况下，如果W<(N+1)/2，并且写入的节点不重叠的话，则会存在写冲突  

### 服务发现

- eureka

- nacos

- Zookeeper

## 分布式中间件

### 微服务

- Spring Cloud

	-  

		- 重点：Eureka、Ribbon、Feign、Hystrix、Zuul、Sleuth

			-  

	- Eureka

		- 服务提供者

服务注册：启动的时候会通过发送REST请求的方式将自己注册到Eureka Server上，同时带上了自身服务的一些元数据信息。

服务续约：在注册完服务之后，服务提供者会维护一个心跳用来持续告诉Eureka Server:  "我还活着 ” 、

服务下线：当服务实例进行正常的关闭操作时，它会触发一个服务下线的REST请求给Eureka Server, 告诉服务注册中心：“我要下线了 ”。

服务消费者

获取服务：当我们启动服务消费者的时候，它会发送一个REST请求给服务注册中心，来获取上面注册的服务清单

服务调用：服务消费者在获取服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。在进行服务调用的时候，优先访问同处一个Zone中的服务提供方。

Eureka Server(服务注册中心)：

失效剔除：默认每隔一段时间（默认为60秒） 将当前清单中超时（默认为90秒）没有续约的服务剔除出去。

自我保护：。EurekaServer 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%(通常由于网络不稳定导致)。Eureka Server会将当前的实例注册信息保护起来， 让这些实例不会过期，尽可能保护这些注册信息

	- Sleuth

		- 常用术语：Trace

它是由一组有相同Trace ID的Span串联形成一个树状结构。为了实现请求跟踪，当请求请求到分布式系统的入口端点时，只需要服务跟踪框架为该请求创建一个唯一的跟踪标志，同时在分布式系统内部流转的时候，框架始终保持传递该唯一标志，直到返回请求为止，我们通过它将所有请求过程中的日志关联起来。一句话：类似于树结构的Span集合，表示一条调用链路，存在唯一标志。

常用术语：Span

它代表了一个基础的工作单元，例如服务调用。为了统计各处理单元的时间延迟，当前请求到达各个服务组件时，也通过一个唯一标志来标记它的开始、具体过程以及结束。通过span的开始和结束的时间戳，就能统计该span的时间延迟，除此之外，我们还可以获取如事件名称、请求信息等元数据。一句话：每个trace中会调用若干个服务，为了记录调用了哪些服务，以及每次调用的消耗时间等信息，在每次调用服务时，埋入一个调用记录。有可以这么理解：表示调用链路来源，通俗的理解span就是一次请求信息。

常用术语：Annotation

它用于记录一段时间内的事件。内部使用的最重要的术语是：

cs - Client Sent/Start - 客户端发送一个请求，这个注解描述了这个Span的开始。

sr - Server Received/Start - 服务端获得请求并准备开始处理它，其中（sr – cs） 时间戳便可得到网络传输的时间。

ss - Server Sent/Finish （服务端发送响应）– 该注解表明请求处理的完成(当请求返回客户端)， （ss – sr）时间戳就可以得到服务器请求的时间。

cr - Client Received/Finished （客户端接收响应）- 表明此时Span的结束，（cr – cs）时间戳便可以得到整个请求所消耗的时间。

	- Feign

		-  

	- Ribbon

		- 首先Ribbon会从 Eureka Client里获取到对应的服务注册表，也就知道了所有的服务都部署在了哪些机器上，在监听哪些端口号。
然后Ribbon就可以使用默认的Round Robin算法，从中选择一台机器
Feign就会针对这台机器，构造并发起请求。

	- Hystrix

		-  

		- 这时就轮到Hystrix闪亮登场了。Hystrix是隔离、熔断以及降级的一个框架。啥意思呢？说白了，Hystrix会搞很多个小小的线程池，比如订单服务请求库存服务是一个线程池，请求仓储服务是一个线程池，请求积分服务是一个线程池。每个线程池里的线程就仅仅用于请求那个服务。

打个比方：现在很不幸，积分服务挂了，会咋样？

当然会导致订单服务里那个用来调用积分服务的线程都卡死不能工作了啊！但由于订单服务调用库存服务、仓储服务的这两个线程池都是正常工作的，所以这两个服务不会受到任何影响。

这个时候如果别人请求订单服务，订单服务还是可以正常调用库存服务扣减库存，调用仓储服务通知发货。只不过调用积分服务的时候，每次都会报错。但是如果积分服务都挂了，每次调用都要去卡住几秒钟干啥呢？有意义吗？当然没有！所以我们直接对积分服务熔断不就得了，比如在5分钟内请求积分服务直接就返回了，不要去走网络请求卡住几秒钟，这个过程，就是所谓的熔断！

那人家又说，兄弟，积分服务挂了你就熔断，好歹你干点儿什么啊！别啥都不干就直接返回啊？没问题，咱们就来个降级：每次调用积分服务，你就在数据库里记录一条消息，说给某某用户增加了多少积分，因为积分服务挂了，导致没增加成功！这样等积分服务恢复了，你可以根据这些记录手工加一下积分。这个过程，就是所谓的降级

	- Zuul

		- 如果前端、移动端要调用后端系统，统一从Zuul网关进入，由Zuul网关转发请求给对应的服务，因为前端和客户端是没有微服务客户端的，无法执行服务发现负载均衡和发起调用，只能通过Zuul网关实现，而且有一个网关之后，还有很多好处，比如可以做统一的降级、限流、认证授权、安全，等等

### 缓存中间件

- redis

	- Redis事件驱动模型

		- 文件事件fileEvent

			- 连接建立、接受请求命令、发送响应等

			- 连接处理函数acceptTcpHandler

			- 请求处理函数readQueryFromClient

			- 命令回复处理函数sendReplyToClient

		- 时间事件timeEvent

			- Redis 中定期要执行的统计、key 淘汰、缓冲数据写出、rehash等

			- 时间事件的处理是在事件循环中的 aeProcessEvents 中进行

				- 首先遍历所有的时间事件

				- 比较事件的时间和当前时间，找出可执行的时间事件

				- 然后执行时间事件的 timeProc 函数

				- 执行完毕后，对于周期性时间，设置时间新的执行时间；对于单次性时间，设置事件的 ID为 -1，后续在事件循环中，下一次执行 aeProcessEvents 的时候从链表中删除

		- 连接 socket-》IO 多路复用程序-》文件事件分派器-》事件处理器

			- 远程客户端连接到 redis 后，redis服务端会为远程客户端创建一个 redisClient 作为代理

			- 读取嵌套字中的数据，写入 querybuf

			- 解析 querybuf 中的命令，记录到 argc 和 argv 中

			- 根据 argv[0] 查找对应的 recommand

			- 执行 recommand 对应的实现函数

			- 执行以后将结果存入 buf & bufpos & reply 中，返回给调用方

			- 返回给调用方。返回数据的时候，会控制写入数据量的大小，如果过大会分成若干次。保证 redis 的相应时间。

			-  

			- reactor事件驱动模式

				- reactor模型

					-  

						- acceptor会new一个读写handler实例绑定到他处理的链接来处理该链接的读写操作，后面该链接的读写的就调用这两个实例来处理

						- acceptor是特殊的handler

				- 单reactor单线程

					-  

				- 单reactor多线程

					-  

				- 多reactor多线程

					-  

		- 如果同一个 socket，同时有读事件和写事件，Redis 派发器会首先派发处理读事件，然后再派发处理写事件

		- epoll

			- epoll_create

				- 创建一个epoll句柄

				- 打开io通道

			- epoll_ctl

				- 注册事件

				- typedef union epoll_data {
    void *ptr;
    int fd; //关联句柄，就是要监视的句柄，不是epoll的句柄
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};

			- epoll_wait

				- ET触发

				- LT触发

			- close

			-  

				- 红黑树

					- 节点不是红色就是黑色，根节点是黑色

					- 红黑树的叶子节点并非传统的叶子节点，红黑树的叶子节点是null节点（空节点）且为黑色

					- 同一路径，不存在连续的红色节点

					- 每个节点到叶子节点的所有路径，都包含相同数目的黑色节点

			- server.c

				- #include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/select.h>
#include <sys/epoll.h>
 
int main()
{
    // 1.创建套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket");
        exit(0);
    }
    // 2. 绑定 ip, port
    struct sockaddr_in addr;
    addr.sin_port = htons(1234);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    int ret = bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    if(ret == -1)
    {
        perror("bind");
        exit(0);
    }
    // 3. 监听
    ret = listen(lfd, 100);
    if(ret == -1)
    {
        perror("listen");
        exit(0);
    }
    
    // 4. 创建epoll树
    int epfd = epoll_create(1000);//1000并没有什么意义
    if(epfd == -1)
    {
        perror("epoll_create");
        exit(-1);
    }
    //5、将用于监听的lfd挂的epoll树上（红黑树）
    struct epoll_event ev;//这个结构体记录了检测什么文件描述符的什么事件
    ev.events = EPOLLIN;
    ev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &ev);//ev里面记录了检测lfd的什么事件
    // 循环检测 委托内核去处理
    struct epoll_event events[1024];//当内核检测到事件到来时会将事件写到这个结构体数组里
    while(1)
    {
        int num = epoll_wait(epfd, events, sizeof(events)/sizeof(events[0]), -1);//最后一个参数表示阻塞
        //遍历事件，并进行处理
        for(int i=0; i<num; i++)
        {
            if(events[i].data.fd == lfd)//有连接请求到来
            {
                struct sockaddr_in client_sock;
                int len = sizeof(client_sock);
                int connfd = accept(lfd, (struct sockaddr *)&client_sock, &len);
                if(connfd == -1)
                {
                    perror("accept");
                    exit(-1);
                }
                char ip[32]={0};
                inet_ntop(AF_INET, &(client_sock.sin_addr.s_addr), ip, sizeof(ip));
                printf("a new client connected! ip:%s  port:%d\n", ip, ntohs(client_sock.sin_port));
                //将用于通信的文件描述符挂到epoll树上
                ev.data.fd = connfd;
                ev.events = EPOLLIN;
                epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);
            }
            else//通信
            {
                //通信也有可能是写事件
                if(events[i].events & EPOLLOUT)
                {
                    //这里先忽略写事件
                    continue;
                }
                char buf[1024]={0};
                struct sockaddr_in peerAddr;//用于保存当前通信的客户端的信息
                int peerLen;
                char ip[32]={0};
                inet_ntop(AF_INET, &(peerAddr.sin_addr), ip, sizeof(ip));
                getpeername(events[i].data.fd, (struct sockaddr *)&peerAddr, &peerLen); //获取对应客户端相应信息
                int count = read(events[i].data.fd, buf, sizeof(buf));
                if(count == 0)//客户端关闭了连接
                {
                    printf("%s(%d) 客户端关闭了连接。。。。\n",ip, ntohs(peerAddr.sin_port));
                    //将对应的文件描述符从epoll树上取下
                    close(events->data.fd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, events->data.fd, NULL);
                }
                else
                {
                    if(count == -1)
                    {
                        perror("read");
                        exit(-1);
                    }
                    else
                    {
                        //正常通信
                        printf("%s:%d say: %s\n",ip , ntohs(peerAddr.sin_port), buf);
                        //printf("client say:%s\n",buf);
                        write(events[i].data.fd, buf, strlen(buf)+1);
                    }
                }
            }
        }
    }
    close(lfd);
    return 0;
}

			- client.c

				- #include <stdio.h>
#include <string.h> 
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
int main(){
    //创建套接字
    int sock = socket(AF_INET, SOCK_STREAM, 0);
	printf("create socket success!\n");
    //向服务器（特定的IP和端口）发起请求
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));  //每个字节都用0填充
    serv_addr.sin_family = AF_INET;  //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");  //具体的IP地址
    serv_addr.sin_port = htons(1234);  //端口
    if(connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr))<0){
		printf("connect failed!!!\n");
		return 0;
	}
	printf("connect success!!!\n");
    //读取服务器传回的数据
    char buffer[100];
    //recv(sock, buffer, sizeof(buffer),0);
   
    //printf("Message form server: %s\n", buffer);
    while(1){
		memset(buffer,0,sizeof(buffer));
		fgets(buffer,sizeof(buffer),stdin);//fgets会读取换行符\n
		buffer[strlen(buffer)-1]='\0';//去掉换行符
		send(sock, buffer, strlen(buffer)+1,0);
		//sleep(1);
		recv(sock, buffer, sizeof(buffer),0);
		printf("Message form server: %s\n", buffer);
	}
    //关闭套接字
    close(sock);
    return 0;
}

		- [源代码分析](https://blog.csdn.net/xing_hung/article/details/123805728)

	- 基础常识

		- 应用：token生成、session共享、分布式锁、自增id、验证码等

		- 三大文件

			- redis-server、redis-cli、redis-benchmark性能测试工具

		- redis的链接是tcp长连接

		- redis6.0之后的多线程改进

			- ● 单线程性能瓶颈主要在网络IO上。
● 将网络数据读写和协议解析通过多线程的方式来处理 ，对于命令执行来说，仍然使用单线程操作。

	- 数据结构

		- String

			- SDS

				-  

		- Hash

			- ziplist（压缩列表） 、hashtable（哈希表）

				- 压缩列表

					-  

					-  

					-  

		- List

			- ziplist（压缩列表）、linkedlist（链表）

		- Set

			- intset（整数集合）、hashtable（哈希表）

		- zset

			- ziplist（压缩列表）、skiplist（跳跃表）

				- 跳表

					-  

		- 新数据结构

			- Geospatial

			- Hyperloglog

			- Bitmap

		- redis的序列化协议

			- RESP，英文全称是Redis Serialization Protocol

		- KV存储

			- Redis 作为一个K-V的内存数据库，它使用用一张全局的哈希来保存所有的键值对

				- 哈希冲突链上的元素只能通过指针逐一查找再操作。当往哈希表插入数据很多，冲突也会越多，冲突链表就会越长，那查询效率就会降低了。为了保持高效，Redis 会对哈希表做rehash操作，也就是增加哈希桶，减少冲突。为了rehash更高效，Redis还默认使用了两个全局哈希表，一个用于当前使用，称为主哈希表，一个用于扩容，称为备用哈希表

	- 性能分析

		- 数据结构、纯内存、单线程、IO多路复用、虚拟内存

			- 虚拟内存

				- 虚拟内存机制就是暂时把不经常访问的数据(冷数据)从内存交换到磁盘中，从而腾出宝贵的内存空间用于其它需要访问的数据(热数据)。通过VM功能可以实现冷热数据分离，使热数据仍在内存中、冷数据保存到磁盘。这样就可以避免因为内存不足而造成访问速度下降的问题。

			- 10W~100Wqps

		- 缓存问题

			- 穿透

				- 指查询一个一定不存在的数据

					- 1.如果是非法请求，我们在API入口，对参数进行校验，过滤非法值。

					- 2.如果查询数据库为空，我们可以给缓存设置个空值，或者默认值。但是如有有写请求进来的话，需要更新缓存哈，以保证缓存一致性，同时，最后给缓存设置适当的过期时间。（业务上比较常用，简单有效）

					- 3.使用布隆过滤器快速判断数据是否存在。即一个查询请求过来时，先通过布隆过滤器判断值是否存在，存在才继续往下查

						- 布隆过滤器

							- 布隆过滤器原理是？假设我们有个集合A，A中有n个元素。利用k个哈希散列函数，将A中的每个元素映射到一个长度为a位的数组B中的不同位置上，这些位置上的二进制数均设置为1。如果待检查的元素，经过这k个哈希散列函数的映射后，发现其k个位置上的二进制数全部为1，这个元素很可能属于集合A，反之，一定不属于集合A。

							- 布隆过滤器是一种占用空间很小的数据结构，它由一个很长的二进制向量和一组Hash映射函数组成，它用于检索一个元素是否在一个集合中，空间效率和查询时间都比一般的算法要好的多，缺点是有一定的误识别率和删除困难

			- 击穿

				- 指热点key在某个时间点过期的时候

					- 1.使用互斥锁方案。缓存失效时，不是立即去加载db数据，而是先使用某些带成功返回的原子操作命令，如(Redis的setnx）去操作，成功的时候，再去加载db数据库数据和设置缓存。否则就去重试获取缓存。

					- 2. “永不过期”，是指没有设置过期时间，但是热点数据快要过期时，异步线程去更新和设置过期时间。

				- 热点key

					- Redis集群扩容：增加分片副本，均衡读流量；

					- 将热key分散到不同的服务器中；

					- 使用二级缓存，即JVM本地缓存,减少Redis的读请求。

			- 雪崩

				- 指缓存中数据大批量到过期时间

					- 缓存雪奔一般是由于大量数据同时过期造成的，对于这个原因，可通过均匀设置过期时间解决，即让过期时间相对离散一点。如采用一个较大固定值+一个较小的随机值，5小时+0到1800秒酱紫。

					- Redis 故障宕机也可能引起缓存雪奔。这就需要构造Redis高可用集群啦。

					- 双缓存方案

			- 内存淘汰策略

				-  

	- redis事务机制

		- Redis通过MULTI、EXEC、WATCH等一组命令集合，来实现事务机制。事务支持一次执行多个命令，一个事务中所有命令都会被序列化。在事务执行过程，会按照顺序串行化执行队列中的命令，其他客户端提交的命令请求不会插入到事务执行命令序列中。

			- MULTI

			- EXEC

			- DISCARD

			- WATCH

	- 缓存一致性问题

		- 延迟双删

			- 延时双删流程

先删除缓存
再更新数据库
休眠一会（比如1秒），再次删除缓存。

		- 消息重试

			-  

		- binlog异步删除

			-  

	- 持久化

		-  

			- AOF

				-  

					- 采用日志的形式来记录每个写操作，追加到文件中

						- 优点

							- 数据的一致性和完整性更高

						- 缺点

							- AOF记录的内容越多，文件越大，数据恢复变慢

			- RDB

				- RDB，就是把内存数据以快照的形式保存到磁盘上

					- 优点

						- 适合大规模的数据恢复场景，如备份，全量复制等

							-  

					- 缺点

						- 没办法做到实时持久化/秒级持久化

	- 可用性

		- 主从模式

			- 首次同步过程

				-  

			- 实时同步过程

				-  

		- 哨兵模式

			-  

		- 集群模式

			- 分布式存储算法

				- Hash Slot插槽算法

					- 插槽算法把整个数据库被分为16384个slot（槽），每个进入Redis的键值对，根据key进行散列，分配到这16384插槽中的一个。

						- 但一定要注意的是，对于槽位的转移和分派，Redis集群是不会自动进行的，而是需要人工配置的

							- 所以Redis集群的高可用是依赖于节点的主从复制与主从间的自动故障转移

					- 另外一个分布式存储算法是：一致性hash（如memcach）

						- 一致性hash算法会建立一个有2^32个槽点(0 - 2^32-1)的hash环，假设现在有A、B、C三台服务器，以A为栗，会进行hash(A)%2^32，得到一个0 - 2^32-1之间的数，然后映射到hash环上

							-  

						- 接下来，我们同样以csdn.jpg为例，我们照样算出hash(csdn.jpg)%2^32的值，然后映射到hash环上，然后以该点出发，顺时针遇到的第一个服务器，即为数据即将存储的服务器

							-  

						- 这时增加了服务器D又会发生什么事情呢?
如果这个时候在A - C之间插入了服务器D，请求获取getKey(csdn.jpg)时，顺时针获得的服务器是D，从D上获取数据理所当然会失败，因为数据存在A上缓存。这样看缓存好像还是失效了。

						- hash偏斜问题

							- A、B、C服务节点，如果像上图那样接近于将hash环平均分配那固然理想，但是如果他们hash值十分相近，会发生什么呢

								-  

									- 如图这种情况称之为hash偏斜，在这种情况下，大部分数据都会分部在C-A段，这个时候去A节点被删除，会有大量请求涌向B节点，给B节点带来巨大的压力，同时这部分缓存也会全部失效，有可能引发缓存雪崩

							- 解决办法

								- 所以引入了虚拟节点的概念，以A节点为例，虚拟构造出(A0,A1,A2....AN)，只要是落在这些虚拟节点上的数据，都存入A节点。读取时也相同，顺时针获取的是A0虚拟节点，就到A节点上获取数据，这样就能解决数据分布不均的问题

			- 主要协议

				- 各个节点之间的通信协议

					- Gossip协议

						- 每个节点是通过集群总线(cluster bus) 与其他的节点进行通信的。通讯时，使用特殊的端口号，即对外服务端口号加10000。例如如果某个node的端口号是6379，那么它与其它nodes通信的端口号是 16379。nodes 之间的通信采用特殊的二进制协议。

							- MEET：通过「cluster meet ip port」命令，已有集群的节点会向新的节点发送邀请，加入现有集群，然后新节点就会开始与其他节点进行通信；
PING：节点按照配置的时间间隔向集群中其他节点发送 ping 消息，消息中带有自己的状态，还有自己维护的集群元数据，和部分其他节点的元数据；
PONG: 节点用于回应 PING 和 MEET 的消息，结构和 PING 消息类似，也包含自己的状态和其他信息，也可以用于信息广播和更新；
FAIL: 节点 PING 不通某节点后，会向集群所有节点广播该节点挂掉的消息。其他节点收到消息后标记已下线。

								- 定时同步

									-  

								- 新节点加入

									-  

								- 节点疑似下线和真正下线

									- Redis Cluster 中的节点会定期检查已经发送 PING 消息的接收方节点是否在规定时间 ( cluster-node-timeout ) 内返回了 PONG 消息，如果没有则会将其标记为疑似下线状态，也就是 PFAIL 状态

									- 节点一会通过 PING 消息，将节点二处于疑似下线状态的信息传递给其他节点，例如节点三

									- 随着时间的推移，如果节点十 (举个例子) 也因为 PONG 超时而认为节点二疑似下线了，并且发现自己维护的节点二的 clusterNode 的 fail_reports 中有半数以上的主节点数量的未过时的将节点二标记为 PFAIL 状态报告日志，那么节点十将会把节点二将被标记为已下线 FAIL 状态，并且节点十会立刻向集群其他节点广播主节点二已经下线的 FAIL 消息，所有收到 FAIL 消息的节点都会立即将节点二状态标记为已下线。

				- 单节点主从选举协议

					- raft

### 消息中间件

- kafka

	-  

		- kafka的消息文件

			- 文件的命名是以该segment最小offset来命名的

			-  

			- message结构

				- 1、 offset：offset是一个占8byte的有序id号，它可以唯一确定每条消息在parition内的位置！
2、 消息大小：消息大小占用4byte，用于描述消息的大小。
3、 消息体：消息体存放的是实际的消息数据（被压缩过），占用的空间根据具体的消息而不一样。

			- 存储策略

				- 无论消息是否被消费，kafka都会保存所有的消息。那对于旧数据有什么删除策略呢？
1、 基于时间，默认配置是168小时（7天）。
2、 基于大小，默认配置是1073741824。
　　需要注意的是，kafka读取特定消息的时间复杂度是O(1)，所以这里删除过期的文件并不会提高kafka的性能

			- 消费者是怎么记录自己消费到哪里了呢

				- 那每个消费者又是怎么记录自己消费的位置呢？在早期的版本中，消费者将消费到的offset维护zookeeper中，consumer每间隔一段时间上报一次，这里容易导致重复消费，且性能不好！在新的版本中消费者消费到的offset已经直接维护在kafk集群的__consumer_offsets这个topic中！

		- 工作流程

			-  

			- 写入负载原则

				- 1、 partition在写入的时候可以指定需要写入的partition，如果有指定，则写入对应的partition。
2、 如果没有指定partition，但是设置了数据的key，则会根据key的值hash出一个partition。
3、 如果既没指定partition，又没有设置key，则会轮询选出一个partition。

					- 生产者可以指定partition写入，消费者也可以指定partition消费

			- 保证消息不丢失

				- 在生产者向队列写入数据的时候可以设置参数来确定是否确认kafka接收到数据，这个参数可设置的值为0、1、all。
0代表producer往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最高。
1代表producer往集群发送数据只要leader应答就可以发送下一条，只确保leader发送成功。
all代表producer往集群发送数据需要所有的follower都完成从leader的同步才会发送下一条，确保leader发送成功和所有的副本都完成备份。安全性最高，但是效率最低。

					- kafka中与leader副本保持一定同步程度的副本（包括leader）组成ISR。与leader滞后太多的副本组成OSR。分区中所有的副本通称为AR。
ISR : 速率和leader相差低于10秒的follower的集合
OSR : 速率和leader相差大于10秒的follower
AR : 全部分区的follower

						- ISR(InSyncRepli)、OSR(OutSyncRepli)、AR(AllRepli)

							- 只要在 replica.lag.time.max.ms 时间内 follower 有同步消息，即认为该 follower 处于 ISR 中

								- follower是主动到leader上拉取消息的

				- 消息丢失问题也是不可避免的

					- 比如新leader故障后新选的follower有数据没有同步完成，导致消息丢失

			- 消息的顺序性

				- producer采用push模式将数据发布到broker，每条消息追加到分区中，顺序写入磁盘，所以保证同一分区内的数据是有序的

			- 消息重复消费

				- 很多情况下都会导致消息重复消费的问题，需要consumer确保幂等

					- 比如消息消费后，消费者异常导致消费成功ack没有返回消费offset没有后移

					- leader没月将offset同步到follower就发生故障，导致新leader offset没有后移，从而重复消费

			- zookeeper

				- leader 检测、leader选举、分布式同步、配置管理、识别新节点何时离开或连接、集群、节点实时状态，老版本还负责消息offset的记录

			- Kafka Rebalance
它本质上是一组协议，它规定了一个 consumer group 是如何达成一致来分配订阅 topic 的所有分区的

				-  

					- 确定消费者所属的GroupCoordinator所在的broker，如果消费者已经保存了GroupCordinator信息，可以进入下一个阶段，否则需要查找_consumer_offset对应的分区的leader副本所对应的broker，
具体查找 Group Coordinator 的方式是先根据消费组 groupid 的晗希值计算＿consumer_offsets
中的分区编号，具体算法如代码清单


image.png

以此broker作为GroupCordinator角色，又扮演分区分配和组内消费者位移的角色。

				-  

				- 每个consumer group会从kafka broker中选出一个作为组协调者

				- kafka新版本提供了三种rebalance分区分配策略：
range
round-robin
sticky

			- 数据一致性问题

				-  

					-  

						- HW：高水位，指消费者只能拉取到这个offset之前的数据

						- LEO：标识当前日志文件中下一条待写入的消息的offset，大小等于当前日志文件最后一条消息的offset+1.

				- leader选举

					- Controller leader（borker集群中以一个leader）
当broker启动的时候，都会创建KafkaController对象，但是集群中只能有一个leader对外提供服务，这些每个节点上的KafkaController会在指定的zookeeper路径下创建临时节点，只有第一个成功创建的节点的KafkaController才可以成为leader，其余的都是follower。当leader故障后，所有的follower会收到通知，再次竞争在该路径下创建节点从而选举新的leader

Partition leader 
由controller leader执行，从Zookeeper中读取当前分区的所有ISR(in-sync replicas)集合

						- 当前， Kafka 有 4 种分区 Leader 选举策略。
OfflinePartition Leader 选举：每当有分区上线时，就需要执行 Leader 选举。所谓的分区上 线，可能是创建了新分区，也可能是之前的下线分区重新上线。这是最常见的分区 Leader 选 举场景。 ReassignPartition Leader 选举：当你手动运行 kafka-reassign-partitions 命令，或者是 调用 Admin 的 alterPartitionReassignments 方法执行分区副本重分配时，可能触发此类选 举。假设原来的 AR 是[1，2，3]，Leader 是 1，当执行副本重分配后，副本集 合 AR 被设置 成[4，5，6]，显然，Leader 必须要变更，此时会发生 Reassign Partition Leader 选举。

PreferredReplicaPartition Leader 选举：当你手动运行 kafka-preferred-replicaelection 命令，或自动触发了 Preferred Leader 选举时，该类策略被激活。所谓的 Preferred Leader，指的是 AR 中的第一个副本。比如 AR 是[3，2，1]，那么， Preferred Leader 就是 3。

ControlledShutdownPartition Leader 选举：当 Broker 正常关闭时，该 Broker 上 的所有 Leader 副本都会下线，因此，需要为受影响的分区执行相应的 Leader 选举。


		- 高效的原因

			- 1.kafka是分布式的消息队列
2.对log文件进行了segment,并对segment创建了索引
3.(对于单节点)使用了顺序读写,速度能够达到600Mps，10w+qps
4.引用了zero拷贝,在os系统就完成了读写操做

### 数据库中间件

- mysql

	- 架构

		-  

			- 链接器

			- 缓存

			- 分析器

			- 优化器

				- MySQL Query Optimizer

					- MySQL 中有专门负责优化 SELECT 语句的优化器模块，主要功能：通过计算分析系统中收集到的统计信息，为客户端请求的 Query 提供他认为最优的执行计划（他认为最优的数据检索方式，但不见得是 DBA 认为是最优的，这部分最耗费时间）

					- 当客户端向 MySQL 请求一条 Query，命令解析器模块完成请求分类，区别出是 SELECT 并转发给 MySQL Query Optimize r时，MySQL Query Optimizer 首先会对整条 Query 进行优化，处理掉一些常量表达式的预算，直接换算成常量值。并对 Query 中的查询条件进行简化和转换，如去掉一些无用或显而易见的条件、结构调整等。然后分析 Query 中的 Hint 信息（如果有），看显示 Hint 信息是否可以完全确定该 Query 的执行计划。如果没有 Hint 或 Hint 信息还不足以完全确定执行计划，则会读取所涉及对象的统计信息，根据 Query 进行写相应的计算分析，然后再得出最后的执行计划。

						- 何谓 hint
我们知道在执行一条SQL语句时，MySQL会生成一个执行计划，而hint就是告诉查询优化器需要按照我们告诉它的方式来生成执行计划。
Hint可基于表的连接顺序、方法、访问路径、并行度等规则对DML（数据操纵语言，Data Manipulation Language）语句产生作用，范围如下：

使用的优化器类型；
基于代价的优化器的优化目标，是all_rows还是first_rows；
表的访问路径，是全表扫描，还是索引扫描，还是直接用rowid；
表之间的连接类型；
表之间的连接顺序；
语句的并行程度；

常用 hint
强制索引 FORCE INDEX
SELECT * FROM tbl FORCE INDEX (FIELD1) …
忽略索引 IGNORE INDEX
SELECT * FROM tbl IGNORE INDEX (FIELD1, FIELD2) …
关闭查询缓冲 SQL_NO_CACHE
SELECT SQL_NO_CACHE field1, field2 FROM tbl;
需要查询实时数据且频率不高时，可以考虑把缓冲关闭，即不论此SQL是否曾被执行，MySQL都不会在缓冲区中查找。
强制查询缓冲 SQL_CACHE
SELECT SQL_CACHE * FROM tbl;
功能同上一条相反，但仅在my.ini中的query_cache_type设为2时起作用。
优先操作 HIGH_PRIORITY
HIGH_PRIORITY可以使用在select和insert操作中，让MYSQL知道，这个操作优先进行。
SELECT HIGH_PRIORITY * FROM tbl;
滞后操作 LOW_PRIORITY
LOW_PRIORITY可以使用在insert和update操作中，让mysql知道，这个操作滞后。
update LOW_PRIORITY tbl set field1= where field1= …
延时插入 INSERT DELAYED
INSERT DELAYED INTO tbl set field1= …
指客户端提交插入数据申请，MySQL返回OK状态却并未实际执行，而是存储在内存中排队，当mysql有空余时再插入。
一个重要的好处是，来自多个客户端的插入请求被集中在一起，编写入一个块，比独立执行许多插入要快很多。
坏处是，不能返回自增ID，以及系统崩溃时，MySQL还未来得及被插入的数据将会丢失。
强制连接顺序 STRAIGHT_JOIN
SELECT tbl.FIELD1, tbl2.FIELD2 FROM tbl STRAIGHT_JOIN tbl2 WHERE …
由上面的SQL语句可知，通过STRAIGHT_JOIN强迫MySQL按tbl、tbl2的顺序连接表。如果你认为按自己的顺序比MySQL推荐的顺序进行连接的效率高的话，就可以通过STRAIGHT_JOIN来确定连接顺序。
不常用
强制使用临时表 SQL_BUFFER_RESULT
SELECT SQL_BUFFER_RESULT * FROM tbl WHERE …
当我们查询的结果集中的数据比较多时，可以通过SQL_BUFFER_RESULT.选项强制将结果集放到临时表中，这样就可以很快地释放MySQL的表锁（这样其它的SQL语句就可以对这些记录进行查询了），并且可以长时间地为客户端提供大记录集。
分组使用临时表 SQL_BIG_RESULT和SQL_SMALL_RESULT
SELECT SQL_BUFFER_RESULT FIELD1, COUNT(*) FROM tbl GROUP BY FIELD1;
对SELECT语句有效，告诉MySQL优化去对GROUP BY和DISTINCT查询如何使用临时表排序，SQL_SMALL_RESULT表示结果集很小，可以直接在内存的临时表排序；反之则很大，需要使用磁盘临时表排序。
SQL_CALC_FOUND_ROWS
它其实不是优化器提示，也不影响优化器的执行计划，但会让mysql返回的结果集中包含本次操作影响的总行数，需与 FOUND_ROWS() 联用。
SQL_CALC_FOUND_ROWS 通知MySQL将本次处理的行数记录下来； FOUND_ROWS() 用于取出被记录的行数，可以应用到分页场景。
一般的分页写法为：先查总数，计算页数，再查询某一页的详情。
SELECT COUNT(*) from tbl WHERE …
SELECT * FROM tbl WHERE … limit m,n
但借助SQL_CALC_FOUND_ROWS，可以简化成如下写法：
SELECT SQL_CALC_FOUND_ROWS * FROM tbl WHERE … limit m,n;
SELECT FOUND_ROWS();
第二条SELECT将返回第一条SELECT不带limit时的总行数，如此只需执行一次较耗时的复杂查询就可同时得到总行数。
LOCK IN SHARE MODE、 FOR UPDATE
同样的，这俩也不是优化提示，是控制SELECT语句的锁机制，只对行级锁有效，即InnoDB支持。
概念和区别
SELECT ... LOCK IN SHARE MODE添加的是IS锁(意向共享锁)，即在符合条件的rows上都加了共享锁，其他session可读取记录，亦可继续添加IS锁，但无法修改，直到这个加锁的session done(否则直接锁等待超时)。
SELECT ... FOR UPDATE 添加的是IX锁(意向排它锁)，即符合条件的rows上都加了排它，其他session无法给这些记录添加任何S锁或X锁。如果不存在一致性非锁定读的话，则其他session是无法读取和修改这些记录的，但innodb有非锁定读(快照读不需要加锁)。
因此，for update的加锁方式只是比lock in share mode的方式多阻塞了select...lock in share mode的查询方式，并不会阻塞快照读。
应用场景
LOCK IN SHARE MODE的适用于两张存在关系的表的写场景，以mysql官方例子来说，一个表是child表，一个是parent表，假设child表的某一列child_id映射到parent表的c_child_id列，从业务角度讲，此时直接insert一条child_id=100记录到child表是存在风险的，因为insert的同时可能存在parent表执行了删除c_child_id=100的记录，业务数据有不一致的风险。正确方法是先执行select * from parent where c_child_id=100 lock in share mode，锁定parent表的这条记录，然后执行insert into child(child_id) values (100)。

			- 执行器

				- count(*) 和 count(1)和count(列名)区别 

					- 执行效果：
1、count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL。

2、count(1)包括了忽略所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL。

3、count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空。

执行效率：
列名为主键，count(列名)会比count(1)快  

列名不为主键，count(1)会比count(列名)快  

如果表多个列并且没有主键，则 count（1） 的执行效率优于 count（*）  

如果有主键，则 select count（主键）的执行效率是最优的  

如果表只有一个字段，则 select count（*）最优

			- 存储引擎

				-  InnoDB、MyISAM、Memory、NDB

					- 一个数据库中多个表可以使用不同引擎以满足各种性能和实际需求

						- 查看存储引擎

							- -- 查看支持的存储引擎
SHOW ENGINES

-- 查看默认存储引擎
SHOW VARIABLES LIKE 'storage_engine'

--查看具体某一个表所使用的存储引擎，这个默认存储引擎被修改了！
show create table tablename

--准确查看某个数据库中的某一表所使用的存储引擎
show table status like 'tablename'
show table status from database where name="tablename"

						- 设置存储引擎

							- -- 建表时指定存储引擎。默认的就是INNODB，不需要设置
CREATE TABLE t1 (i INT) ENGINE = INNODB;
CREATE TABLE t2 (i INT) ENGINE = CSV;
CREATE TABLE t3 (i INT) ENGINE = MEMORY;

-- 修改存储引擎
ALTER TABLE t ENGINE = InnoDB;

-- 修改默认存储引擎，也可以在配置文件my.cnf中修改默认引擎
SET default_storage_engine=NDBCLUSTER;

					- 可以从运行中的 MySQL 服务器加载或卸载存储引擎

					- 对比

						- MyISAM物理文件结构

							- .frm文件：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
.MYD (MYData) 文件：MyISAM 存储引擎专用，用于存储MyISAM 表的数据
.MYI (MYIndex)文件：MyISAM 存储引擎专用，用于存储MyISAM 表的索引相关信息

						- InnoDB 物理文件结构

							- .frm 文件：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
.ibd 文件或 .ibdata 文件： 这两种文件都是存放 InnoDB 数据的文件，之所以有两种文件形式存放 InnoDB 的数据，是因为 InnoDB 的数据存储方式能够通过配置来决定是使用共享表空间存放存储数据，还是用独享表空间存放存储数据。
独享表空间存储方式使用.ibd文件，并且每个表一个.ibd文件 共享表空间存储方式使用.ibdata文件，所有表共同使用一个.ibdata文件（或多个，可自己配置）

						- 功能

							- InnoDB 支持事务，MyISAM 不支持事务。这是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

							- InnoDB 支持外键，而 MyISAM 不支持。对一个包含外键的 InnoDB 表转为 MYISAM 会失败；

							- InnoDB 是聚簇索引，MyISAM 是非聚簇索引。聚簇索引的文件存放在主键索引的叶子节点上，因此 InnoDB 必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大。而 MyISAM 是非聚集索引，数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的。

							- InnoDB 不保存表的具体行数，执行select count(*) from table 时需要全表扫描。而 MyISAM 用一个变量保存了整个表的行数，执行上述语句时只需要读出该变量即可，速度很快；

							- InnoDB 最小的锁粒度是行锁，MyISAM 最小的锁粒度是表锁。一个更新语句会锁住整张表，导致其他查询和更新都会被阻塞，因此并发访问受限。这也是 MySQL 将默认存储引擎从 MyISAM 变成 InnoDB 的重要原因之一；

							- InnoDB不仅缓存索引还要缓存真实数据，对内存要求较高，而且内存大小对性能有决定性的影响，MyISAM缓存只缓存索引，不缓存真实数据

							-  

					- 一张表，里面有ID自增主键，当insert了17条记录之后，删除了第15,16,17条记录，再把Mysql重启，再insert一条记录，这条记录的ID是18还是15 

						- 如果表的类型是MyISAM，那么是18。因为MyISAM表会把自增主键的最大ID 记录到数据文件中，重启MySQL自增主键的最大ID也不会丢失；

						- 如果表的类型是InnoDB，那么是15。因为InnoDB 表只是把自增主键的最大ID记录到内存中，所以重启数据库或对表进行OPTION操作，都会导致最大ID丢失

				- .frm文件

					- 在其数据目录对应的数据库目录下都有对应表的 .frm 文件，.frm 文件是用来保存每个数据表的元数据(meta)信息，包括表结构的定义等，与数据库存储引擎无关，也就是任何存储引擎的数据表都必须有.frm文件，命名方式为 数据表名.frm，如user

				- 数据类型

					- 五大类

						- 整数类型：BIT、BOOL、TINY INT、SMALL INT、MEDIUM INT、 INT、 BIG INT

						- 浮点数类型：FLOAT、DOUBLE、DECIMAL

						- 字符串类型：CHAR、VARCHAR、TINY TEXT、TEXT、MEDIUM TEXT、LONGTEXT、TINY BLOB、BLOB、MEDIUM BLOB、LONG BLOB

							- CHAR 和 VARCHAR 

								- 共同点

									- char(n)，varchar(n)中的n都代表字符的个数

									- 超过char，varchar最大长度n的限制后，字符串会被截断

								- 不同点

									- char不论实际存储的字符数都会占用n个字符的空间，而varchar只会占用实际字符应该占用的字节空间加1（实际长度length，0<=length<255）或加2（length>255）。因为varchar保存数据时除了要保存字符串之外还会加一个字节来记录长度（如果列声明长度大于255则使用两个字节来保存长度）

									- 能存储的最大空间限制不一样：char的存储上限为255字节

									- char在存储时会截断尾部的空格，而varchar不会。

							- BLOB和TEXT

								- BLOB是一个二进制对象，可以容纳可变数量的数据。有四种类型的BLOB：TINYBLOB、BLOB、MEDIUMBLO和 LONGBLOB

								- TEXT是一个不区分大小写的BLOB。四种TEXT类型：TINYTEXT、TEXT、MEDIUMTEXT 和 LONGTEXT。

								- BLOB 保存二进制数据，TEXT 保存字符数据

						- 日期类型：Date、DateTime、TimeStamp、Time、Year

						- 其他数据类型：BINARY、VARBINARY、ENUM、SET、Geometry、Point、MultiPoint、LineString、MultiLineString、Polygon、GeometryCollection等

					- 列的字符串类型可以是什么

						- 意思是一个的直接用字符串输入，可以映射到什么类型上去，如 select * from a = ‘123’ 那么a可以是什么类型

							- 字符串类型是：SET、BLOB、ENUM、CHAR、TEXT、VARCHAR

				- 索引

					- 平常说的索引，没有特别指明的话，就是B+树（多路搜索树，不一定是二叉树）结构组织的索引。其中聚集索引，次要索引，覆盖索引，复合索引，前缀索引，唯一索引默认都是使用B+树索引，统称索引。此外还有哈希索引等

						- 创建索引：CREATE [UNIQUE] INDEX indexName ON mytable(username(length));
如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length

							- 分类

								- 数据结构

									- B+树索引

										- InnoDB和MyISAM

									- Hash索引

										- MySQL目前有Memory引擎和NDB引擎支持Hash索引。

											- 为何不采用Hash方式？

												- 因为Hash索引底层是哈希表，哈希表是一种以key-value存储数据的结构，所以多个数据在存储关系上是完全没有任何顺序关系的，所以，对于区间查询是无法直接通过索引查询的，就需要全表扫描。所以，哈希索引只适用于等值查询的场景。而B+ Tree是一种多路平衡查询树，所以他的节点是天然有序的（左子节点小于父节点、父节点小于右子节点），所以对于范围查询的时候不需要做全表扫描。

												- 哈希索引不支持多列联合索引的最左匹配规则，如果有大量重复键值得情况下，哈希索引的效率会很低，因为存在哈希碰撞问题

									- Full-Text全文索引

										- 全文索引也是MyISAM的一种特殊索引类型，主要用于全文索引，InnoDB从MYSQL5.6版本提供对全文索引的支持。
它用于替代效率较低的LIKE模糊匹配操作，而且可以通过多字段组合的全文索引一次性全模糊匹配多个字段。
同样使用B-Tree存放索引数据，但使用的是特定的算法，将字段数据分割后再进行索引（一般每4个字节一次分割），索引文件存储的是分割前的索引字符串集合，与分割后的索引信息，对应Btree结构的节点存储的是分割后的词信息以及它在分割前的索引字符串集合中的位置。

									- R-Tree位置索引

										- 空间索引是MyISAM的一种特殊索引类型，主要用于地理空间数据类型

								- 逻辑分类

									- 主键索引：主键索引是一种特殊的唯一索引，不允许有空值

									- 普通索引或者单列索引：每个索引只包含单个列，一个表可以有多个单列索引

									- 多列索引（复合索引、联合索引）：复合索引指多个字段上创建的索引，只有在查询条件中使用了创建索引时的第一个字段，索引才会被使用。使用复合索引时遵循最左匹配规则

									- 唯一索引或者非唯一索引

									- 空间索引：空间索引是对空间数据类型的字段建立的索引，MYSQL中的空间数据类型有4种，分别是GEOMETRY、POINT、LINESTRING、POLYGON。 MYSQL使用SPATIAL关键字进行扩展，使得能够用于创建正规索引类型的语法创建空间索引。创建空间索引的列，必须将其声明为NOT NULL，空间索引只能在存储引擎为MYISAM的表中创建

					- 首先要明白索引（index）是在存储引擎（storage engine）层面实现的，而不是server层面。不是所有的存储引擎都支持所有的索引类型。即使多个存储引擎支持某一索引类型，它们的实现和行为也可能有所差别。

						- MyISAM 和 InnoDB 存储引擎，都使用 B+Tree的数据结构

					- B树和B+树

						- MyISAM 和 InnoDB 存储引擎，都使用 B+Tree的数据结构，它相对与 B-Tree结构，所有的数据都存放在叶子节点上，且把叶子节点通过指针连接到一起，形成了一条数据链表，以加快相邻数据的检索效率。

						- 磁盘的块和InnoDB的页

							- 系统从磁盘读取数据到内存时是以磁盘块（block）为基本单位的，位于同一个磁盘块中的数据会被一次性读取出来，而不是需要什么取什么。

							- InnoDB 存储引擎中有页（Page）的概念，页是其磁盘管理的最小单位。InnoDB 存储引擎中默认每个页的大小为16KB，可通过参数 innodb_page_size 将页的大小设置为 4K、8K、16K，在 MySQL 中可通过如下命令查看页的大小：show variables like 'innodb_page_size';

							- 而系统一个磁盘块的存储空间往往没有这么大，因此 InnoDB 每次申请磁盘空间时都会是若干地址连续磁盘块来达到页的大小 16KB。InnoDB 在把磁盘数据读入到磁盘时会以页为基本单位，在查询数据时如果一个页中的每条数据都能有助于定位数据记录的位置，这将会减少磁盘I/O次数，提高查询效率。

						- 一棵m阶的B-Tree

							- 每个节点最多有m个孩子

							- 除了根节点和叶子节点外，其它每个节点至少有Ceil(m/2)个孩子

							- 若根节点不是叶子节点，则至少有2个孩子

							- 所有叶子节点都在同一层，且不包含其它关键字信息

							- 每个非终端节点包含n个关键字信息（P0,P1,…Pn, k1,…kn）

							- 关键字的个数n满足：ceil(m/2)-1 <= n <= m-1

							- ki(i=1,…n)为关键字，且关键字升序排序

							- Pi(i=1,…n)为指向子树根节点的指针。P(i-1)指向的子树的所有节点关键字均小于ki，但都大于k(i-1)

							- 如图

								-  

						- 一颗m阶的B+树

							- 在B+Tree中，所有数据记录节点都是按照键值大小顺序存放在同一层的叶子节点上，而非叶子节点上只存储key值信息，这样可以大大加大每个节点存储的key值数量，降低B+Tree的高度

							- 如图

								-  

					- MyISAM索引

						- MyISAM引擎的索引文件和数据文件是分离的。MyISAM引擎索引结构的叶子节点的数据域，存放的并不是实际的数据记录，而是数据记录的地址。索引文件与数据文件分离，这样的索引称为"非聚簇索引"。MyISAM的主索引与辅助索引区别并不大，只是主键索引不能有重复的关键字。

							-  

					- InnoDB索引

						- 我们知道InnoDB索引是聚集索引，它的索引和数据是存入同一个.idb文件中的，因此它的索引结构是在同一个树节点中同时存放索引和数据，如下图中最底层的叶子节点有三行数据，对应于数据表中的id、stu_id、name数据项

							- 主索引

								-  

									- 数据文件本身就是索引文件

									- 表数据文件本身就是按 B+Tree 组织的一个索引结构文件

									- 聚集索引中叶节点包含了完整的数据记录

									- InnoDB 表必须要有主键，并且推荐使用整型自增主键

							- 辅索引

								-  

						- 正如我们上面介绍 InnoDB 存储结构，索引与数据是共同存储的，不管是主键索引还是辅助索引，在查找时都是通过先查找到索引节点才能拿到相对应的数据，如果我们在设计表结构时没有显式指定索引列的话，MySQL 会从表中选择数据不重复的列建立索引，如果没有符合的列，则 MySQL 自动为 InnoDB 表生成一个隐含字段作为主键，并且这个字段长度为6个字节，类型为整型。

						- 为什么非主键索引结构叶子节点存储的是主键值？

							- 保证数据一致性和节省存储空间，可以这么理解：商城系统订单表会存储一个用户ID作为关联外键，而不推荐存储完整的用户信息，因为当我们用户表中的信息（真实名称、手机号、收货地址···）修改后，不需要再次维护订单表的用户数据，同时也节省了存储空间

					- 覆盖索引

						- 就是select的数据列只用从索引中就能够取得，不必读取数据行，MySQL可以利用索引返回select列表中的字段，而不必根据索引再次读取数据文件，换句话说查询列要被所建的索引覆盖。

						- 索引是高效找到行的一个方法，但是一般数据库也能使用索引找到一个列的数据，因此它不必读取整个行。毕竟索引叶子节点存储了它们索引的数据，当能通过读取索引就可以得到想要的数据，那就不需要读取行了。一个索引包含（覆盖）满足查询结果的数据就叫做覆盖索引。
判断标准

						- 使用explain，可以通过输出的extra列来判断，对于一个索引覆盖查询，显示为using index，MySQL查询优化器在执行查询前会决定是否有索引覆盖查询

					- 索引适用情况

						- 哪些情况需要创建索引

							- 主键自动建立唯一索引

							- 频繁作为查询条件的字段

							- 查询中与其他表关联的字段，外键关系建立索引

							- 单键/组合索引的选择问题，高并发下倾向创建组合索引，针对单条查询优化，把查询语句的查询字段建立组合索引可以加快查询速度，高并发场景下降低RT

							- 查询中排序的字段，排序字段通过索引访问大幅提高排序速度

							- 查询中统计或分组字段

						- 哪些情况不要创建索引

							- 表记录太少

							- 经常增删改的表，频繁更新的字段不适合创建索引（会加重IO负担）

							- 数据重复且分布均匀的表字段，只应该为最经常查询和最经常排序的数据列建立索引（如果某个数据类包含太多的重复数据，建立索引没有太大意义）比如 男 女

							- where条件里用不到的字段不创建索引

	- MySQL查询

		- 常见的问题

			- count(*) 和 count(1)和count(列名)区别

				- 效果上

					- count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL

					- count(1)包括了所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL

					- count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空（这里的空不是只空字符串或者0，而是表示null）的计数，即某个字段值为NULL时，不统计。

				- 效率上

					- 多字段条件下

						- 有主键

							- count(主键列名) 效率最高

						- 没有主键

							- count(1)效率最高

					- 单字段条件下

						- count(*)效率最高

			- MySQL中 in和 exists 的区别

				- exists：exists对外表用loop逐条查询，每次查询都会查看exists的条件语句，当exists里的条件语句能够返回记录行时（无论记录行是的多少，只要能返回），条件就为真，返回当前loop到的这条记录；反之，如果exists里的条件语句不能返回记录行，则当前loop到的这条记录被丢弃，exists的条件就像一个bool条件，当能返回结果集则为true，不能返回结果集则为false

				- in：in查询相当于多个or条件的叠加

				- 子查询表大的用exists，子查询表小的用in

			- UNION和UNION ALL的区别

				- UNION在进行表连接后会筛选掉重复的数据记录（效率较低），而UNION ALL则不会去掉重复的数据记录；

				- UNION会按照字段的顺序进行排序，而UNION ALL只是简单的将两个结果合并就返回

		- join

			-  

		- 语句的执行顺序

			-  

	- MySQL事务

		- ACID

			- Atomicity

			- Consistency

			- Isolation

			- Durability

		- 并发事务处理

			- 并发事务处理带来的问题

				- 更新丢失（Lost Update)： 事务A和事务B选择同一行，然后基于最初选定的值更新该行时，由于两个事务都不知道彼此的存在，就会发生丢失更新问题

				- 脏读(Dirty Reads)：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据

					- 重点是事务回滚

				- 不可重复读（Non-Repeatable Reads)：事务 A 多次读取同一数据，事务B在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致

					- 不可重复读的重点是修改：在同一事务中，同样的条件，第一次读的数据和第二次读的数据不一样。（因为中间有其他事务提交了修改）

				- 幻读（Phantom Reads)：幻读与不可重复读类似。它发生在一个事务A读取了几行数据，接着另一个并发事务B插入了一些数据时。在随后的查询中，事务A就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

					- 幻读的重点在于新增或者删除：在同一事务中，同样的条件,，第一次和第二次读出来的记录数不一样。（因为中间有其他事务提交了插入/删除）

						- 解决方案
快照读：通过mvcc，RR的隔离级别解决了幻读问题，因为每次使用的都是同一个readview。
当前读：通过next-key锁（行锁+gap锁），RR隔离级别并不能解决幻读问题。

			- 并发事务处理带来的问题的解决办法

				- “更新丢失”通常是应该完全避免的。但防止更新丢失，并不能单靠数据库事务控制器来解决，需要应用程序对要更新的数据加必要的锁来解决，因此，防止更新丢失应该是应用的责任。

				- “脏读” 、 “不可重复读”和“幻读” ，其实都是数据库读一致性问题，必须由数据库提供一定的事务隔离机制来解决

					- 一种是加锁：在读取数据前，对其加锁，阻止其他事务对数据进行修改。

					- 另一种是数据多版本并发控制（MultiVersion Concurrency Control，简称 MVCC 或 MCC），也称为多版本数据库：不用加任何锁， 通过一定机制生成一个数据请求时间点的一致性数据快照 （Snapshot)， 并用这个快照来提供一定级别 （语句级或事务级） 的一致性读取。从用户的角度来看，好象是数据库可以提供同一数据的多个版本。

		- 事务的隔离级别

			- READ-UNCOMMITTED(读未提交)

			- READ-COMMITTED(读已提交)

				- 普通数据库的默认隔离级别

			- REPEATABLE-READ(可重复读)： 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。

				- 重复读，就是在开始读取数据（事务开启）时，不再允许修改操作

				- 重复读可以解决不可重复读问题。写到这里，应该明白的一点就是，不可重复读对应的是修改，即UPDATE操作。但是可能还会有幻读问题。因为幻读问题对应的是插入INSERT操作，而不是UPDATE操作

					- 这里需要注意的是：与 SQL 标准不同的地方在于InnoDB 存储引擎在 **REPEATABLE-READ（可重读）**事务隔离级别下使用的是Next-Key Lock 算法，因此可以避免幻读的产生，这与其他数据库系统(如 SQL Server)是不同的。所以说InnoDB 存储引擎的默认支持的隔离级别是 REPEATABLE-READ（可重读）已经可以完全保证事务的隔离性要求，即达到了 SQL标准的 **SERIALIZABLE(可串行化)**隔离级别，而且保留了比较好的并发性能

				- mysql的默认隔离级别

			- SERIALIZABLE(可串行化)： 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。

				- 能解决由INSERT DEL导致的幻读问题

		- MVCC

			- 解决的问题

				- mvcc解决的就是读写时的线程安全问题，线程不用去争抢读写锁

			- 读

				- 快照读

					- mvcc所提到的读是快照读，也就是普通的select语句。快照读在读写时不用加锁，不过可能会读到历史数据

						- ​InnoDB mvcc的实现，基于undolog、版本链、readview

							-  

								- 比如一行数据记录，主键ID是10，name='Jack'，age=10, 被update更新set为name= 'Tom'，age=23。
事务会先使用“排他锁”锁定该行，将该行当前的值复制到undo log中，然后再真正地修改当前行的值，最后填写事务的DB_TRX_ID，使用回滚指针DB_ROLL_PTR指向undo log中修改前的行DB_ROW_ID。

									- 当事务正常提交时只需要更改事务状态为COMMIT即可，不需做其他额外的工作，而Rollback则稍微复杂点，需要根据当前回滚指针从undo log中找出事务修改前的版本并恢复。如果事务影响的行非常多，回滚则可能会变的效率不高，根据经验值没事务行数在1000～10000之间，Innodb效率还是非常高的。很显然，Innodb是一个COMMIT效率比Rollback高的存储引擎。

							-  

								- （1）如果要读取的事务id等于进行读操作的事务id，说明是我读取我自己创建的记录，那么为什么不可以呢。
（2）如果要读取的事务id小于最小的活跃事务id，说明要读取的事务已经提交，那么可以读取。
（3）max_trx_id表示生成readview时，分配给下一个事务的id，如果要读取的事务id大于max_trx_id，说明该id已经不在该readview版本链中了，故无法访问。
（4）m_ids中存储的是活跃事务的id，如果要读取的事务id不在活跃列表，那么就可以读取，反之不行。

									- 示例

										-  

											- 事务A执行查询过程如下:

在执行Select语句时会先生成一个ReadView，m_ids列表的内容就是[200]
从版本链中查找可见的记录，最新版本trx_id值为200，在m_ids列表内，不符合要求，继续跳到下一个版本
下个版本的trxid值为50，小于m_ids列表中最小的事务id 200，所以这个版本是符合要求的，返回余额1000

								-  

							- 一个理论的MVCC模型

								- 假如 test 表有两个字段 name 和 age；MVCC 的三个隐藏列字段名为 transaction_id、 create_version 和 delete_version。

									- insert

										-  

									- update

										-  

									- delete

										-  

								- REPEATABLE READ（可重读）隔离级别下MVCC如何工作：

SELECT
InnoDB会根据以下两个条件检查每行记录：
InnoDB只查找版本早于当前事务版本的数据行，这样可以确保事务读取的行，要么是在开始事务之前已经存在要么是事务自身插入或者修改过的
行的删除版本号要么未定义，要么大于当前事务版本号，这样可以确保事务读取到的行在事务开始之前未被删除
只有符合上述两个条件的才会被查询出来

INSERT：InnoDB为新插入的每一行保存当前系统版本号作为行版本号
DELETE：InnoDB为删除的每一行保存当前系统版本号作为行删除标识
UPDATE：InnoDB为插入的一行新纪录保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为删除标识

						- Read Committed隔离级别：每次select都生成一个快照读
Read Repeatable隔离级别：开启事务后第一个select语句才是快照读的地方，而不是一开启事务就快照读

				- 当前读

					- 还有一种读取数据的方式是当前读，是一种悲观锁的操作。它会对当前读取的数据进行加锁，所以读到的数据都是最新的

						- 主要包括以下几种操作：
select lock in share mode（共享锁）
select for update（排他锁）
update（排他锁）
insert（排他锁）
delete（排他锁）

		- 事务日志

			- 使用事务日志，存储引擎在修改表的数据时只需要修改其内存拷贝，再把该修改行为记录到持久在硬盘上的事务日志中，而不用每次都将修改的数据本身持久到磁盘。

			- 事务日志采用的是追加的方式，因此写日志的操作是磁盘上一小块区域内的顺序I/O，而不像随机I/O需要在磁盘的多个地方移动磁头，所以采用事务日志的方式相对来说要快得多。

			- 事务日志持久以后，内存中被修改的数据在后台可以慢慢刷回到磁盘。

			- 如果数据的修改已经记录到事务日志并持久化，但数据本身没有写回到磁盘，此时系统崩溃，存储引擎在重启时能够自动恢复这一部分修改的数据。

		- 事务是如何通过日志来实现的，说得越深入越好

			- redo log（重做日志） 实现持久化和原子性
在innoDB的存储引擎中，事务日志通过重做(redo)日志和innoDB存储引擎的日志缓冲(InnoDB Log Buffer)实现。事务开启时，事务中的操作，都会先写入存储引擎的日志缓冲中，在事务提交之前，这些缓冲的日志都需要提前刷新到磁盘上持久化，这就是DBA们口中常说的“日志先行”(Write-Ahead Logging)。当事务提交之后，在Buffer Pool中映射的数据文件才会慢慢刷新到磁盘。此时如果数据库崩溃或者宕机，那么当系统重启进行恢复时，就可以根据redo log中记录的日志，把数据库恢复到崩溃前的一个状态。未完成的事务，可以继续提交，也可以选择回滚，这基于恢复的策略而定。
在系统启动的时候，就已经为redo log分配了一块连续的存储空间，以顺序追加的方式记录Redo Log，通过顺序IO来改善性能。所有的事务共享redo log的存储空间，它们的Redo Log按语句的执行顺序，依次交替的记录在一起。

			- undo log（回滚日志） 实现一致性
undo log 主要为事务的回滚服务。在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log记录了数据在每个操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。单个事务的回滚，只会回滚当前事务做的操作，并不会影响到其他的事务做的操作。
Undo记录的是已部分完成并且写入硬盘的未完成的事务，默认情况下回滚日志是记录下表空间中的（共享表空间或者独享表空间）

		- 又引出个问题：你知道MySQL 有多少种日志吗？

			- 错误日志：记录出错信息，也记录一些警告信息或者正确的信息。

			- 查询日志：记录所有对数据库请求的信息，不论这些请求是否得到了正确的执行。

			- 慢查询日志：设置一个阈值，将运行时间超过该值的所有SQL语句都记录到慢查询的日志文件中。

			- 二进制日志：记录对数据库执行更改的所有操作。

				- redo日志和undo日志的区别

					- redo

			- 中继日志：中继日志也是二进制日志，用来给slave 库恢复

			- 事务日志：重做日志redo和回滚日志undo

		- MySQL对分布式事务的支持

			- MySQL 从 5.0.3 InnoDB 存储引擎开始支持XA协议的分布式事务

				-  

					- 模型中分三块

						- 应用程序：定义了事务的边界，指定需要做哪些事务；

						- 资源管理器：提供了访问事务的方法，通常一个数据库就是一个资源管理器；

						- 事务管理器：协调参与了全局事务中的各个事务。

					- 分布式事务采用两段式提交（two-phase commit）的方式

						- 第一阶段所有的事务节点开始准备，告诉事务管理器ready。

						- 第二阶段事务管理器告诉每个节点是commit还是rollback。如果有一个节点失败，就需要全局的节点全部rollback，以此保障事务的原子性。

	- MySQL的锁机制

		- 分类

			- 内部锁

				- 从对数据操作的类型分类

					- 读锁（共享锁）：针对同一份数据，多个读操作可以同时进行，不会互相影响

					- 写锁（排他锁）：当前写操作没有完成前，它会阻断其他写锁和读锁

				- 从对数据操作的粒度分类

					- 表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低（MyISAM 和 MEMORY 存储引擎采用的是表级锁）；

					- 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高（InnoDB 存储引擎既支持行级锁也支持表级锁，但默认情况下是采用行级锁）；

					- 页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。

			- 外部锁

				- 仅用于MyISAM表，当表可能由多个mysql服务器或者myisamchk在线访问时，用以协同访问；

				- 当服务器激活外部锁时，进程访问表之前需要申请文件系统锁，如果无法获取则等待；

				- 可任意时刻使用myisamchk进行读操作，当进行诸如修复或优化表等写操作时，还是要确保mysqld此刻没有用到这个表，否则同时访问表可能会造成其损坏；

				- 可使用repair/optimize/check table替代myisamchk；

				- 参数skip_external_locking用于控制是否激活外部锁，默认不激活

			- 元数据锁

				- 执行事务时基表会被添加metadata lock，以阻止其他会话对该表进行DDL；

				- 5.5.3之前，一旦语句执行完就会释放该锁，但若此时其他事务对基表执行了DDL，则二进制日志的顺序可能会乱掉，故5.5.3起只在事务结束后才释放metadata lock；

				- 倘若获取metadata lock的语句解析成功但执行失败，依旧不会提前释放此锁，只因该失败的sql会被写入二进制日志，需要锁保护日志一致性；

				- 元数据锁在表锁之前获取，如果有会话采用lock table write锁住了表(获取了排他元数据锁)，则后续试图访问该表的其他会话被阻塞于元数据锁等待，而table_locks_waited不会增加；

		- InnoDB 行锁

			- 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。

			- 排他锁（X）：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

			- 意向锁

				- 为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB 还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁

					- 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。

					- 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁。

				- 意向锁的原理

					- 1、一个事务对表中的某一行加了排它锁并且未提交，其他事务想要获得该表的表锁就需要保证：

•当前没有其他事务持有该表的排他锁

•当前没有其他事务持有该表的其中任一行的排它锁

为了检测是否满足第二个条件，就需要在确保users表不存在排它锁的前提下去检测每一行是否存在排它锁，这样的话效率就会很差。因此就出现了意向锁来提高检测互斥的效率。

					- 2、第一个事务获取某一行的排他锁之后其实表中存在两把锁，一把是加在该行上的锁，一把是加在表上的意向排它锁，当其他事务想要获得该表的排他锁时发现已经有事务获得了该表的意向排它锁，那么此次申请该表的排它锁就会失败，而不用一行一行的检测是不是存在某一行被加了锁。

					- 3、意向锁只会和表级的共享锁/排它锁互斥，不会和行级的共享锁/排它锁互斥。具体来讲：共享意向锁和表级共享锁兼容，和标记排它锁互斥；意向排它锁和表级共享锁互斥，和表级排他锁也互斥。另外，意向锁相互之间是兼容的，即一个事务对表中的一行加锁（同时获得了该表的意向锁），另一个事务可以对该表的另一行加锁，此时同样得到了该表的意向锁。

					- 4、意向锁是由数据库引擎自己维护的，用户无法手动操作意向锁。

			- 锁模式(InnoDB有三种行锁的算法)

				- 记录锁(Record Locks)： 单个行记录上的锁。对索引项加锁，锁定符合条件的行。其他事务不能修改和删除加锁项；
SELECT * FROM table WHERE id = 1 FOR UPDATE;
它会在 id=1 的记录上加上记录锁，以阻止其他事务插入，更新，删除 id=1 这一行
在通过 主键索引 与 唯一索引 对数据行进行 UPDATE 操作时，也会对该行数据加记录锁：
-- id 列为主键列或唯一索引列 UPDATE SET age = 50 WHERE id = 1;

				- 间隙锁（Gap Locks）： 当我们使用范围条件而不是相等条件检索数据，并请求共享或排他锁时，InnoDB会给符合条件的已有数据记录的索引项加锁。对于键值在条件范围内但并不存在的记录，叫做“间隙”。
InnoDB 也会对这个“间隙”加锁，这种锁机制就是所谓的间隙锁。
对索引项之间的“间隙”加锁，锁定记录的范围（对第一条记录前的间隙或最后一条将记录后的间隙加锁），不包含索引项本身。其他事务不能在锁范围内插入数据，这样就防止了别的事务新增幻影行。
间隙锁基于非唯一索引，它锁定一段范围内的索引记录。间隙锁基于下面将会提到的Next-Key Locking 算法，请务必牢记：使用间隙锁锁住的是一个区间，而不仅仅是这个区间中的每一条数据。
SELECT * FROM table WHERE id BETWEN 1 AND 10 FOR UPDATE;
即所有在（1，10）区间内的记录行都会被锁住，所有id 为 2、3、4、5、6、7、8、9 的数据行的插入会被阻塞，但是 1 和 10 两条记录行并不会被锁住。
GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况

				- 临键锁(Next-key Locks)： 临键锁，是记录锁与间隙锁的组合，它的封锁范围，既包含索引记录，又包含索引区间。(临键锁的主要目的，也是为了避免幻读(Phantom Read)。如果把事务的隔离级别降级为RC，临键锁则也会失效。)
Next-Key 可以理解为一种特殊的间隙锁，也可以理解为一种特殊的算法。通过临建锁可以解决幻读的问题。 每个数据行上的非唯一索引列上都会存在一把临键锁，当某个事务持有该数据行的临键锁时，会锁住一段左开右闭区间的数据。需要强调的一点是，InnoDB 中行级锁是基于索引实现的，临键锁只与非唯一索引列有关，在唯一索引列（包括主键列）上不存在临键锁。
对于行的查询，都是采用该方法，主要目的是解决幻读的问题。

			- InnoDB行锁原理

				- 1.innodb的行锁是通过给索引的索引项加锁来实现的，准确的说是innodb在走索引的过程中，给查到索引项加锁，如果一个查询没有走索引的过程，就不会加行锁，这个查询包括CRUD的查询

				- 2.innodb按照辅助索引进行数据操作时,辅助索引和主键索引都将锁定指定的索引项

				- 3.通过索引进行数据检索时,innodb才使用行级锁,否则innodb将使用表锁

		- 锁的两种机制

			- 乐观锁会“乐观地”假定大概率不会发生并发更新冲突，访问、处理数据过程中不加锁，只在更新数据时再根据版本号或时间戳判断是否有冲突，有则处理，无则提交事务。用数据版本（Version）记录机制实现，这是乐观锁最常用的一种实现方式

			- 悲观锁会“悲观地”假定大概率会发生并发更新冲突，访问、处理数据前就加排他锁，在整个数据处理过程中锁定数据，事务提交或回滚后才释放锁。另外与乐观锁相对应的，悲观锁是由数据库自己实现了的，要用的时候，我们直接调用数据库的相关语句就可以了。

		- 死锁问题

			- MySQL 遇到过死锁问题吗，你是如何解决的？

				- 死锁产生的条件

					- 两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源

					- 当事务试图以不同的顺序锁定资源时

					- 锁的行为和顺序和存储引擎相关。以同样的顺序执行语句，有些存储引擎会产生死锁有些不会——死锁有双重原因：真正的数据冲突；存储引擎的实现方式。

				- 检测死锁

					- 数据库系统实现了各种死锁检测和死锁超时的机制。InnoDB存储引擎能检测到死锁的循环依赖并立即返回一个错误

				- 死锁恢复

					- 死锁发生以后，只有部分或完全回滚其中一个事务，才能打破死锁，InnoDB目前处理死锁的方法是，将持有最少行级排他锁的事务进行回滚。所以事务型应用程序在设计时必须考虑如何处理死锁，多数情况下只需要重新执行因死锁回滚的事务即可。

				- 外部锁的死锁检测

					- 发生死锁后，InnoDB 一般都能自动检测到，并使一个事务释放锁并回退，另一个事务获得锁，继续完成事务。但在涉及外部锁，或涉及表锁的情况下，InnoDB 并不能完全自动检测到死锁， 这需要通过设置锁等待超时参数 innodb_lock_wait_timeout 来解决

				- 死锁影响性能

					- 死锁会影响性能而不是会产生严重错误，因为InnoDB会自动检测死锁状况并回滚其中一个受影响的事务。在高并发系统上，当许多线程等待同一个锁时，死锁检测可能导致速度变慢。 有时当发生死锁时，禁用死锁检测（使用innodb_deadlock_detect配置选项）可能会更有效，这时可以依赖innodb_lock_wait_timeout设置进行事务回滚。

				- MyISAM避免死锁

					- 在自动加锁的情况下，MyISAM 总是一次获得 SQL 语句所需要的全部锁，所以 MyISAM 表不会出现死锁

				- InnoDB避免死锁

					- 为了在单个InnoDB表上执行多个并发写入操作时避免死锁，可以在事务开始时通过为预期要修改的每个元祖（行）使用SELECT ... FOR UPDATE语句来获取必要的锁，即使这些行的更改语句是在之后才执行的。

					- 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁、更新时再申请排他锁，因为这时候当用户再申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁

					- 如果事务需要修改或锁定多个表，则应在每个事务中以相同的顺序使用加锁语句。 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会

					- 通过SELECT ... LOCK IN SHARE MODE获取行的读锁后，如果当前事务再需要对该记录进行更新操作，则很有可能造成死锁。

					- 改变事务隔离级别

					- 如果出现死锁，可以用 show engine innodb status;命令来确定最后一个死锁产生的原因。返回结果中包括死锁相关事务的详细信息，如引发死锁的 SQL 语句，事务已经获得的锁，正在等待什么锁，以及被回滚的事务等。据此可以分析死锁产生的原因和改进措施。

	- MySQL调优

		- MySQL常见瓶颈

			- CPU：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据时候

			- IO：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候

			- 服务器硬件的性能瓶颈：top，free，iostat 和 vmstat来查看系统的性能状态

				- vmstat 命令会报告有关内核线程、虚拟内存、磁盘、管理程序页面、陷阱和处理器活动的统计信息

		- 性能下降SQL慢 执行时间长 等待时间长 原因分析

			- 查询语句写的烂

			- 索引失效（单值、复合）

			- 关联查询太多join（设计缺陷或不得已的需求）

			- 服务器调优及各个参数设置（缓冲、线程数等）

		- MySQL常见性能分析手段

			- 慢查询日志

				- 运行时间超过 long_query_time 值的 SQL，则会被记录到慢查询日志中

					- long_query_time 的默认值为10，意思是运行10秒以上的语句

					- 默认情况下，MySQL数据库没有开启慢查询日志，需要手动设置参数开启

						- 临时配置

							- mysql> set global slow_query_log='ON';
mysql> set global slow_query_log_file='/var/lib/mysql/hostname-slow.log';
mysql> set global long_query_time=2;

						- 永久配置

							- 修改配置文件my.cnf或my.ini，在[mysqld]一行下面加入两个配置参数
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/lib/mysql/hostname-slow.log
long_query_time = 3

					- 查看慢sql日志

						- 通过 mysqldumpslow --help 查看操作帮助信息

得到返回记录集最多的10个SQL
mysqldumpslow -s r -t 10 /var/lib/mysql/hostname-slow.log
得到访问次数最多的10个SQL
mysqldumpslow -s c -t 10 /var/lib/mysql/hostname-slow.log
得到按照时间排序的前10条里面含有左连接的查询语句
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/hostname-slow.log
也可以和管道配合使用
mysqldumpslow -s r -t 10 /var/lib/mysql/hostname-slow.log | more

			- EXPLAIN 分析查询

				-  

					- id（select 查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序）
id相同，执行顺序从上往下
id全不同，如果是子查询，id的序号会递增，id值越大优先级越高，越先被执行
id部分相同，执行顺序是先按照数字大的先执行，然后数字相同的按照从上往下的顺序执行

					- select_type（查询类型，用于区别普通查询、联合查询、子查询等复杂查询）
SIMPLE ：简单的select查询，查询中不包含子查询或UNION
PRIMARY：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
SUBQUERY：在select或where列表中包含了子查询
DERIVED：在from列表中包含的子查询被标记为DERIVED，MySQL会递归执行这些子查询，把结果放在临时表里
UNION：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
UNION RESULT：从UNION表获取结果的select

					- select_type（查询类型，用于区别普通查询、联合查询、子查询等复杂查询）
SIMPLE ：简单的select查询，查询中不包含子查询或UNION
PRIMARY：查询中若包含任何复杂的子部分，最外层查询被标记为PRIMARY
SUBQUERY：在select或where列表中包含了子查询
DERIVED：在from列表中包含的子查询被标记为DERIVED，MySQL会递归执行这些子查询，把结果放在临时表里
UNION：若第二个select出现在UNION之后，则被标记为UNION，若UNION包含在from子句的子查询中，外层select将被标记为DERIVED
UNION RESULT：从UNION表获取结果的select

					- table（显示这一行的数据是关于哪张表的）

					- type（显示查询使用了那种类型，从最好到最差依次排列 system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL ）tip: 一般来说，得保证查询至少达到range级别，最好到达ref

						- system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现

						- const：表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const用于用常数值比较PRIMARY KEY或UNIQUE索引的所有部分时。

						- eq_ref：对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型。它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY。eq_ref可以用于使用= 操作符比较的带索引的列。比较值可以为常量或一个使用在该表前面所读取的表的列的表达式。

							- 简单地说是const是直接按主键或唯一键读取，eq_ref用于联表查询的情况，按联表的主键或唯一键联合查询

						- ref：对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY(换句话说，如果联接不能基于关键字选择单个行的话)，则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。ref可以用于使用=或<=>操作符的带索引的列。

						- ref_or_null：该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。

						- unique_subquery：该类型替换了下面形式的IN子查询的ref：value IN (SELECT primary_key FROMsingle_table WHERE some_expr);unique_subquery是一个索引查找函数，可以完全替换子查询，效率更高。

						- index_subquery：该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：value IN (SELECT key_column FROM single_table WHERE some_expr)

						- range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range

						- index：该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小。

						- all：对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在它情况下很差。通常可以增加更多的索引而不要使用ALL，使得行能基于前面的表中的常数值或列值被检索出。

					- possible_keys（显示可能应用在这张表中的索引，一个或多个，查询涉及到的字段若存在索引，则该索引将被列出，但不一定被查询实际使用）

					- key：实际使用的索引，如果为NULL，则没有使用索引

					- key_len：key_len列显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。注意通过key_len值我们可以确定MySQL将实际使用一个多部关键字的几个部分
key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的

					- ref：显示索引的哪一列被使用了，如果可能的话，是一个常数。哪些列或常量被用于查找索引列上的值

						-  

					- rows：根据表统计信息及索引选用情况，大致估算找到所需的记录所需要读取的行数

					- Extra：包含不适合在其他列中显示但十分重要的额外信息

						- using filesort: 表示在索引之外，需要额外进行外部的排序动作。导致该问题的原因一般和order by有者直接关系，一般可以通过合适的索引来减少或者避免。

						- Using temporary：使用了临时表保存中间结果，mysql在对查询结果排序时内存配额不够，需要使用临时表。常见于排序order by和分组查询group by。

						- using index：表示相应的select操作中使用了覆盖索引，避免访问了表的数据行，效率不错，如果同时出现using where，表明索引被用来执行索引键值的查找；否则索引被用来读取数据而非执行查找操作

						- using where：使用了where过滤

						- using join buffer：使用了连接缓存

						- impossible where：where子句的值总是false，不能用来获取任何数据

						- select tables optimized away：在没有group by子句的情况下，基于索引优化操作或对于MyISAM存储引擎优化COUNT(*)操作，不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化

						- distinct：优化distinct操作，在找到第一匹配的元祖后即停止找同样值的动作

			- profiling分析

				- 分析步骤

					- 是否支持，看看当前的mysql版本是否支持
mysql>Show variables like 'profiling'; --默认是关闭，使用前需要开启

					- 开启功能，默认是关闭，使用前需要开启
mysql>set profiling=1;

					- 运行SQL

					- 查看结果
mysql> show profiles; +----------+------------+---------------------------------+
 | Query_ID | Duration | Query | 
+----------+------------+---------------------------------+ 
| 1 | 0.00385450 | show variables like "profiling" | 
| 2 | 0.00170050 | show variables like "profiling" | 
| 3 | 0.00038025 | select * from t_base_user | 
+----------+------------+---------------------------------+

						- 诊断SQL，show profile cpu,block io for query id(上一步前面的问题SQL数字号码)

						- 日常开发需要注意的结论
converting HEAP to MyISAM 查询结果太大，内存都不够用了往磁盘上搬了。
create tmp table 创建临时表，这个要注意
Copying to tmp table on disk 把内存临时表复制到磁盘
locked

			- show命令查询系统状态及系统变量

				- Mysql> show status ——显示状态信息（扩展show status like ‘XXX’）

				- Mysql> show variables ——显示系统变量（扩展show variables like ‘XXX’）

				- Mysql> show innodb status ——显示InnoDB存储引擎的状态

				- Mysql> show processlist ——查看当前SQL执行，包括执行状态、是否锁表等

				- Shell> mysqladmin variables -u username -p password——显示系统变量

				- Shell> mysqladmin extended-status -u username -p password——显示状态信息

		- MySQL常见性能优化手段

			- 索引优化

				- 1、全值匹配我最爱
2、最佳左前缀法则，比如建立了一个联合索引(a,b,c)，那么其实我们可利用的索引就有(a), (a,b), (a,b,c)
3、不在索引列上做任何操作（计算、函数、(自动or手动)类型转换），会导致索引失效而转向全表扫描
4、存储引擎不能使用索引中范围条件右边的列
5、尽量使用覆盖索引(只访问索引的查询(索引列和查询列一致))，减少select
6、is null ,is not null 也无法使用索引
7、like "xxxx%" 是可以用到索引的，like "%xxxx" 则不行(like "%xxx%" 同理)。like以通配符开头('%abc...')索引失效会变成全表扫描的操作，
8、字符串不加单引号索引失效
9、少用or，用它来连接时会索引失效
10、<，<=，=，>，>=，BETWEEN，IN 可用到索引，<>，not in ，!= 则不行，会导致全表扫描

				- 一般性建议

					- 1、对于单键索引，尽量选择针对当前query过滤性更好的索引
2、在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前越好。
3、在选择组合索引的时候，尽量选择可以能够包含当前query中的where字句中更多字段的索引
4、尽可能通过分析统计信息和调整query的写法来达到选择合适索引的目的
5、少用Hint强制索引

			- 查询优化

				- 永远小表驱动大表

					- slect * from A where id in (select id from B)`等价于
#等价于
select id from B
select * from A where A.id=B.id
当 B 表的数据集必须小于 A 表的数据集时，用 in 优于 exists

					- select * from A where exists (select 1 from B where B.id=A.id)
#等价于
select * from A
select * from B where B.id = A.id`
当 A 表的数据集小于B表的数据集时，用 exists优于用 in

				- order by关键字优化

					- order by子句，尽量使用 Index 方式排序，避免使用 FileSort 方式排序

					- MySQL 支持两种方式的排序，FileSort 和 Index，Index效率高，它指 MySQL 扫描索引本身完成排序，FileSort 效率较低；

					- ORDER BY 满足两种情况，会使用Index方式排序；①ORDER BY语句使用索引最左前列 ②使用where子句与ORDER BY子句条件列组合满足索引最左前列

					- 尽可能在索引列上完成排序操作，遵照索引建的最佳最前缀

					- 如果不在索引列上，filesort 有两种算法，mysql就要启动双路排序和单路排序

						- 双路排序：MySQL 4.1之前是使用双路排序,字面意思就是两次扫描磁盘，最终得到数据

						- 单路排序：从磁盘读取查询需要的所有列，按照order by 列在 buffer对它们进行排序，然后扫描排序后的列表进行输出，效率高于双路排序

						- 优化策略

							- 增大sort_buffer_size参数的设置，sort_buffer_size是connection级别的参数，每一个链接开辟一个buffer，所以要考虑高并发情况下，内存的承载能力

							- 增大max_length_for_sort_data参数的设置

				- GROUP BY关键字优化

					- group by实质是先排序后进行分组，遵照索引建的最佳左前缀

					- 当无法使用索引列，增大 max_length_for_sort_data 参数的设置，增大sort_buffer_size参数的设置

					- where高于having，能写在where限定的条件就不要去having限定了

			- 数据类型优化

				- 更小的通常更好：一般情况下，应该尽量使用可以正确存储数据的最小数据类型。

				- 简单就好：简单的数据类型通常需要更少的CPU周期。例如，整数比字符操作代价更低，因为字符集和校对规则（排序规则）使字符比较比整型比较复杂。

				- 尽量避免NULL：通常情况下最好指定列为NOT NULL

	- 分区、分表、分库

		- 当数据量较大时（一般千万条记录级别以上），MySQL的性能就会开始下降，这时我们就需要将数据分散到多组存储文件，保证其单个文件的执行效率

			- 能干嘛

				- 逻辑数据分割

				- 提高单一的写和读应用速度

				- 提高分区范围读查询的速度

				- 分割数据能够有多个不同的物理文件路径

				- 高效的保存历史数据

			- 怎么玩

				- 首先查看当前数据库是否支持分区

					- MySQL5.6以及之前版本：SHOW VARIABLES LIKE '%partition%';

					- MySQL5.6：show plugins;

				- 分区类型及操作

					- RANGE分区：基于属于一个给定连续区间的列值，把多行分配给分区。mysql将会根据指定的拆分策略，,把数据放在不同的表文件上。相当于在文件上,被拆成了小块.但是,对外给客户的感觉还是一张表，透明的。
按照 range 来分，就是每个库一段连续的数据，这个一般是按比如时间范围来的，比如交易表啊，销售表啊等，可以根据年月来存放数据。可能会产生热点问题，大量的流量都打在最新的数据上了。
range 来分，好处在于说，扩容的时候很简单。

					- LIST分区：类似于按RANGE分区，每个分区必须明确定义。它们的主要区别在于，LIST分区中每个分区的定义和选择是基于某列的值从属于一个值列表集中的一个值，而RANGE分区是从属于一个连续区间值的集合。

					- HASH分区：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。hash 分发，好处在于说，可以平均分配每个库的数据量和请求压力；坏处在于说扩容起来比较麻烦，会有一个数据迁移的过程，之前的数据需要重新计算 hash 值重新分配到不同的库或表

					- KEY分区：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

			- 看上去分区表很帅气，为什么大部分互联网还是更多的选择自己分库分表来水平扩展咧

				- 分区表，分区键设计不太灵活，如果不走分区键，很容易出现全表锁

				- 一旦数据并发量上来，如果在分区表实施关联，就是一个灾难

				- 自己分库分表，自己掌控业务场景与访问模式，可控。分区表，研发写了一个sql，都不确定mysql是怎么玩的，不太可控

		- 随着业务的发展，业务越来越复杂，应用的模块越来越多，总的数据量很大，高并发读写操作均超过单个数据库服务器的处理能力怎么办？

			- 这个时候就出现了数据分片，数据分片指按照某个维度将存放在单一数据库中的数据分散地存放至多个数据库或表中。数据分片的有效手段就是对关系型数据库进行分库和分表。

区别于分区的是，分区一般都是放在单机里的，用的比较多的是时间范围分区，方便归档。只不过分库分表需要代码实现，分区则是mysql内部实现。分库分表和分区并不冲突，可以结合使用。

				- MySQL分表

					- 垂直拆分

						- 垂直分表，通常是按照业务功能的使用频次，把主要的、热门的字段放在一起做为主要表。然后把不常用的，按照各自的业务属性进行聚集，拆分到不同的次要表中；主要表和次要表的关系一般都是一对一的。

					- 水平拆分(数据分片)

						- 单表的容量不超过500W，否则建议水平拆分。是把一个表复制成同样表结构的不同表，然后把数据按照一定的规则划分，分别存储到这些表中，从而保证单表的容量不会太大，提升性能；当然这些结构一样的表，可以放在一个或多个数据库中。

							- 使用MD5哈希，做法是对UID进行md5加密，然后取前几位（我们这里取前两位），然后就可以将不同的UID哈希到不同的用户表（user_xx）中了。

							- 还可根据时间放入不同的表，比如：article_201601，article_201602。

							- 按热度拆分，高点击率的词条生成各自的一张表，低热度的词条都放在一张大表里，待低热度的词条达到一定的贴数后，再把低热度的表单独拆分成一张表。

							- 根据ID的值放入对应的表，第一个表user_0000，第二个100万的用户数据放在第二 个表user_0001中，随用户增加，直接添加用户表就行了。

				- MySQL分库

					- 为什么要分库

						- 数据库集群环境后都是多台 slave，基本满足了读取操作; 但是写入或者说大数据、频繁的写入操作对master性能影响就比较大，这个时候，单库并不能解决大规模并发写入的问题，所以就会考虑分库。

					- 分库是什么

						- 一个库里表太多了，导致了海量数据，系统性能下降，把原本存储于一个库的表拆分存储到多个库上， 通常是将表按照功能模块、关系密切程度划分出来，部署到不同库上

		- 分库分表后的问题

			- 分布式事务的问题，数据的完整性和一致性问题

			- 数据操作维度问题

				- 用户、交易、订单各个不同的维度，用户查询维度、产品数据分析维度的不同对比分析角度。 跨库联合查询的问题，可能需要两次查询 跨节点的count、order by、group by以及聚合函数问题，可能需要分别在各个节点上得到结果后在应用程序端进行合并

					- 额外的数据管理负担，如：访问数据表的导航定位 

					- 额外的数据运算压力，如：需要在多个节点执行，然后再合并计算程序编码开发难度提升，没有太好的框架解决，更多依赖业务看如何分，如何合，是个难题

	- 主从复制

		- 复制的基本原理

			- slave 会从 master 读取 binlog 来进行数据同步

			- 步骤

				- master将改变记录到二进制日志（binary log）。这些记录过程叫做二进制日志事件，binary log events；

				- salve 将 master 的 binary log events 拷贝到它的中继日志（relay log）;

					-  

				- slave 重做中继日志中的事件，将改变应用到自己的数据库中。MySQL 复制是异步且是串行化的。

		- 基本原则

			- 每个 slave只有一个 master

			- 每个 salve只能有一个唯一的服务器 ID

			- 每个master可以有多个salve

		- 主从复制存在延迟问题

	- 其他

		- 说一说三个范式

			- 第一范式（1NF）：数据库表中的字段都是单一属性的，不可再分。这个单一属性由基本类型构成，包括整型、实数、字符型、逻辑型、日期型等。

			- 第二范式（2NF）：数据库表中不存在非关键字段对任一候选关键字段的部分函数依赖（部分函数依赖指的是存在组合关键字中的某些字段决定非关键字段的情况），也即所有非关键字段都完全依赖于任意一组候选关键字。

			- 第三范式（3NF）：在第二范式的基础上，数据表中如果不存在非关键字段对任一候选关键字段的传递函数依赖则符合第三范式。所谓传递函数依赖，指的是如 果存在"A → B → C"的决定关系，则C传递函数依赖于A。因此，满足第三范式的数据库表应该不存在如下依赖关系： 关键字段 → 非关键字段 x → 非关键字段y

		- 百万级别或以上的数据如何删除

			- 关于索引：由于索引需要额外的维护成本，因为索引文件是单独存在的文件,所以当我们对数据的增加,修改,删除,都会产生额外的对索引文件的操作,这些操作需要消耗额外的IO,会降低增/改/删的执行效率。所以，在我们删除数据库百万级别数据的时候，查询MySQL官方手册得知删除数据的速度和创建的索引数量是成正比的。

				- 所以我们想要删除百万数据的时候可以先删除索引（此时大概耗时三分多钟）

				- 然后删除其中无用数据（此过程需要不到两分钟）

				- 删除完成后重新创建索引(此时数据较少了)创建索引也非常快，约十分钟左右。

				- 与之前的直接删除绝对是要快速很多，更别说万一删除中断,一切删除会回滚。那更是坑了。

