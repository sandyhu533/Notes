---
description: 阅读libevent源码过程中的一些记录
---

# Libevent - Source Code \(5\)

## 优先级设置

Libevent可以对event进行优先级设置，在默认情况下是event_base只支持一个优先级，可以在初始化event_base的时候增加优先级的级别。

`int event_base_priority_init(struct event_base *base, int n_priorities);`

这个函数必须在任何event被激活之前调用。

看一下它的实现：

```c
// event.c
int
event_base_priority_init(struct event_base *base, int npriorities)
{
	int i;
	
  // 检查了不能有被激活的event，npriorities的值必须合法
	if (N_ACTIVE_CALLBACKS(base) || npriorities < 1
	    || npriorities >= EVENT_MAX_PRIORITIES)
		return (-1);
  
  // 如果以前已经设置过这个值，直接返回
	if (npriorities == base->nactivequeues)
		return (0);
  
  // 如果之前设置过其他的值，free掉原本的数据结构
	if (base->nactivequeues) {
		mm_free(base->activequeues);
		base->nactivequeues = 0;
	}
  
  // 重新分配activequeues，并且做初始化
	/* Allocate our priority queues */
	base->activequeues = (struct event_list *)
	  mm_calloc(npriorities, sizeof(struct event_list));
	if (base->activequeues == NULL) {
		event_warn("%s: calloc", __func__);
		return (-1);
	}
	base->nactivequeues = npriorities;

	for (i = 0; i < base->nactivequeues; ++i) {
		TAILQ_INIT(&base->activequeues[i]);
	}

	return (0);
}

// event_list的定义
TAILQ_HEAD (event_list, event);

#define TAILQ_HEAD(name, type)			\
struct name {					\
	struct type *tqh_first;			\
	struct type **tqh_last;			\
}
#endif

// 翻译过来就是
struct event_list {
  struct event *tqh_first;
  struct event **tqh_last;
}

// 所以event_list是一个链表，链表里面node的类型是event
```

这里可以基本得出event_base对于每一个优先级都维护了一个链表，对于任意优先级的event添加的时候往对应的链表里面插入event，在处理event事件时先处理高优先级的链表再处理低优先级的链表。

那么如何设置一个event的优先级呢，主要是通过`event_priority_set`

```c
// event.c
int
event_priority_set(struct event *ev, int pri)
{
	_event_debug_assert_is_setup(ev);
	
  // 检查这个event没有被激活
	if (ev->ev_flags & EVLIST_ACTIVE)
		return (-1);
	if (pri < 0 || pri >= ev->ev_base->nactivequeues)
		return (-1);
	
  // 设置ev_pri标志位为对应的优先级
	ev->ev_pri = pri;

	return (0);
}
```

下面看看event的优先级是如何影响event_base的，核心的地方在于在`event_queue_insert`里将event插入到了对应优先级的队列，并且`event_queue_remove`的时候将其从对应的队列中删除掉了

下面看看event_base的优先级是如何影响event_base的事件处理顺序的，event_base对于优先级从低到高处理，调用`event_process_active_single_queue`

>  Helper for event_process_active to process all the events in a single queue,
>
>  releasing the lock as we go.  This function requires that the lock be held
>
>  when it's invoked.  Returns -1 if we get a signal or an event_break that
>
>  means we should stop processing any active events now.  Otherwise returns
>
>  the number of non-internal events that we processed.

如果有non-internal的事件在`event_process_active`里面就立即返回。

## Event相关的一些函数操作

1. `event_add_internal`

* 首先根据信号的类型将event分流到`evmap_io_add`(EV_READ, EV_WRITE)和`evmap_signal_add`（EV_SIGNAL)	

* 然后插入到timeout的小根堆中

  





## 参考资料

http://www.wangafu.net/~nickm/libevent-book/Ref2_eventbase.html



