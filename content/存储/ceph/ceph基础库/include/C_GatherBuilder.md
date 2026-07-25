---
写作年份: 2023
tags: ceph基础库
---
> 代码路径：include/Context.h


# 1 效果
C_GatherBuilder记录C_GatherBuilder.new_sub()生成的所有C_Context。每当一个C_Context complete，C_Gather就不在关注它；当C_GatherBuilder发现所有C_Context都已经complete，就调用C_GatherBuilder中设置的finisher。

# 2 如何使用C_GatherBuilder:

1. Create a C_GatherBuilder on the stack
2. Call gather_bld.new_sub() as many times as you want to create new subs, It is safe to call this 0 times, or 100, or anything in between.
3. If you didn't supply a finisher in the C_GatherBuilder constructor, set one with gather_bld.set_finisher(my_finisher)
4. Call gather_bld.activate()

举例：

```c++
C_SaferCond all_done;
//step 1
C_GatherBuilder gb(g_ceph_context, all_done);
//step 2
j.submit_entry(1, first, 0, gb.new_sub()); // add a C_Context to C_Gather
j.submit_entry(2, first, 0, gb.new_sub()); // add a C_Context to C_Gather
//step 3
gb.activate(); // consume C_Context as soon as they complete()

all_done.wait(); // all_done is complete() after all new_sub() are complete()

// The finisher may be called at any point after step 4, including immediately from the activate() function.
// The finisher will never be called before activate().
```

# 3 源码解析

```c++
typedef C_GatherBase<Context, Context> C_Gather;
typedef C_GatherBuilderBase<Context, C_Gather > C_GatherBuilder;

/*
 * ContextType must be an ancestor class of ContextInstanceType, or the same class.
 * ContextInstanceType must be default-constructable.
*/
template <class ContextType, class ContextInstanceType>
class C_GatherBase {
private:
	CephContext *cct;
	int result = 0; //结果
	ContextType *onfinish;
	
	int sub_created_count = 0;
	int sub_existing_count = 0;
	
	mutable ceph::recursive_mutex lock = ceph::make_recursive_mutex("C_GatherBase::lock"); // disable lockdep
	bool activated = false;

	/* 递减sub_existing_count，当r < 0 && result = 0时更新result = r
	** 当(activated == false) || (sub_existing_count != 0)时直接返回不再做任何处理
	** 否则调用delete_me(),完成onfinish回调，且析构自己
	*/
	void sub_finish(ContextType* sub, int r) { 
		...
	}
	void delete_me(); //onfinish非空时执行回调，且析构自己

	//子类
	class C_GatherSub : public ContextInstanceType {
		C_GatherBase *gather;
	public:
		C_GatherSub(C_GatherBase *g) : gather(g) {}
		void complete(int r) override {
			Context::complete(r);
		}
		void finish(int r) override {
			gather->sub_finish(this, r);
			gather = 0;
		}
		
		~C_GatherSub() override {
			if (gather)
			gather->sub_finish(this, 0);
		}
	}

public:
	... // 多版本的析构函数

	void set_finisher(ContextType *onfinish_); //为字段onfinish赋值
	/*设置activated为true，当sub_existing_count非0时返回，否则调用delete_me(执行onfinish回调并析构自己)
	**onfinish的complete可能在某个sub完成的时候，可能是在activate调用的时候
	**所以onfinish的complete可能在某个sub被调用的线程，也可能是activate执行的线程
	**/
	void activate(); 
	
	/* 只能在activated == false时调用
	** 递增sub_created_count和sub_existing_count
	** new一个C_GatherSub对象并返回()
	** Notice: ContextType必须是ContextInstanceType的父类或者祖先类
	**/
	ContextType *new_sub(); 
	
}

template <class ContextType, class GatherType>
class C_GatherBuilderBase
{
public:
	ContextType *new_sub();//生成一个ContextType的子类
	void activate(); //设置activated标记，并调用c_gather->active();
	void deactivate();//重置activated标记、c_gather值和reset finisher
	void set_finisher(); //设置finisher的值
private:
	CephContext *cct;
	GatherType *c_gather;
	ContextType *finisher;
	bool activated;
}
``` 