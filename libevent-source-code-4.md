---
description: 阅读libevent源码过程中的一些记录
---

# Libevent - Source Code \(4\)

### epoll backend的实现

相关文件：

```c
epoll.c
changelist-internal.h
```

epoll在`event.c`里的`eventops`里面排在第三位，由于比较常见故分析之以更好地理解`libevent`是

如何调用`backend`的。

先回顾一下`epoll API`

> epoll API 由以下 3 个系统调用组成。
>
> * 系统调用 epoll_create()创建一个 epoll 实例，返回代表该实例的文件描述符。
> * 系统调用 epoll_ctl()操作同 epoll 实例相关联的兴趣列表。通过 epoll_ctl()，我们可以增加新的描述符到列表中，将已有的文件描述符从该列表中移除，以及修改代表文件描述符上事件类型的位掩码。
> * 系统调用 epoll_wait()返回与 epoll 实例相关联的就绪列表中的成员。

`epoll`有两个版本，即是否带有`changelist`

```c
// epoll.c
static const struct eventop epollops_changelist = {
	"epoll (with changelist)",
	epoll_init,
	event_changelist_add,
	event_changelist_del,
	epoll_dispatch,
	epoll_dealloc,
	1, /* need reinit */
	EV_FEATURE_ET|EV_FEATURE_O1,
	EVENT_CHANGELIST_FDINFO_SIZE
};

const struct eventop epollops = {
	"epoll",
	epoll_init,
	epoll_nochangelist_add,
	epoll_nochangelist_del,
	epoll_dispatch,
	epoll_dealloc,
	1, /* need reinit */
	EV_FEATURE_ET|EV_FEATURE_O1,
	0
};
```

先看他们的共同初始化函数`epoll_init`:

```c
static void *
epoll_init(struct event_base *base)
{
	int epfd;
	struct epollop *epollop;

	/* Initialize the kernel queue.  (The size field is ignored since
	 * 2.6.8.) */
  // 初始化epoll，创建一个epfd
  // 参数 size(3200) 指定了我们想要通过 epoll 实例来检查的文件描述符个数。该参数并不是一个上限，而是告诉内核应该如何为内部数据结构划分初始大小。
	if ((epfd = epoll_create(32000)) == -1) {
		if (errno != ENOSYS)
			event_warn("epoll_create");
		return (NULL);
	}

	evutil_make_socket_closeonexec(epfd);
  
  // epollop定义了这个epoll backend的数据结构和具体的信息，存到evbase里
	if (!(epollop = mm_calloc(1, sizeof(struct epollop)))) {
		close(epfd);
		return (NULL);
	}

	epollop->epfd = epfd;

	/* Initialize fields */
  // 初始化分配epoll event的结构
	epollop->events = mm_calloc(INITIAL_NEVENT, sizeof(struct epoll_event));
	if (epollop->events == NULL) {
		mm_free(epollop);
		close(epfd);
		return (NULL);
	}
	epollop->nevents = INITIAL_NEVENT;
  
  // 在init之前base->evsel = eventops[i];将evsel指向了epoll
  // 如果base里面指定了要用changelist或者是环境变量指定了，将这个值改为用epoll (with changelist)
	if ((base->flags & EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST) != 0 ||
	    ((base->flags & EVENT_BASE_FLAG_IGNORE_ENV) == 0 &&
		evutil_getenv("EVENT_EPOLL_USE_CHANGELIST") != NULL))
		base->evsel = &epollops_changelist;

	evsig_init(base);

	return (epollop);
}
```

这里主要进行了`epoll_create`系统调用创建epoll，并在数据区预留了32个epoll_event的空间。然后判断是用带或者不带changelist的版本。

下面看看changelist是什么吧，在`changelist-internal.h`里面有解释说明：

>   A "changelist" is a list of all the fd status changes that should be made
>
>   between calls to the backend's dispatch function.  There are a few reasons
>
>   that a backend would want to queue changes like this rather than processing
>
>   them immediately.
>
> 
>
> ​    1) Sometimes applications will add and delete the same event more than
>
> ​       once between calls to dispatch.  Processing these changes immediately
>
> ​       is needless, and potentially expensive (especially if we're on a system
>
> ​       that makes one syscall per changed event).
>
> 
>
> ​    2) Sometimes we can coalesce multiple changes on the same fd into a single
>
> ​       syscall if we know about them in advance.  For example, epoll can do an
>
> ​       add and a delete at the same time, but only if we have found out about
>
> ​       both of them before we tell epoll.
>
> 
>
> ​    3) Sometimes adding an event that we immediately delete can cause
>
> ​       unintended consequences: in kqueue, this makes pending events get
>
> ​       reported spuriously.

主要是为了在系统调用之前判断那些操作时表示相互可以抵消，如add和del同一个event，或者聚合一些操作，以此减少系统调用的次数，提高效率。

先看`event_changelist_add`：

