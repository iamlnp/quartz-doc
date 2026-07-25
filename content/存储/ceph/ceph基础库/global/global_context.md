---
写作年份: 2024
tags:
  - ceph
  - 基础库
---
`global/global_context.cc`
```c++
namespace TOPNSPC::global {
	CephContext *g_ceph_context = NULL;
ConfigProxy& g_conf() {
#if defined(WITH_SEASTAR) && !defined(WITH_ALIEN)
	return crimson::common::local_conf();
#else
	return g_ceph_context->_conf;
#endif
}
```