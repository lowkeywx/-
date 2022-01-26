# tars 客户端部分代码分析

## 主要类和功能

- Communicator

    管理io线程和异步回调线程，相当于大脑， 协调各个部分的合作
- CommunicatorEpoll

    io线程，内涵协程调度器，调度器内包含epoll切驱动epoll
- ServantProxy

    远程调用的入口，不同的入口最终会调用servant_invoke, 函数内部组装message所需的数据。选择本线程的消息队列以及轮训选择CommunicatorEpoll（io线程）。message入队，唤醒io线程的epoller。
- ObjectProxy

    servantProxy在CommunicatorEpoll中的替身。message经过线程消息队列，被epoller唤醒以后会交给ProjectProxy， 对应接口为invoke
- AdaptorProxy

    objectProxy痛殴endpointMgr管理多个adaptorProxy， 每个AdaptorProxy对应一个远程服务。objectProxy承载的invoke会通过adaptorProxy负载到一个远程服务。具体的收发工作由TC_Transceiver承担。

## 初始化流程

- ServantProxy创建

  - 创建ServantProxy依赖于Communicator实例。通过StringToServant生成。
  - 这里有一点需要注意，特例话的ServantProxy，例如helloPrx不会直接创建对象。tars的解决方案是生成基类ServantProxy，并且通过类型转换为HelloPrx类型。由于helloPrx中没有重写的虚接口，也没有自己的成员。所以，这样做是可行的。
  - helloPrx的代码是tars2cpp工具生成的。生成的代码主要功能是将函数参数根据tars协议序列化到buff中。
  - 具体流程体现在接口上为Communicator::stringToProxy(*)->Communicator::getServantProxy(*)->ServantProxyFactory::getServantProxy()->ServantProxy::ServantProxy()
    - ServantProxyFactory是在Communicator::initialize()中创建的。Communicator::initialize()在Communicator::getServantProxy首次调用的使用调用，在**Cummunicator初始化**部分会详细描述。
    - ServantProxy构造出来以后会调用ServantProxy::initialize()。其主要工作就是创建自己的替身并交由CommunicatorEpoll保管。每个CommunicatorEpoll都会创建一个。Communicator.h中也有详细的描述。

- Cummunicator初始化

  - 创建ServantProxyFactory对象，初始化相应成员。主要用于创建ServantProxy。支撑的接口为stringToProxy(*)。
  - 根据配置初始化相应数量的IO线程，即CommunicatorEpoll。**ComunicatorEpoll初始化**中会详细分析。
  - 根据配置初始化相应数量的异步回调线程，即AsyncProcThread。**AsyncProcThread初始化**中会详细分析。
> 以上三个成员是Communicator的核心，可以算做远程调用的极简模型，支撑了tars远程调用的大多数工作。在**驱动模型**中会详细描述。

- CommunicatorEpoll初始化

  - CommunicatorEpoll::CommunicatorEpoll()构造函数中主要给成员初始化默认值。最主要的就是_notify成员，这个成员是远程调用驱动模型中的一个关键，衔接了远程调用发送阶段，消息队列中消息的生产(ServantProxy::invoke)和消费(Epoller->ObjectProxy::invoke)。**驱动模型**中会详细描述。
  - CommunicatorEpoll::startCoroutine()此接口继承自thread。根据接口名字我们可以知道，是启动协程。主要就是三步： 第一步，创建协程调度器TC_CoroutineScheduler。第二步，线程的入口函数以协程方式调度。第三步，协程调度器阻塞执行调度函数。
  - 协程调度器的调度是基于epoll的，tars使用的边缘触发模式。

- AsyncProcThread初始化

  - 构造中就会调用线程的启动函数，启动函数会调用子类重写的run，然后开始消费者模式。
  - 拥有一个msgqueue，并且push操作提供了notify，pop提供了wait操作。
  - 取出mesaage，获取msg中绑定的ballback，执行dispath

- CommunicatorEpollInfo初始化

  - 每个线程都会有一个。
  - 如果是IO线程，初始化是在startCoroutine中调用CommunicatorEpoll的入口函数以后进行的。每个CommunicatorEpoll::run()中都会通过调用addCommunicatorEpoll创建CommunicatorEpollInfo（每个线程都有一个专属的）和CommunicatorEpollReqQueueInfo。CommunicatorEpollReqQueueInfo绑定消息队列、CommunicatorEpoll、线程的关系。CommunicatorEpollInfo绑定Communicator、线程、CommunicatorEpollReqQueueInfo（每个CommunicatorEpoll都会有一个）。
  - 如果是业务线程是在首次调用servant_invoke()->*->selectNetThreadInfo()中创建的。
  - 如果是协程方式调用则会创建私有通信器(SchedCommunicatorEpoll)和SchedCommunicatorEpollInfo。SchedCommunicatorEpollInfo绑定了Communicato、SchedCommunicatorEpoll、msgQueue。

