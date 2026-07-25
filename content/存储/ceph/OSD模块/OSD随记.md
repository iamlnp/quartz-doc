---
写作年份:
  - "2025"
imageNameKey: OSD随记
tags:
  - ceph
  - osd
---

# 1 路径
1. /var/lib/ceph/osd/ceph-2
```bash
[20:58:31] root@ceph-221:/var/lib/ceph/osd/ceph-2# ll
总用量 52
-rw-r--r-- 1 ceph ceph 144 11月  5 15:13 activate.monmap
lrwxrwxrwx 1 ceph ceph  93 11月  5 15:13 block -> /dev/ceph-1064db6d-d071-46a3-879c-e77d4f1fc946/osd-block-0ee28839-9704-4ab0-a71f-87e52ec6366c
-rw------- 1 ceph ceph   2 11月  5 15:13 bluefs
-rw------- 1 ceph ceph  37 11月  5 15:13 ceph_fsid
-rw-r--r-- 1 ceph ceph  37 11月  5 15:13 fsid
-rw------- 1 ceph ceph  55 11月  5 15:13 keyring
-rw------- 1 ceph ceph   8 11月  5 15:13 kv_backend
-rw------- 1 ceph ceph  21 11月  5 15:13 magic
-rw------- 1 ceph ceph   4 11月  5 15:13 mkfs_done
-rw------- 1 ceph ceph  41 11月  5 15:13 osd_key
-rw------- 1 ceph ceph   6 11月  5 15:13 ready
-rw------- 1 ceph ceph   3 11月  5 15:13 require_osd_release
-rw------- 1 ceph ceph  10 11月  5 15:13 type
-rw------- 1 ceph ceph   2 11月  5 15:13 whoami
```


# 2 debug命令  

```bash
[root@node1 ~]# ceph daemon osd.0 dump_objectstore_kv_stats
{
    "block_cache_usage": "4156092224",
    "block_cache_pinned_blocks_usage": "0",
    "rocksdb_memtable_usage": "121195464",
    "rocksdb_index_filter_blocks_usage": "0"
}
```