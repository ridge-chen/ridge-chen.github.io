# nova-compute启动 #

程序从nova/cmd/compute.py的`main()`开始执行。

在做参数解析之类的初始化之后，`main()`创建一个Service对象，然后对该对象调用`service.serve()`方法。

    server = service.Service.create(binary='nova-compute',
                                    topic=CONF.compute_topic,
                                    db_allowed=False)
    service.serve(server)
    service.wait()

创建Service对象会跳转到nova/service.py。create()是Service类的类函数（@classmethod），它填充了一些默认参数后，创建并返回一个Service类的对象。这里的nova/service.py::Service类其实是nova/openstack/common/service.py::Service的子类。

# ServiceLauncher类 #

serve()创建了一个`_launcher`对象，而wait()会等待在这个`_launcher`对象上。

    def serve(server, workers=None):
        global _launcher
        if _launcher:
            raise RuntimeError(_('serve() can only be called once'))
        _launcher = service.launch(server, workers=workers)

    def wait():
         _launcher.wait()

`_launcher`是ServiceLauncher类的实例。`serve()`方法通过调用调用nova/openstack/common/service.py的`launch()`方法，创建了一个ServiceLauncher对象，

    def launch(service, workers=None):
        if workers:
            launcher = ProcessLauncher()
            launcher.launch_service(service, workers=workers)
        else:
            launcher = ServiceLauncher()
            launcher.launch_service(service)
    return launcher

ServiceLauncher类是Launcher类的子类，launch_service()为基类方法，内容：

        service.backdoor_port = self.backdoor_port
        self.services.add(service)

其中，backdoor_port和services成员在`__init__`中声明。

        self.services = Services()
        self.backdoor_port = eventlet_backdoor.initialize_if_enabled()

Services类包含了一个线程池，**为每个加入的service创建一个线程**。

    def add(self, service):
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)

run_service方法为线程执行体，其内容就是执行对应service的start()方法。这里service就是一开始在`main()`创建的Service对象，属于nova/service.py::Service类。

ServiceLauncher类的意义在于，在基类Launcher类的基础上，实现signal的处理，以及业务代码异常后重启。

# Service类 #

代码位置nova/service.py

现在我们来看Service类。它是nova/openstack/common/service.py::Service的子类，父类实现只包含了一个线程池，没有什么内容。

子类的start()方法主要：

- 创建了conn对象，这个是一个连接消息队列的对象
- 创建了rpc_dispatcher对象，这个是一个处理消息的对象
- 连接conn对象和rpc_dispatcher对象，即create_consumer。等待在3个队列上。

提示：

- manager是一个`nova.compute.manager.ComputeManager`类型的对象，构造参数只有host。
- rpc_backend: `nova.openstack.common.rpc.impl_kombu`
- self.conn: `nova.openstack.common.rpc.amqp.ConnectionContext`
- rpc_dispatcher: `nova.openstack.common.rpc.dispatcher.RpcDispatcher`

我们先跳过其中关于消息队列相关的细节，这里只需要知道，如果有nova-api或nova-scheduler发送消息给nova-compute，rpc_dispatcher的`dispatch()`方法会被调用。

nova-compute注册consumer到三个队列：

- `topic=compute，fanout=False`
- `topic=compute.<host>，fanout=False`
- `topic=compute，fanout=True`

# RpcDispatcher #

RpcDispatcher对象是在Service::start()中创建的。

    rpc_dispatcher = self.manager.create_rpc_dispatcher(self.backdoor_port)

这里manager是nova.compute.manager.py::ComputeManager类。create_rpc_dispatcher在类中定义：

    def create_rpc_dispatcher(self, backdoor_port=None, additional_apis=None):
        additional_apis = additional_apis or []
        additional_apis.append(ComputeV3Proxy(self))
        return super(ComputeManager, self).create_rpc_dispatcher(
                     backdoor_port, additional_apis)

