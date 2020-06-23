WD同步异步设计

-v0.1 wangzhou 2020/6/17 init
-v0.2 wangzhou 2020/6/22 add part 3

本文基于现有Linaro wd库的设计，做其中的异步和同步功能的设计。目前wd库中并没有
异步的设计，而且同步的实现中，由于存在cpu循环等待，存在cpu占用率搞的的问题。
本文梳理这些关系，并做相应设计。

0. 对齐概念
-----------

 在wd设计文档中，我们把全部wd相关的软件分为了，1. libwd库，2. libhisi_qm库，
 3. libhisi_comp库(可以是其他的算法库)。libwd提供基础通道的API接口，libhisi_qm
 在libwd的基础之上，提供海思qm的操作接口，算法库实现业务驱动，算法库调用
 libhisi_qm提供的队列管理接口。libwd库的API基本稳定，libhisi_qm库是对硬件的封装，
 也基本稳定，本文讨论算法库线程模型的设计。

 我们面对的硬件模型是：整个系统内分享硬件设备，e.g. /dev/hisi-zip[01]。多进程
 通过open硬件设备独占硬件队列，并行的使用硬件设备。设备上各个硬件队列没有做QoS，
 只要发包够快，单队列可以沾满设备全部性能。

 直接使用算法库的是用户。用户可能：1. 同步调用算法库接口，2. 异步调用算法库接口。
 其中，同步调用可能有大量线程同时调用算法库接口, e.g. ceph。因为硬件队列的个数
 有限，不同线程上的请求可能被发送到相同的硬件队列上。

 wd库基于SVA,在SVA的情况下，CPU只需要组包发给硬件，CPU的消耗很小，瓶颈在加速器上。
 由于目前的加速器硬件没有做QoS, 负载会直接加到整个加速器上。曾大队列数没有作用。

1. 同步接口
-----------

 基于这样的模型，对于同步的情况，当硬件处理时间很短的时候，发包后直接poll即可。
 当硬件处理比较长的时候，需要让出CPU，避免消耗过多CPU做poll。这里硬件处理时间长
 可能是 1. 包长比较大，2. 硬件资源被他人占用。因为我们无法准确估计，处理时间长
 的时候，需要让出CPU的时间，我们引入专门的IO处理线程去处理IO，业务线程发包完毕
 后让出CPU，并激活IO线程去处理IO，IO处理完后通知业务线程。

2. 异步接口
-----------

 用户可能会下发异步请求给wd库，用户下发一个请求，以及这个请求完成后需要调用的
 回调函数。算法层接收用户发送的请求后，发送给硬件队列。算法层需要起线程poll完成
 的任务，当poll到完成的任务时调用用户下发的回调函数。异步场景下这个poll线程，可以
 放在算法层，也可以提供一个接口用户。

3. 算法层设计
-------------

 算法层我们引入一个session的概念，他表示用户执行算法请求的一个上下文。用户只需要
 感知session这个接口而不需要知道驱动是怎么支持他的。这就需要在算法层维护一个
 session到ctx(硬件队列句柄)的关系。当用户申请多session的时候，这个关系是多session
 对应多ctx。整体上看，一个加速器设备(dev)的所有硬件队列在多进程之间共享。我们需要
 决定一个进程分配的ctx数量(ctx_num)。多session在多ctx上的关联也可以先使用RR的
 方式。session和ctx的关联可以在session创建的时候就确定。一般，session的个数大于
 ctx的个数，会出现多个session被关联到一个ctx上的情况。

 基于以上的session，ctx的方式。同步接口的内部实现，可以是session请求在发包和收包
 的整个过程中加锁。可以看出，多session的场景，性能会受到这个锁的限制。但是，我们
 可以先搞成这样。可以看到为了性能，我们需要叫多个并发的请求可以进到硬件。硬件处理
 完成后，软件继续处理。这里，软件可以同步的轮询硬件，获得硬件是否处理完成的标记。
 但是，如果硬件处理包的时间比较长，这时的CPU开销就比较大, 因为多个线程会在CQ上
 排队，后续线程的CPU开销将和队列深度有关系。
```
       T1    T2    T3     T4

     +-----+-----+-----+-----+
     | cqe | cqe | cqe | cqe |  	
     +-----+-----+-----+-----+

       head              tail
```
 对于大包的情况，不仅需要并发的把请求送到硬件，还需要发送线程可以让出CPU，当任务
 处理完后再由软件通知发送线程继续。为此，大包(硬件耗时长的时候)的情况下，发送线程
 可以在发完包后就wait，算法层里起线程轮询任务完成情况，对于完成的任务，轮询线程
 signal等待线程，使其继续运行。
