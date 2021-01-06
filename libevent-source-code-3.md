---
description: 阅读libevent源码过程中的一些记录
---

# Libevent - Source Code \(3\)

\#\#\# 配置event\_base

相关文件：

```c
event.h
event-internal.h
event.c
```

初始化出一个`event_base`有有两种方法，`event_base_new()`和`event_base_new_with_config()`

先看`event_base_new()`:

```c
// event.c
struct event_base *
event_base_new(void)
{
    struct event_base *base = NULL;
    struct event_config *cfg = event_config_new();
    if (cfg) {
        base = event_base_new_with_config(cfg);
        event_config_free(cfg);
    }
    return base;
}
```

实际上是new除了一个默认的config然后调`event_base_new_with_config()`，看看`event_config_new（）`里面主要是初始化了一个`event_config`

```c
// event-internal.h
/** Internal structure: describes the configuration we want for an event_base
 * that we're about to allocate. */
struct event_config {
    TAILQ_HEAD(event_configq, event_config_entry) entries;

    int n_cpus_hint;
    enum event_method_feature require_features;
    enum event_base_config_flag flags;
};c
```

向`entries`里面添加`avoid_method`：

```c
// event.c
int
event_config_avoid_method(struct event_config *cfg, const char *method)
{
    struct event_config_entry *entry = mm_malloc(sizeof(*entry)); // 申请一块内存
    if (entry == NULL)
        return (-1);

    if ((entry->avoid_method = mm_strdup(method)) == NULL) {    // 把屏蔽的method复制进去
        mm_free(entry);
        return (-1);
    }

    TAILQ_INSERT_TAIL(&cfg->entries, entry, next);    // 插入到cfg->entries里

    return (0);
}
```

能够看出这个配置的作用是避免使用指定的多路IO复用函数

> 查看Libevent源码包里的文件，可以发现有诸如epoll.c、select.c、poll.c、devpoll.c、kqueue.c这些文件。打开这些文件就可以发现在文件内容的前面都会定义一个struct eventop类型变量。该结构体的第一个成员必然是一个字符串。这个字符串就描述了对应的多路IO复用函数的名称。所以是可以通过名称来禁用某种多路IO复用函数的。

设置`n_cpus_hint`，来方便libevent作为参考用核。这只是一个提示，实际libevent使用的核可能比这个多或者少

```c
int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus);
```

设置`require_features`，配置需要的功能，用或传进去

```c
int event_config_require_features(struct event_config *cfg, int features);

//event.h文件
enum event_method_feature {
    //支持边沿触发
    EV_FEATURE_ET = 0x01,
    //添加、删除、或者确定哪个事件激活这些动作的时间复杂度都为O(1)
    //select、poll是不能满足这个特征的.epoll则满足
    EV_FEATURE_O1 = 0x02,
    //支持任意的文件描述符，而不能仅仅支持套接字
    EV_FEATURE_FDS = 0x04
};
```

设置flags，配置需要的，用或传进去

```c
int event_config_set_flag(struct event_config *cfg, int flag);

enum event_base_config_flag {
    //不要为event_base分配锁。设置这个选项可以为event_base节省一点加锁和解锁的时间，但是当多个线程访问event_base会变得不安全
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    //选择多路IO复用函数时，不检测EVENT_*环境变量。使用这个标志要考虑清楚：因为这会使得用户更难调试程序与Libevent之间的交互
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    //仅用于Windows。这使得Libevent在启动时就启用任何必需的IOCP分发逻辑，而不是按需启用。如果设置了这个宏，那么evconn_listener_new和bufferevent_socket_new函数的内部将使用IOCP
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    //在执行event_base_loop的时候没有cache时间。该函数的while循环会经常取系统时间，如果cache时间，那么就取cache的。如果没有的话，就只能通过系统提供的函数来获取系统时间。这将更耗时
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    //告知Libevent，如果决定使用epoll这个多路IO复用函数，可以安全地使用更快的基于changelist 的多路IO复用函数：epoll-changelist多路IO复用可以在多路IO复用函数调用之间，同样的fd 多次修改其状态的情况下，避免不必要的系统调用。但是如果传递任何使用dup()或者其变体克隆的fd给Libevent，epoll-changelist多路IO复用函数会触发一个内核bug，导致不正确的结果。在不使用epoll这个多路IO复用函数的情况下，这个标志是没有效果的。也可以通过设置EVENT_EPOLL_USE_CHANGELIST 环境变量来打开epoll-changelist选项
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10
};
```

## 跨平台Reactor接口