然后调用最上层父类nova/manager.py::Manager的create_rpc_dispatcher()方法来创建。

    def create_rpc_dispatcher(self, backdoor_port=None, additional_apis=None):
        '''Get the rpc dispatcher for this manager.

        If a manager would like to set an rpc API version, or support more than
        one class as the target of rpc messages, override this method.
        '''
        apis = []
        if additional_apis:
            apis.extend(additional_apis)
        base_rpc = baserpc.BaseRPCAPI(self.service_name, backdoor_port)
        apis.extend([self, base_rpc])
        serializer = objects_base.NovaObjectSerializer()
        return rpc_dispatcher.RpcDispatcher(apis, serializer)


RpcDispatcher代码位置：**nova/openstack/common/rpc/dispatcher.py**

我们先看其中的`dispatch()`方法，该方法比较简单（但是有点长，就不贴代码了），挨个检查self.callbacks（这个是个数组）中的元素，比较元素的RPC_API_NAMESPACE，RPC_API_VERSION是否和请求匹配，以及是否包含请求的method同名方法。如果都符合，就执行该方法，`dispatch()`方法结束。

self.callbacks是在RpcDispatcher的构造参数中传入的，是List of proxy objects，也就是上面`create_rpc_dispatcher()`方法的`additional_apis`参数。

现在来看callbacks。

包含3个元素：

- nova.compute.manager.ComputeV3Proxy
- nova.compute.manager.ComputeManager
- nova.baserpc.BaseRPCAPI

ComputeV3Proxy声明`RPC_API_VERSION = '3.0'`。它只是实现了`__getattr__()`,并把supported_methods中声明的方法全部重定向到manager（nova.compute.manager.ComputeManager）。这样，`RpcDispatcher::dispatch()`会把请求调度到ComputeManager上。

nova.compute.manager.ComputeManager声明`RPC_API_VERSION = '2.49'`。


# ComputeManager #

代码位置： nova/compute/manager.py

ComputeManager对象在Service中通过compute_manager配置项指定。

    cfg.StrOpt('compute_manager',
               default='nova.compute.manager.ComputeManager',
               help='full class name for the Manager for compute'),

现在我们来看nova-compute创建虚拟机的过程。

1. 在用户执行nova boot命令后，在nova-compute上，ComputeManager的`run_instance()`方法被调度。（注：参数相当多）
2. 对instance加锁后，以相同参数调用`_run_instance()`方法
3. `_run_instance()`主线调用`_build_instance()`方法
4. `_build_instance()`方法在获取网络，块设备后，主线调用`_spawn()`方法。之后通过qga注入文件。
5. `_spawn()`方法主要调用self.driver.spawn()。调用这个方法前，设置虚拟机状态为SPAWNING，调用后，再次更新状态。self.driver是在构造函数中创建的，类型为nova.virt.libvirt.driver.LibvirtDriver

现在回头看一下driver的创建过程。

    self.virtapi = ComputeVirtAPI(self)
    // 这里compute_driver的值为None
    // load_compute_driver()内部会使用compute_driver配置，内容为libvirt.LibvirtDriver，命名空间固定为nova.virt
    // 在nova/virt/libvirt/__init__.py中，写明LibvirtDriver = nova.virt.libvirt.driver.LibvirtDriver
    self.driver = driver.load_compute_driver(self.virtapi, compute_driver)

ComputeVirtAPI是nova/virt/virtapi.py::VirtAPI接口的实现，作为driver反向使用openstack服务的接口。例如虚拟机状态更新到数据库。

# LibvirtDriver #

代码位置： **nova/virt/libvirt/driver.py**

LibvirtDriver是nova/virt/driver.py::ComputeDriver接口的实现。

ComputeDriver接口在虚拟化层管理虚拟机。内部self.virtapi为上层提供的VirtAPI接口的实现，作为driver反向使用openstack服务的接口。

我们接着前面的思路，看一下nova-compute创建虚拟机的过程。上面说到会执行LibvirtDriver的`spawn()`方法。

1. 通过_create_image()方法获取镜像文件
2. 通过`to_xml()`方法生成虚拟机的xml配置，这里最核心的是通过`get_guest_config()`生成配置，然后再将conf转成xml返回。
3. 通过`_create_domain_and_network()`方法根据xml内容调用libvirt生成虚拟机。应该是使用了python-libvirt。

libvirt的细节先跳过。

nova-compute大致的流程就这些。