```
       T1    T2    T3     T4
       |     |
       v     v             signal
      wait  wait <--------------------------------  T_polling
                                                      
		     +-----+-----+-----+-----+       ^    |
		     | cqe | cqe | cqe | cqe |       |    |
		     +-----+-----+-----+-----+       +----+
       |     |
       v     v
```           
 这个轮询线程都在算法层的控制之内，负载并不会重。可以间隔时间去轮询硬件队列。
 后续优化有几点考虑，1. 多线程时，可能是多ctx的，T_polling的数目可能要增加;
 2. 由于我们是一个SMP系统，用户线程可能来自不同的NUMA域，将来可能要为不同的NUMA
 域预留不同的ctx，创建session的时候，session和同一个NUMA域的ctx绑定。


 再来看异步接口的设计。异步接口的实现和上面大包同步接口比较像，区别在于: 1. 发送
 线程发送完请求后直接返回，发送请求会带一个回调函数，这个回调函数需要在请求执行
 完后被调用; 2. 需要有线程去执行回调函数。所以，session和ctx的绑定关系，异步的
 场景和上述同步的场景可以是一样的。轮询线程可以放到算法库，也可以暴露给用户，
 因为轮询线程需要执行用户传入的回调函数，所以，如果这个回调函数开销大，性能瓶颈
 将会出现在这里。如果，出现性能瓶颈，需要考虑加入更多的轮询线程。

 我们可以先把轮询接口暴露给用户，因为用户只能感知到session，这个轮询接口可以是
 针对一个进程里的所有异步session做轮询。那么算法库里就需要维护所有异步session的
 数据结构。轮询接口的内部实现是先通过session找见所有异步ctx，然后逐个轮询ctx，
 执行完成任务的回调函数。如果需要增加轮询线程，就需要分配各个线程轮询的ctx，这里
 需要增加算法层的接口，在一开始告诉算法层寻轮线程的个数，算法层根据异步ctx的数目
 预先决定多线程轮询的方式。
```
       T1    T2    T3     T4                         T_polling_1   T_polling_2
       |     |                                            |             |
       v     v                                            |             |
      return return                                       |             |
                                                          |             |
		     +-----+-----+-----+-----+            |             |
		     | cqe | cqe | cqe | cqe |  <---------+             |
		     +-----+-----+-----+-----+                          |
		     ...                                                |
		     +-----+-----+-----+-----+                          |
		     | cqe | cqe | cqe | cqe |  <-----------------------+
		     +-----+-----+-----+-----+       
```           

 轮询线程也可以被封装到算法层内部。这时，我们面对和同步实现一样的问题: 1. 需要
 根据轮询的负载调节轮询线程数目；2. 轮询线程, 发送线程和ctx要在一个NUMA域，避免
 可能的性能问题。
```
       T1    T2    T3     T4                         
       |     |                                                         
       v     v                                                         
      return return                                                    
                                                                       
		     +-----+-----+-----+-----+                         
		     | cqe | cqe | cqe | cqe |  <-- T_polling_1
		     +-----+-----+-----+-----+                          
		     ...                                                
		     +-----+-----+-----+-----+                          
		     | cqe | cqe | cqe | cqe |  <-- T_polling_2
		     +-----+-----+-----+-----+       
```           

 如果用户APP不感知自己的NUMA域，一个用户进程里的线程可能来自不同的NUMA域，我们就
 需要在算法层里提前为用户预留不同NUMA域的资源，比如为每个NUMA域预留ctx。以现在的
 2P系统举例，我们有四个NUMA域，需要4个同步ctx，4个异步ctx，一个进程需要8个ctx。
 1024个ctx都分给一个PF可以支持128个进程。但是，如果1024个ctx先分到64个PF/VF，那
 每个PF/VF只可以支持2个进程。我们也可以动态的分配发送线程所在NUMA域上的硬件资源，
 以及把轮询线程于同NUMA域上的CPU绑定，这个需要在算法层做相关的检测和匹配。

 异步的具体实现，可以在qm驱动里实现一个请求缓存池，建立缓存池和ctx的对应关系。
 每一个进程通过一个全局的结构(ctx pool)记录所有的ctx。轮询线程通过ctx pool轮询
 本进程的ctx，当有硬件任务完成的时候，通过硬件任务找见task pool里缓存的任务，
 执行相应的回调函数。
```
       T1    T2    T3     T4                         T_polling_1   T_polling_2
       |     |
       | ----+----------+                                 |             |
       v     v          |                                 |             |
      return return     |                                 +-----+  +----+
                        |                                       |  |
                        v                                       v  v
                    +---------+                              +--------+
                    |Task pool|                              |Ctx pool|
                    +---------+                              +--------+
                                                                  |
		     +-----+-----+-----+-----+                    |
		     | cqe | cqe | cqe | cqe |                    |
		     +-----+-----+-----+-----+                    |     
		     ...                         <----------------+
		     +-----+-----+-----+-----+                          
		     | cqe | cqe | cqe | cqe |  
		     +-----+-----+-----+-----+       
```           