```c
struct event_base {
    /** Function pointers and other data to describe this event_base's
     * backend. */
    const struct eventop *evsel;
    /** Pointer to backend-specific data. */
    void *evbase;
  ...
}

/** Structure to define the backend of a given event_base. */
struct eventop {
    /** The name of this backend. */
    const char *name;
    /** Function to set up an event_base to use this backend.  It should
     * create a new structure holding whatever information is needed to
     * run the backend, and return it.  The returned pointer will get
     * stored by event_init into the event_base.evbase field.  On failure,
     * this function should return NULL. */
    void *(*init)(struct event_base *);
    /** Enable reading/writing on a given fd or signal.  'events' will be
     * the events that we're trying to enable: one or more of EV_READ,
     * EV_WRITE, EV_SIGNAL, and EV_ET.  'old' will be those events that
     * were enabled on this fd previously.  'fdinfo' will be a structure
     * associated with the fd by the evmap; its size is defined by the
     * fdinfo field below.  It will be set to 0 the first time the fd is
     * added.  The function should return 0 on success and -1 on error.
     */
    int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    /** As "add", except 'events' contains the events we mean to disable. */
    int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    /** Function to implement the core of an event loop.  It must see which
        added events are ready, and cause event_active to be called for each
        active event (usually via event_io_active or such).  It should
        return 0 on success and -1 on error.
     */
    int (*dispatch)(struct event_base *, struct timeval *);
    /** Function to clean up and free our data from the event_base. */
    void (*dealloc)(struct event_base *);
    /** Flag: set if we need to reinitialize the event base after we fork.
     */
    int need_reinit;
    /** Bit-array of supported event_method_features that this backend can
     * provide. */
    enum event_method_feature features;
    /** Length of the extra information we should record for each fd that
        has one or more active events.  This information is recorded
        as part of the evmap entry for each fd, and passed as an argument
        to the add and del functions above.
     */
    size_t fdinfo_len;
};
```

可以看出这里跟前面的log, lock等一样，定义了一个`eventop`存放所有对`event`的操作的函数指针，在每个函数的文件里面会有针对这些函数的实现，如`epoll, select`等。然后每个`event_base`根据创建的时候传入的`config`选择具体要用哪个`eventop`。

看一下这部分是如何实现的：

在`event_base_new_with_config`里面有对`eventop`的选择

```c
// event.c

struct event_base *
event_base_new_with_config(const struct event_config *cfg)
{
    ...
    for (i = 0; eventops[i] && !base->evbase; i++) {
        if (cfg != NULL) {
            /* determine if this backend should be avoided */
            if (event_config_is_avoided_method(cfg,
                eventops[i]->name)) // 用FOREACH遍历cfg->entries看这个方法是不是被屏蔽掉了
                continue;
            if ((eventops[i]->features & cfg->require_features)
                != cfg->require_features) // 看这个方法是否满足用户需要的feature
                continue;
        }

        /* also obey the environment variables */
        if (should_check_environment &&
            event_is_method_disabled(eventops[i]->name)) // 看环境变量有没有屏蔽某个方法
            continue;

        base->evsel = eventops[i]; // 确定了方法之后把它放进event_base里

        base->evbase = base->evsel->init(base);
    }
    ...
}
```

`eventops`是怎么来的呢，基本是判断系统里有没有某个方法，然后加进一个数组里。把效率高的放在前面优先用

```c
#ifdef _EVENT_HAVE_EVENT_PORTS
extern const struct eventop evportops;
#endif
#ifdef _EVENT_HAVE_SELECT
extern const struct eventop selectops;
#endif
#ifdef _EVENT_HAVE_POLL
extern const struct eventop pollops;
#endif
#ifdef _EVENT_HAVE_EPOLL
extern const struct eventop epollops;
#endif
#ifdef _EVENT_HAVE_WORKING_KQUEUE
extern const struct eventop kqops;
#endif
#ifdef _EVENT_HAVE_DEVPOLL
extern const struct eventop devpollops;
#endif
#ifdef WIN32
extern const struct eventop win32ops;
#endif

/* Array of backends in order of preference. */
static const struct eventop *eventops[] = {
#ifdef _EVENT_HAVE_EVENT_PORTS
    &evportops,
#endif
#ifdef _EVENT_HAVE_WORKING_KQUEUE
    &kqops,
#endif
#ifdef _EVENT_HAVE_EPOLL
    &epollops,
#endif
#ifdef _EVENT_HAVE_DEVPOLL
    &devpollops,
#endif
#ifdef _EVENT_HAVE_POLL
    &pollops,
#endif
#ifdef _EVENT_HAVE_SELECT
    &selectops,
#endif
#ifdef WIN32
    &win32ops,
#endif
    NULL
};
```

## 参考资料

[https://blog.csdn.net/luotuo44/category\_9262895.html](https://blog.csdn.net/luotuo44/category_9262895.html)

[http://www.wangafu.net/~nickm/libevent-book/Ref2\_eventbase.html](http://www.wangafu.net/~nickm/libevent-book/Ref2_eventbase.html)

