---
写作年份: 2024
tags:
  - ceph
  - 基础库
---
`common/perf_counters.cc`

# 1 枚举值
```c++
enum perfcounter_type_d : uint8_t
{
  PERFCOUNTER_NONE = 0,
  PERFCOUNTER_TIME = 0x1,       // float (measuring seconds)
  PERFCOUNTER_U64 = 0x2,        // integer (note: either TIME or U64 *must* be set)
  PERFCOUNTER_LONGRUNAVG = 0x4, // paired counter + sum (time)
  PERFCOUNTER_COUNTER = 0x8,    // counter (vs gauge)
  PERFCOUNTER_HISTOGRAM = 0x10, // histogram (vector) of values
};

enum unit_t : uint8_t
{
  UNIT_BYTES,
  UNIT_NONE
};
```

# 2 PerfCounters
```c++
class PerfCounters
{
public:
	struct perf_counter_data_any_d {
		const char *name;
	    const char *description;
	    const char *nick;
	    uint8_t prio = 0;
	    enum perfcounter_type_d type;
	    enum unit_t unit;
	    std::atomic<uint64_t> u64 = { 0 };
	    std::atomic<uint64_t> avgcount = { 0 };
	    std::atomic<uint64_t> avgcount2 = { 0 };
	    std::atomic<uint64_t> max_latency = { 0 };
	    std::unique_ptr<PerfHistogram<>> histogram; 
	}
```
PerfHistogram的定义见：[[PerfHistogram]]


# 3 PerfCountersBuilder
```c++
class PerfCountersBuilder
{
public:
	enum {
	    PRIO_CRITICAL = 10,
	    // 'interesting' is the default threshold for `daemonperf` output
	    PRIO_INTERESTING = 8,
	    // `useful` is the default threshold for transmission to ceph-mgr
	    // and inclusion in prometheus/influxdb plugin output
	    PRIO_USEFUL = 5,
	    PRIO_UNINTERESTING = 2,
	    PRIO_DEBUGONLY = 0,
	};
	PerfCountersBuilder(CephContext *cct, const std::string &name,
		    int first, int last);
	~PerfCountersBuilder();

	void add_u64(int key, const char *name, const char *description=NULL, 
				 const char *nick = NULL,int prio=0, int unit=UNIT_NONE);
	void add_u64_counter(int key, const char *name, const char *description=NULL,
						 const char *nick = NULL, int prio=0, int unit=UNIT_NONE);
	void add_u64_avg(int key, const char *name, const char *description=NULL,
				     const char *nick = NULL, int prio=0, int unit=UNIT_NONE);
	void add_time(int key, const char *name, const char *description=NULL,
				  const char *nick = NULL, int prio=0);
	void add_time_avg(int key, const char *name, const char *description=NULL,
				      const char *nick = NULL, int prio=0);
	void add_u64_counter_histogram(int key, const char* name,
								   PerfHistogramCommon::axis_config_d x_axis_config,
							       PerfHistogramCommon::axis_config_d y_axis_config,
								   const char *description=NULL, const char* nick = NULL,
							       int prio=0, int unit=UNIT_NONE);

	void set_prio_default(int prio_); //设置优先级
	PerfCounters* create_perf_counters(); //PerfCounters生成器
```


## 3.1 使用方法
```c++
//在.h中定义: PerfCounters *logger = nullptr;

//定义key枚举
enum {
	l_mdl_first = 5000,
	l_mdl_evadd,
	l_mdl_evex,
	...
	l_mdl_replayed,
	l_mdl_last,
};

//初始化
void MDLog::create_logger() {
	PerfCountersBuilder plb(g_ceph_context, "mds_log", l_mdl_first, l_mdl_last); //构造builder
	...

	plb.add_u64_counter(l_mdl_evadd, "evadd", "Events submitted", "subm",
                      PerfCountersBuilder::PRIO_INTERESTING);
	plb.add_u64(l_mdl_ev, "ev", "Events", "evts",
              PerfCountersBuilder::PRIO_INTERESTING);
    ...
	plb.set_prio_default(PerfCountersBuilder::PRIO_USEFUL);
	plb.add_time_avg(l_mdl_jlat, "jlat", "Journaler flush latency");
	...

	logger = plb.create_perf_counters();
	g_ceph_context->get_perfcounters_collection()->add(logger);
}

//析构
MDLog::~MDLog()
{
	if (logger) {
	    g_ceph_context->get_perfcounters_collection()->remove(logger);
	    delete logger;
	    logger = 0;
	}
}

```