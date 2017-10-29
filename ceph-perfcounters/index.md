# Ceph PerfCounters #

最近由于工作需要，为ceph mon进程添加了一个统计项输出。同时仔细看了下PerfCounters的内部实现方法。

PerfCounters是ceph的一个公共模块，用来将ceph内部的一些统计数据输出。下面的输出的
`throttle-msgr_dispatch_throttler-mon`就是一个PerfCounters实例，属于**Throttle**类。一个PerfCounters包含多个统计项。

PerfCounters使用**perf dump**命令从进程中获取。

    root@ceph-0:~# ceph daemon osd.0 perf dump
    {
        ......
        "throttle-msgr_dispatch_throttler-mon": {
            "val": 0,
            "max": 104857600,
            "get": 298402,
            "get_sum": 141453611,
            "get_or_fail_fail": 0,
            "get_or_fail_success": 0,
            "take": 0,
            "take_sum": 0,
            "put": 298402,
            "put_sum": 141453611,
            "wait": {
                "avgcount": 0,
                "sum": 0.000000000
            }
        }
        ......
    }

## PerfCounters数据结构 ##

PerfCounters使用一个结构`perf_counter_data_any_d`来保存每个统计项，然后PerfCounters保存一个数组`perf_counter_data_vec_t`，每个数值元素均为一个`perf_counter_data_any_d`结构。



    class PerfCounters
    {
    private:
        /** Represents a PerfCounters data element. */
        struct perf_counter_data_any_d {
            const char *name;
            enum perfcounter_type_d type;
            atomic64_t u64;
            atomic64_t avgcount;
            atomic64_t avgcount2;
        };
        typedef std::vector<perf_counter_data_any_d> perf_counter_data_vec_t;
    
        CephContext *m_cct;
        int m_lower_bound;
        int m_upper_bound;
        std::string m_name;
        const std::string m_lock_name;
    
        /** Protects m_data */
        mutable Mutex m_lock;
    
        perf_counter_data_vec_t m_data;
    
        friend class PerfCountersBuilder;
    public:
        ....
    };

结构`perf_counter_data_any_d`中：

- **name**为该统计项的名称
- **type**为统计项的类型，类型可以是数值，计数值，计时值，数值平均值，计时平均值。
- **u64**为统计项的数值，保存数据量或时间量。
- **avgcount**和**avgcount2**为统计项的次数，仅在平均值上需要使用。这里用了两个变量，采用了类似spinlock的无锁实现机制。



## 统计项分类 ##

每个PerfCounter统计项可以是以下类型：

- **数值** 该类型用来表示一个统计项的当前值，一般使用set()， inc()， dec()来设置。

    该类型使用`u64`这个域来保存数值。

    该类型不能被用户请求reset。

    举例：统计队列长度。

- **计数值** 该类型用来表示一个统计项的累计值，一般使用inc()， dec()来设置。

    该类型使用`u64`这个域来保存计数值。

    该类型允许被用户请求reset，这样用户可以获取一个时间点到另一个时间点的累计数量。

    举例：统计日志累计写入字节数。

- **计时值** 该类型用来表示一个统计项的当前计时值，一般使用tset()来设置。

    该类型使用`u64`这个域来保存计数值。

    该类型在ceph中基本没有被使用。

- **计数平均值** 该类型在**计数**的基础上增加一个**次数**的信息，一般使用inc()来设置。这样用户可以方便统计出，采样间隔内的平均每次计数的增加量。

    该类型使用`u64`这个域来保存计数值，avgcount/avgcount2来保存次数。

    该类型允许被用户请求reset。

    举例：统计处理的消息的总字节数（计数），以及消息的数量（计次）。获得两次采样的数据可以求出每个消息的平均字节数。

- **计时平均值** 该类型在**计时**的基础上增加一个次数的信息，一般使用tinc()来设置。这样用户可以方便的统计出，采样间隔内的平均每次计时的增加量。

    该类型使用`u64`这个域来保存计时值，avgcount/avgcount2来保存次数。

    该类型允许被用户请求reset。

    举例：统计消息的总处理时间（计时），以及消息的数量（计次）。获得两次采样的数据可以求出每个消息的平均处理时间。

## PerfCounters使用方法 ##

下面还是以**Throttle**类的实例来说明如何使用中使用PerfCounters。

首先创建一个PerfCountersBuilder对象，参数需要传入PerfCounters的名称和统计项目起始和结束序号。名称最后会呈现给用户，而序号为每个统计项的代号，统计项必须填满起始到结束的每个序号。

    PerfCountersBuilder b(cct, string("throttle-") + name, l_throttle_first,l_throttle_last);

然后加入所有的统计项。这里大部分加入的统计项是**计数值**类型的，最后一个为**计时平均值**类型。

    b.add_u64_counter(l_throttle_val, "val");
    b.add_u64_counter(l_throttle_max, "max");
    b.add_u64_counter(l_throttle_get, "get");
    b.add_u64_counter(l_throttle_get_sum, "get_sum");
    b.add_u64_counter(l_throttle_get_or_fail_fail, "get_or_fail_fail");
    b.add_u64_counter(l_throttle_get_or_fail_success, "get_or_fail_success");
    b.add_u64_counter(l_throttle_take, "take");
    b.add_u64_counter(l_throttle_take_sum, "take_sum");
    b.add_u64_counter(l_throttle_put, "put");
    b.add_u64_counter(l_throttle_put_sum, "put_sum");
    b.add_time_avg(l_throttle_wait, "wait");

创建PerfCouters对象，并加入系统。 logger为PerfCounters类型，需要保存在类成员上，后续设置统计数据需要使用。

    logger = b.create_perf_counters();
    cct->get_perfcounters_collection()->add(logger);

前面三步一般放在类的构造函数中。

统计数据设置，在数据发生变化时需要调用set, inc, tinc等方法设置统计项的数据。特别说明，tinc默认参数每次调用增加次数为1。

    logger->set(l_throttle_max, max.read());
    logger->tinc(l_throttle_wait, dur);
    logger->inc(l_throttle_take);
    logger->inc(l_throttle_take_sum, c);
    logger->set(l_throttle_val, count.read());

PerfCounter的清出，一般放在类的析构函数中。

    cct->get_perfcounters_collection()->remove(logger);
    delete logger;


## PerfCouners的输出过程 ##

由上述使用方法可知，我们实际上是将PerfCounters加入到了PerfCountersCollection中。在执行perf dump命令时，程序调度到`PerfCountersCollection::dump_formatted()`方法，而该方法将遍历其中的每个PerfCounters，并输出每个统计项。 `PerfCountersCollection::dump_formatted()`方法支持仅输出单个PerfCounters，或其中的单个统计项。

这部分代码比较简单，不再作说明了。
