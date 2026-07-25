# 1 file_to_extents

```c++
void Striper::file_to_extents(
    CephContext *cct, const file_layout_t *layout, uint64_t offset,
    uint64_t len, uint64_t trunc_size, uint64_t buffer_offset,
    striper::LightweightObjectExtents* object_extents) {
  ldout(cct, 10) << "file_to_extents " << offset << "~" << len << dendl;
  ceph_assert(len > 0);

  /*
   * we want only one extent per object!  this means that each extent
   * we read may map into different bits of the final read
   * buffer.. hence buffer_extents
   */
    
  /* object_size：对象的大小，就是rados底层对象的大小，一般默认是4M 
   * su：对象分片大小，即条带单元    
   * stripe_count：条带宽度，也就是一个strip跨多少个对象，也就是一个objectset中对象的个数
   * stripes_per_object：一个对象包含的对象分片数 
   */
 
  __u32 object_size = layout->object_size;
  __u32 su = layout->stripe_unit;
  __u32 stripe_count = layout->stripe_count;
  ceph_assert(object_size >= su);
  if (stripe_count == 1) {
    ldout(cct, 20) << " sc is one, reset su to os" << dendl;
    su = object_size;
  }
  uint64_t stripes_per_object = object_size / su;
  ldout(cct, 20) << " su " << su << " sc " << stripe_count << " os "
		 << object_size << " stripes_per_object " << stripes_per_object
		 << dendl;

  uint64_t cur = offset;
  uint64_t left = len;
  while (left > 0) {
    // layout into objects， 位于第几个对象分片，0开始
    uint64_t blockno = cur / su; // which block
    // which horizontal stripe (Y)，位于第几个条带，0开始
    uint64_t stripeno = blockno / stripe_count;
    // which object in the object set (X)，位于当前条带中第几个位置，0开始
    uint64_t stripepos = blockno % stripe_count;
    // which object set，位于第几个对象集
    uint64_t objectsetno = stripeno / stripes_per_object;
    // object id，位于第几个对象
    uint64_t objectno = objectsetno * stripe_count + stripepos;

    // map range into object
    uint64_t block_start = (stripeno % stripes_per_object) * su;
    uint64_t block_off = cur % su;
    uint64_t max = su - block_off;

    uint64_t x_offset = block_start + block_off;
    uint64_t x_len;
    if (left > max)
      x_len = max;
    else
      x_len = left;

    ldout(cct, 20) << " off " << cur << " blockno " << blockno << " stripeno "
		   << stripeno << " stripepos " << stripepos << " objectsetno "
		   << objectsetno << " objectno " << objectno
		   << " block_start " << block_start << " block_off "
		   << block_off << " " << x_offset << "~" << x_len
		   << dendl;

    striper::LightweightObjectExtent* ex = nullptr;
    auto it = std::upper_bound(object_extents->begin(), object_extents->end(),
                               objectno, OrderByObject());
    striper::LightweightObjectExtents::reverse_iterator rev_it(it);
    if (rev_it == object_extents->rend() ||
        rev_it->object_no != objectno ||
        rev_it->offset + rev_it->length != x_offset) {
      // expect up to "stripe-width - 1" vector shifts in the worst-case
      ex = &(*object_extents->emplace(
        it, objectno, x_offset, x_len,
        object_truncate_size(cct, layout, objectno, trunc_size)));
        ldout(cct, 20) << " added new " << *ex << dendl;
    } else {
      ex = &(*rev_it);
      ceph_assert(ex->offset + ex->length == x_offset);

      ldout(cct, 20) << " adding in to " << *ex << dendl;
      ex->length += x_len;
    }

    ex->buffer_extents.emplace_back(cur - offset + buffer_offset, x_len);

    ldout(cct, 15) << "file_to_extents  " << *ex << dendl;
    // ldout(cct, 0) << "map: ino " << ino << " oid " << ex.oid << " osd "
    //		  << ex.osd << " offset " << ex.offset << " len " << ex.len
    //		  << " ... left " << left << dendl;

    left -= x_len;
    cur += x_len;
  }
```

对象切割模型：
![[【01】技术系统/【01】通用技术/存储/ceph/osdc模块/assets/IMG-2025-02-11-14.png]]

