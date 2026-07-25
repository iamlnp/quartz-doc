# 时间概念及类说明

类模板 `std::ratio` 及相关的模板提供[编译时有理数算术支持](https://zh.cppreference.com/w/cpp/numeric/ratio)。此模板的每个实例化都准确表示任一确定有理数，只要分子 `Num` 与分母 `Denom` 能表示为 [std::intmax_t](https://zh.cppreference.com/w/cpp/types/integer) 类型的编译时常量

![c++时间ratio定义|500](【01】技术系统/【01】通用技术/存储/ceph/ceph基础库/assets/IMG-2025-02-11-14.png)

## 时长

时长由时间跨度组成，定义为某时间单位的某个计次数。例如，“ 42 秒”可表示为由 42 个 1 秒时间点位的计次所组成的时长

类模板 `std::chrono::duration` 表示时间间隔。

它由 `Rep` 类型的计次数和计次周期组成，其中计次周期是一个编译期有理数常量，表示从一个计次到下一个的秒数。

## 时间点

时间点是从特定时钟的纪元开始经过的时间时长

类模板 `std::chrono::time_point` 表示时间中的一个点。它被实现成如同存储一个 `Duration` 类型的自 `Clock` 的纪元起始开始的时间间隔的值。



`int clock_gettime(clockid_t clk_id, struct timespec* tp)`: Linux下获取精准时间函数