## 驱动模型

- C2S
  - 调用流程：helloPrx::XXX_hello()->ServantProxy::servant_Invoke()->ServantProxy::invoke()->selectNetThreadInfo()->msgqueue::push()->comunicatorEpoll::notify()->epoller::fireEvent()->CommunicatorEpoll::handleNotify()->msgqueue::pop()->ObjectProxy::invoke()->selectAdapterProxy()->AdapterProxy::invoke()->_reqTimeoutQueue::push()->AdapterProxy::invoke_connection_serial()->TC_Transceiver::sendRequest()->TC_Transceiver::send()->socket::send()
  - helloPrx::XXX_hello()

    主要是接口名称和参数序列化，调用栈的打印。最后调用servant_invoke（）
  - ServantProxy::servant_Invoke

    message消息的封装，填充callbake，servantName，funcName，调用类型（同步调用 异步调用 单向调用）等信息。调用invoke
  - ServantProxy::invoke()

    - 通过selectNetThreadInfo选择线程专属的msgqueue和ObjectProxy，
    - 每个线程都会有一个专属的ComunicatorEpollInfo用于绑定，这个名字感觉起的不好，用于混淆，感觉叫做CommunicatorInfo更合适，这个结构用于绑定线程与Communicator、communicatorQueueInfo的关系。
    - conmmunicatorQueueInfo的作用就是把线程专属的消息队列与CommunicatorEpoll绑定起来。详细描述见**CommunicatorEpollInfo初始化**
    - 选择完消息队列和ObjectProxy以后，讲message加入消息队列。
    > 关于线程专属通信器这部分，个人感觉结构上设计的不太清晰。命名也相对混乱。

  - selectNetThreadInfo

    - 主要是选择消息队列和IO线程对应的ObjectProxy
    - 通过Cummonicator选择CommunicatorEpollInfo，再选择线程对应的CommunicatorEpollQueueInfo，从而获得线程专属队列和绑定的CommunicatorEpoll，从而获得ObjectProxy。
    - 在选择之前如果还没有讲队列、线程、communicatorEpoll进行绑定，则会根据当前所处的线程环境创建CommnicatorEpoll和绑定。详细描述见**CommunicatorEpollInfo初始化**部分。

  - msgqueue::push()
    
    这一部分没有么好说的，在servant_invoke中selectNetThreadInfo之后进行。

  - comunicatorEpoll::notify()

    - 这里需要提到一个类FDInfo。这个类是在addCommunicatorEpoll函数中创建出来的。每个线程一个，以线程序号为索引。前文提到过servantProxy::invoke会通过selectNetThreadInfo选择线程专属的消息队列和CommunicatorEpoll，这个线程专属的基础就是线程序号。FDInfo创建以后会把线程对应的消息队列进行绑定，并把自己添加到CommunicatorEpoll的调度器中最终会形成epollEvent添加到Epoll中。
    - 调用nofiy需要提供线程序号，也就选择了对应线程的FDInfo。通过给对应的EpollInfo添加out event激活Epoller。
    - epoller每次loop都会检测激活的事件，如果有激活的时间就会执行fireEvent

  - epoller::fireEvent()

    没什么好说的，讲对应的EpollInfo设置为Out以后会激活相应的时间，详细的可以自己去搜索epoll的源码或教程了解。

  - CommunicatorEpoll::handleNotify()

    - fireEvent的时候触发。获取绑定到FDInfo中的消息队列中的message。
    - 调用message中存储的ObjectProxy的Invoke方法。

  - selectAdapterProxy()

    选择一个AdapterProxy，每个AdapterProxy对应一个远端服务器。AdapterProxy再初始化ObjectProxy的时候会创建，而且会按照一定的策略更新。

  - AdapterProxy::invoke()

    - 将消息加入到超时队列
    - 根据配置决定是顺序处理还是并行处理。顺序处理调用invoke_connection_serial，并行处理调用invoke_connection_parallel

  - invoke_connection_serial()

    - 调用配置的协议函数进行协议数据的封装
    - 获取requestIo
    - 通过TC_Transceiver进行数据的发送

  - TC_Transceiver::sendRequest()

    - 只要sendbuf不为空就循环发送数据，内部会调用socket的send接口讲序列化好的消息发送出去。
- S2C

