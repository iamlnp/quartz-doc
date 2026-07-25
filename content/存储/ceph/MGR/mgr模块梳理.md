#  mgr概述

ceph-mgr就是将ceph的部分C/C++实现的接口python化（即以前只能通过调用c/c++接口发送msg获取比如osdmap、monmap等集群状态，现通过mgr可以很方便地拿到)。同时，ceph-mgr支持用户自定义的plugin，用以实现特殊功能。