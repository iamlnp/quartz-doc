---
写作年份:
  - "2025"
imageNameKey: gendisk-通用磁盘
tags:
  - Linux
---
Linux内核将磁盘类设备中为磁盘通用的部分信息提取出来，表示为通用磁盘描述符gendisk

磁盘类设备驱动负责分配、初始化通用磁盘描述符，将它添加到系统中

```c
struct gendisk {
	/* major, first_minor and minors are input parameters only,
	 * don't use directly.  Use disk_devt() and disk_max_parts().
	 */
	int major;			/* major number of driver，磁盘的主设备号 */
	int first_minor;    /* 和本磁盘关联的第一个次设备号*/
	int minors;         /* maximum number of minors, =1 for disks that can't be partitioned. */

	char disk_name[DISK_NAME_LEN];	/* name of major driver */
	char *(*devnode)(struct gendisk *gd, umode_t *mode);/*此回调函数可以在为该通用磁盘创建设备节点时，提供名字线索*/

	unsigned int events;		/* supported events */
	unsigned int async_events;	/* async events, subset of all */

	/* Array of pointers to partitions indexed by partno.
	 * Protected with matching bdev lock but stat and other
	 * non-critical accesses use RCU.  Always access through
	 * helpers.
	 */
	struct disk_part_tbl __rcu *part_tbl; /*指向磁盘分区表描述符的指针*/
	struct hd_struct part0; //磁盘的分区0，将整个磁盘也可以作为一个分区，分区编号为0

	const struct block_device_operations *fops;
	struct request_queue *queue;
	void *private_data;

	int flags;
	struct rw_semaphore lookup_sem;
	struct kobject *slave_dir;

	struct timer_rand_state *random;
	atomic_t sync_io;		/* RAID */
	struct disk_events *ev;
#ifdef  CONFIG_BLK_DEV_INTEGRITY
	struct kobject integrity_kobj;
#endif	/* CONFIG_BLK_DEV_INTEGRITY */
	int node_id;
	struct badblocks *bb;
	struct lockdep_map lockdep_map;
};
```