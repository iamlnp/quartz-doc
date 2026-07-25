---
写作年份: 2024
tags:
  - ceph
  - monitor
  - 技术学习
---

# Paxos

Paxos::commit_finish

Paxos::handle_begin

Paxos::finish_round

Paxos::handle_lease


-> propose_pending - STATE_UPDATING

-> begin

-> commit_start - STATE_WRITING_PREVIOUS || STATE_WRITING

-> commit_finish - STATE_REFRESH
    -> commit_proposal
    -> finish_round - STATE_ACTIVE

-> election_finished 
    -> `_active()`
        -> mon->is_leader
            -> create_pending()
        -> on_active()

# 状态机

STATE_UPDATING

# 选举
create_initial()

ClockMonitor::prepare_command() -> 使能clock，创建初始map，

1. 使用ceph_clock_now()设置clock_time和os_time
    
2. 设置init_clock_time、init_monotime、init_clock_time_offset
    
3. epch为1，开始选举
    

ClockMonitor::on_active() -> 心跳处理

1. 第一次启动后修正clock
    

ClockMonitor::on_restart()-> 第一次重启

1. 初始化init_monotime: init_monotime = ceph::coarse_mono_clock::zero()
    
2. 初始化init_clock_time_offset：init_clock_time_offset = utime_t(0,0)
    

ClockMonitor::update_from_paxos() -> 更新paxos，涉及反序列化map

Monitor::init_paxos() -> svc->init() Monitor::tick() -> svc->tick();

Monitor::refresh_from_paxos -> svc->refresh -> svc->post_refresh()

`Monitor::_reset()` -> paxos->restart(); -> svc->restart()

# beacon
PaxosService::dispatch
    -> preprocess_query
    -> prepare_update
        -> prepare_beacon


# tick
初始化 
Monitor::init()
    -> Monitor::new_tick()

定时器
Monitor::tick()
    -> for  svc :  paxos_service 
        -> svc->tick()
            -> Tfs_MDSMonitor::tick()
        -> svc->maybe_trim()