```c
// evmap.c
int
event_changelist_add(struct event_base *base, evutil_socket_t fd, short old, short events,
    void *p)
{
	struct event_changelist *changelist = &base->changelist;
	struct event_changelist_fdinfo *fdinfo = p;
	struct event_change *change;

	event_changelist_check(base);
	
  // 如果有fdinfo的changelist entry就将他返回，如果没有就构造一个
	change = event_changelist_get_or_construct(changelist, fd, old, fdinfo);
	if (!change)
		return -1;

	/* An add replaces any previous delete, but doesn't result in a no-op,
	 * since the delete might fail (because the fd had been closed since
	 * the last add, for instance. */

	if (events & (EV_READ|EV_SIGNAL)) {
		change->read_change = EV_CHANGE_ADD |
		    (events & (EV_ET|EV_PERSIST|EV_SIGNAL));
	}
	if (events & EV_WRITE) {
		change->write_change = EV_CHANGE_ADD |
		    (events & (EV_ET|EV_PERSIST|EV_SIGNAL));
	}

	event_changelist_check(base);
	return (0);
}

static struct event_change *
event_changelist_get_or_construct(struct event_changelist *changelist,
    evutil_socket_t fd,
    short old_events,
    struct event_changelist_fdinfo *fdinfo)
{
	struct event_change *change;

  // fdinfo在changelist里的id+1，用这个能够避免使这个值为负，更加优雅
	if (fdinfo->idxplus1 == 0) {
    // 如果没有就构造
		int idx;
		EVUTIL_ASSERT(changelist->n_changes <= changelist->changes_size);

		if (changelist->n_changes == changelist->changes_size) {
      // 初始大小为64，每次扩容两倍，用realloc将旧的列表移到新的列表
			if (event_changelist_grow(changelist) < 0)
				return NULL;
		}
		// changelist->changes里面存了对于fd的变更数组，采用动态扩容的方式
		idx = changelist->n_changes++;
		change = &changelist->changes[idx];
		fdinfo->idxplus1 = idx + 1;

		memset(change, 0, sizeof(struct event_change));
		change->fd = fd;
		change->old_events = old_events;
	} else {
    // 如果有就返回
		change = &changelist->changes[fdinfo->idxplus1 - 1];
		EVUTIL_ASSERT(change->fd == fd);
	}
	return change;
}
```

再看`epoll_nochangelist_add`:

```c
static int
epoll_nochangelist_add(struct event_base *base, evutil_socket_t fd,
    short old, short events, void *p)
{
	struct event_change ch;
	ch.fd = fd;
	ch.old_events = old;
	ch.read_change = ch.write_change = 0;
	if (events & EV_WRITE)
		ch.write_change = EV_CHANGE_ADD |
		    (events & EV_ET);
	if (events & EV_READ)
		ch.read_change = EV_CHANGE_ADD |
		    (events & EV_ET);

	return epoll_apply_one_change(base, base->evbase, &ch);
}
```

观察一下对于`event_change`的设定，数据结构包含`fd, old_events, read_change, write_change`，`event_changelist_add`和`event_nochangelist_add`对于`fd和old_events`的设置相同，但`changelist`里面会把event的有效位设置为`EV_ET|EV_PERSIST|EV_SIGNAL`，而`nochangelist`里面只有`EV_ET`。

看一下`event_changelist_del`和`epoll_nochangelist_del`。前者判断了changelist里面有没有之前添加上去的`add`读/写/signal事件，如果有则直接清零，没有再标记`EV_CHANGE_DEL`并做系统调用。后者直接标记并做系统调用。

看一下`epoll_dispatch`：

```c
static int
epoll_dispatch(struct event_base *base, struct timeval *tv)
{
	struct epollop *epollop = base->evbase;
	struct epoll_event *events = epollop->events;
	int i, res;
	long timeout = -1;

	if (tv != NULL) {
		timeout = evutil_tv_to_msec(tv);
		if (timeout < 0 || timeout > MAX_EPOLL_TIMEOUT_MSEC) {
			/* Linux kernels can wait forever if the timeout is
			 * too big; see comment on MAX_EPOLL_TIMEOUT_MSEC. */
			timeout = MAX_EPOLL_TIMEOUT_MSEC;
		}
	}
	
  // 将changelist里面的change读出来并进行系统调用
	epoll_apply_changes(base);
  // 读完之后remove掉changelist里面的chnge
	event_changelist_remove_all(&base->changelist, base);

	EVBASE_RELEASE_LOCK(base, th_base_lock);

  // 调用epoll_wait，一直阻塞到timeout时间过或者发生了事件
	res = epoll_wait(epollop->epfd, events, epollop->nevents, timeout);

	EVBASE_ACQUIRE_LOCK(base, th_base_lock);

	if (res == -1) {
		if (errno != EINTR) {
			event_warn("epoll_wait");
			return (-1);
		}

		return (0);
	}

	event_debug(("%s: epoll_wait reports %d", __func__, res));
	EVUTIL_ASSERT(res <= epollop->nevents);
	
  // 挨个激活epoll返回的event
	for (i = 0; i < res; i++) {
		int what = events[i].events;
		short ev = 0;

		if (what & (EPOLLHUP|EPOLLERR)) {
			ev = EV_READ | EV_WRITE;
		} else {
			if (what & EPOLLIN)
				ev |= EV_READ;
			if (what & EPOLLOUT)
				ev |= EV_WRITE;
		}

		if (!ev)
			continue;

		evmap_io_active(base, events[i].data.fd, ev | EV_ET);
	}

	if (res == epollop->nevents && epollop->nevents < MAX_NEVENT) {
		/* We used all of the event space this time.  We should
		   be ready for more events next time. */
		int new_nevents = epollop->nevents * 2;
		struct epoll_event *new_events;

		new_events = mm_realloc(epollop->events,
		    new_nevents * sizeof(struct epoll_event));
		if (new_events) {
			epollop->events = new_events;
			epollop->nevents = new_nevents;
		}
	}

	return (0);
}
```

