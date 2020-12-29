---
description: 阅读libevent源码过程中的一些记录
---

# Libevent - Source Code

### Log模块

相关文件：

```c
log-internal.h
log.c
```

首先看到了文件里有

```
#ifdef __GNUC__
#define EV_CHECK_FMT(a,b) __attribute__((format(printf, a, b)))
#define EV_NORETURN __attribute__((noreturn))
#endif
```

这样的宏。

`__GNUC__`指的是用GNU编译器来编译，`__attribute__((xxx))`是利用GNU编译器提供的一些特性，比如：

`__attribute__((xxx))`告诉编译器这个函数不会返回，因而编译器可以做一些优化；

`__attribute__((format(printf, a, b)))`是对a,b进行格式化检查，printf()的第一个参数是字符串格式，第二个参数是可变参数，将需要用到的变量都传进来。在函数后面加上了这个检查后会在编译期间检查出格式和可变参数是否对应；

可以看到`log-internal.h`里面出现了`...`可变参数，这是c++由c中继承来的特性，用`...`来表示可变参数。

以下是一些关于c风格的可变参数的资料：

> C/C++支持函数的可变参数列表，这个可变参数列表是通过宏来实现的，这些宏定义在头文件stdarg.h中，它是标准库的一部分。这个头文件声明了一个类型va_list和三个宏---- va_start, va_arg, va_end。我们可以声明一个类型为va_list的变量，与这几个宏配合使用，访问参数的值。
>
> 由于将va_start,va_arg,va_end定义成了宏，可变参数的类型和个数在该函数中完全由程序代码控制，并不能智能地进行识别，所以导致编译器对可变参数的函数原型检查不够严格，难于查错，不利于写出高质量的代码。要创建一个可变参数函数，必须把省略号(…)放到参数列表后面。
>
>  一个可变参数函数是指一个函数拥有不定参数，即是它接受一个可变数目的参数。**可变参数函数在C语言存在安全问题**，如C语言在没有长度检查和类型检查，在传入过少的参数或不符的类型时可能会出现溢位的情况，更可能会被利用为攻击目标。所以，在设计函数时可以先考虑其它替补方案，例如以类型安全的方式----重载。
>
> 为了编写处理不同数量实参的函数，C++11新标准提供了两种主要的方法：如果所有的实参类型相同，可以传递一个名为std::initializer_list的标准库类型；如果实参的类型不同，我们可以编写一种特殊的函数，也就是所谓的可变参数模板。

`log-internal.h`里面定义了很多不同类型不同级别的日志函数：

```c
void event_err(int eval, const char *fmt, ...) EV_CHECK_FMT(2,3) EV_NORETURN;
void event_warn(const char *fmt, ...) EV_CHECK_FMT(1,2);
void event_sock_err(int eval, evutil_socket_t sock, const char *fmt, ...) EV_CHECK_FMT(3,4) EV_NORETURN;
void event_sock_warn(evutil_socket_t sock, const char *fmt, ...) EV_CHECK_FMT(2,3);
void event_errx(int eval, const char *fmt, ...) EV_CHECK_FMT(2,3) EV_NORETURN;
void event_warnx(const char *fmt, ...) EV_CHECK_FMT(1,2);
void event_msgx(const char *fmt, ...) EV_CHECK_FMT(1,2);
void _event_debugx(const char *fmt, ...) EV_CHECK_FMT(1,2);
```

但他们的实现大同小异，都是将可变参数用va_list取出来，然后设置日志级别统一放到`_warn_helper`函数中打印

```c
void
event_err(int eval, const char *fmt, ...)
{
	va_list ap; // 定义va_list参数用于访问可变参数

	va_start(ap, fmt); // 开始访问可变参数，传入fmt标明可变参数开始的位置
	_warn_helper(_EVENT_LOG_ERR, strerror(errno), fmt, ap); // 用统一的函数打印
	va_end(ap);	// 结束对可变参数的访问
	event_exit(eval); // 由于是err，所以打印完之后要退出程序
}
```

看一下`_warn_helper`函数

```c
static void
_warn_helper(int severity, const char *errstr, const char *fmt, va_list ap)
{
	char buf[1024]; // 这里的buf定义了1024byte，意味着一条log最多只能有1024byte这么长
	size_t len;

	if (fmt != NULL)
		evutil_vsnprintf(buf, sizeof(buf), fmt, ap); // 根据fmt将msg的内容读出来存到buf里
	else
		buf[0] = '\0';

	if (errstr) {
		len = strlen(buf);
		if (len < sizeof(buf) - 3) { // -3是因为有: 和\0这三个字符
			evutil_snprintf(buf + len, sizeof(buf) - len, ": %s", errstr); // 将errstr附加到buf后面
		}
	}

	event_log(severity, buf); // 根据不同的级别打印出来
}
```

### 内存管理模块

相关文件：

```c
mm-internal.h
event.c
```

首先注意到的是

```c
#ifdef __cpluspluc
extern "C" {
#endif

// some c code

#ifdef __cpluspluc
}
#endif
```

这是在判断是否用的是c++的编译器，如果是的话声明下面的代码是c语言的，来屏蔽掉c++的一些如对象之类的属性。

主要有这几个函数

```
void *event_mm_malloc_(size_t sz); // 申请sz的内存空间
void *event_mm_calloc_(size_t count, size_t size); // 申请count*size的内存空间
char *event_mm_strdup_(const char *s); // 将string s复制到一块内存空间
void *event_mm_realloc_(void *p, size_t sz); // 将p指向的大小为sz的内存空间重新分配到一块空间
void event_mm_free_(void *p); // 释放p指向的内存空间
```

这一块的实现非常的简洁，用了`_EVENT_DISABLE_MM_REPLACEMENT`这个宏来定义是否使用用户自定义的内存管理函数，如果不使用的话则直接用`stdlib.h`里的`malloc, calloc`等，如果使用并且传入的自定义函数不为空的话就用自定义的函数。

以`malloc`为例：

```c
// mm-internal.h
#ifndef _EVENT_DISABLE_MM_REPLACEMENT
void *event_mm_malloc_(size_t sz);
#define mm_malloc(sz) event_mm_malloc_(sz)
#else
#define mm_malloc(sz) malloc(sz)
#endif

// event.c
static void *(*_mm_malloc_fn)(size_t sz) = NULL;

void *
event_mm_malloc_(size_t sz)
{
	if (_mm_malloc_fn)
		return _mm_malloc_fn(sz);
	else
		return malloc(sz);
}

void
event_set_mem_functions(void *(*malloc_fn)(size_t sz), // 设置自定义的内存分配函数
			void *(*realloc_fn)(void *ptr, size_t sz),
			void (*free_fn)(void *ptr))
{
	_mm_malloc_fn = malloc_fn;
	_mm_realloc_fn = realloc_fn;
	_mm_free_fn = free_fn;
}
```

### 锁、条件变量：

相关文件：

```c
event2/thread.h
evthread.c
evthread_pthread.c
evthread_win32.c
```

当多线程应用用到libevent的时候需要用到锁，比如有多个线程需要往一个event base里面添加或者删除event时，Libevent需要锁住它的数据结构。

跟内存管理模块类似，libevent的锁支持自己定义锁（传入需要的函数指针）或者是用默认的锁。

需要注意的是如果需要定制锁和内存管理模块，都必须在event_base创建之前把自定义的函数指针传进去。

libevent定义了两种类型的锁：

```c
/** A recursive lock is one that can be acquired multiple times at once by the
 * same thread.  No other process can allocate the lock until the thread that
 * has been holding it has unlocked it as many times as it locked it. */
#define EVTHREAD_LOCKTYPE_RECURSIVE 1
/* A read-write lock is one that allows multiple simultaneous readers, but
 * where any one writer excludes all other writers and readers. */
#define EVTHREAD_LOCKTYPE_READWRITE 2
```

以及两个数据结构，一个锁结构和一个条件变量结构：

```c
struct evthread_lock_callbacks {
	/** The current version of the locking API.  Set this to
	 * EVTHREAD_LOCK_API_VERSION */
	int lock_api_version;
	/** Which kinds of locks does this version of the locking API
	 * support?  A bitfield of EVTHREAD_LOCKTYPE_RECURSIVE and
	 * EVTHREAD_LOCKTYPE_READWRITE.
	 *
	 * (Note that RECURSIVE locks are currently mandatory, and
	 * READWRITE locks are not currently used.)
	 **/
	unsigned supported_locktypes;
	/** Function to allocate and initialize new lock of type 'locktype'.
	 * Returns NULL on failure. */
	void *(*alloc)(unsigned locktype);
	/** Funtion to release all storage held in 'lock', which was created
	 * with type 'locktype'. */
	void (*free)(void *lock, unsigned locktype);
	/** Acquire an already-allocated lock at 'lock' with mode 'mode'.
	 * Returns 0 on success, and nonzero on failure. */
	int (*lock)(unsigned mode, void *lock);
	/** Release a lock at 'lock' using mode 'mode'.  Returns 0 on success,
	 * and nonzero on failure. */
	int (*unlock)(unsigned mode, void *lock);
};

/** Sets a group of functions that Libevent should use for locking.
 * For full information on the required callback API, see the
 * documentation for the individual members of evthread_lock_callbacks.
 *
 * Note that if you're using Windows or the Pthreads threading library, you
 * probably shouldn't call this function; instead, use
 * evthread_use_windows_threads() or evthread_use_posix_threads() if you can.
 */
int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

#define EVTHREAD_CONDITION_API_VERSION 1

struct timeval;

/** This structure describes the interface a threading library uses for
 * condition variables.  It's used to tell evthread_set_condition_callbacks
 * how to use locking on this platform.
 */
struct evthread_condition_callbacks {
	/** The current version of the conditions API.  Set this to
	 * EVTHREAD_CONDITION_API_VERSION */
	int condition_api_version;
	/** Function to allocate and initialize a new condition variable.
	 * Returns the condition variable on success, and NULL on failure.
	 * The 'condtype' argument will be 0 with this API version.
	 */
	void *(*alloc_condition)(unsigned condtype);
	/** Function to free a condition variable. */
	void (*free_condition)(void *cond);
	/** Function to signal a condition variable.  If 'broadcast' is 1, all
	 * threads waiting on 'cond' should be woken; otherwise, only on one
	 * thread is worken.  Should return 0 on success, -1 on failure.
	 * This function will only be called while holding the associated
	 * lock for the condition.
	 */
	int (*signal_condition)(void *cond, int broadcast);
	/** Function to wait for a condition variable.  The lock 'lock'
	 * will be held when this function is called; should be released
	 * while waiting for the condition to be come signalled, and
	 * should be held again when this function returns.
	 * If timeout is provided, it is interval of seconds to wait for
	 * the event to become signalled; if it is NULL, the function
	 * should wait indefinitely.
	 *
	 * The function should return -1 on error; 0 if the condition
	 * was signalled, or 1 on a timeout. */
	int (*wait_condition)(void *cond, void *lock,
	    const struct timeval *timeout);
};
```

并且提供了设置锁和条件变量的方法，以及设置获取锁id的方法，如果要定制锁的话，自己写好这些方法传进去：

```c
int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

int evthread_set_condition_callbacks(
	const struct evthread_condition_callbacks *);

void evthread_set_id_callback(
    unsigned long (*id_fn)(void));
```

由此可以看出libevent的锁模块通过两个set方法将锁和条件变量的实现传入，然后在使用的地方调用这些接口。基本跟内存管理的方式一样。

如果不需要定制锁，只要用默认的锁实现的话，libevent针对两种平台提供了win和posix两种实现：

```c
// evthread_pthread.c
int
evthread_use_pthreads(void)
{
	struct evthread_lock_callbacks cbs = {
		EVTHREAD_LOCK_API_VERSION,
		EVTHREAD_LOCKTYPE_RECURSIVE,
		evthread_posix_lock_alloc,
		evthread_posix_lock_free,
		evthread_posix_lock,
		evthread_posix_unlock
	};
	struct evthread_condition_callbacks cond_cbs = {
		EVTHREAD_CONDITION_API_VERSION,
		evthread_posix_cond_alloc,
		evthread_posix_cond_free,
		evthread_posix_cond_signal,
		evthread_posix_cond_wait
	};
	/* Set ourselves up to get recursive locks. */
	if (pthread_mutexattr_init(&attr_recursive))
		return -1;
	if (pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE))
		return -1;

	evthread_set_lock_callbacks(&cbs);
	evthread_set_condition_callbacks(&cond_cbs);
	evthread_set_id_callback(evthread_posix_get_id);
	return 0;
}

// evthread_win32.c
int
evthread_use_windows_threads(void)
{
  // 略
}
```

阅读`evthread.c`时，发现libevent的锁支持了debug。如果调用了`evthread_enable_lock_debuging`这个方法将会在原本的锁外面包一层debug，当锁发生常见的错误时将异常抛出。

```c
// evthread.c
void
evthread_enable_lock_debuging(void)
{
	struct evthread_lock_callbacks cbs = {
		EVTHREAD_LOCK_API_VERSION,
		EVTHREAD_LOCKTYPE_RECURSIVE,
		debug_lock_alloc,
		debug_lock_free,
		debug_lock_lock,
		debug_lock_unlock
	};
	if (_evthread_lock_debugging_enabled) // 如果已经支持了debug，就不需要再设置
		return;
	memcpy(&_original_lock_fns, &_evthread_lock_fns, // 将原本用的锁fns拷贝保存
	    sizeof(struct evthread_lock_callbacks));
	memcpy(&_evthread_lock_fns, &cbs, // 定义新的锁fns
	    sizeof(struct evthread_lock_callbacks));

	memcpy(&_original_cond_fns, &_evthread_cond_fns, // 将原本用的条件fns拷贝保存
	    sizeof(struct evthread_condition_callbacks));
	_evthread_cond_fns.wait_condition = debug_cond_wait; // 标记为debug
	_evthread_lock_debugging_enabled = 1;

	/* XXX return value should get checked. */
	event_global_setup_locks_(0);
}
```

点开看`debug_lock_alloc, debug_lock_free, debug_lock_lock, debug_lock_unlock`这几个函数，基本是在原本的lock外面封装了一些检查。

```c
struct debug_lock {
	unsigned locktype; // 记录锁原本的类型，debug的锁申请的都会是可重入锁，非可重入锁将会在debug层进行控制和检查
	unsigned long held_by; // 记录锁被哪个线程持有，保存线程id，在可重入和释放锁的时候检查held_by是否等于当前线程的id
	/* XXXX if we ever use read-write locks, we will need a separate
	 * lock to protect count. */
	int count; // 记录可重入锁被进入的次数，若锁原本是非可重入锁，会检查这个值小于1
	void *lock; // 保存真正的锁
};
```

再看一下`debug_cond_wait`

```c
static int
debug_cond_wait(void *_cond, void *_lock, const struct timeval *tv)
{
	int r;
	struct debug_lock *lock = _lock;
	EVUTIL_ASSERT(lock); 
	EVLOCK_ASSERT_LOCKED(_lock); // 检查lock
	evthread_debug_lock_mark_unlocked(0, lock); // 记录lock被释放了
	r = _original_cond_fns.wait_condition(_cond, lock->lock, tv); // 原本的cond_wait
	evthread_debug_lock_mark_locked(0, lock); // 记录lock被获取了
	return r;
}
```

也就是在原本的cond_wait的基础上检查调用前的释放锁和调用后的获取锁的状态是否正确。

### 锁的使用

相关文件：

```
evthread-internal.h
```

Libevent中，一些函数支持多线程。一般都是使用锁进行线程同步。在Libevent的代码中，一般是使用EVTHREAD_ALLOC_LOCK宏获取一个锁变量，EVBASE_ACQUIRE_LOCK宏进行加锁，EVBASE_RELEASE_LOCK宏进行解锁。在阅读Libevent源代码中，一般都只会看到EVBASE_ACQUIRE_LOCK和EVBASE_RELEASE_LOCK。锁的内部实现是看不见的。

```c

/** Return the ID of the current thread, or 1 if threading isn't enabled. */
#define EVTHREAD_GET_ID() \
	(_evthread_id_fn ? _evthread_id_fn() : 1)

/** Return true iff we're in the thread that is currently (or most recently)
 * running a given event_base's loop. Requires lock. */
#define EVBASE_IN_THREAD(base)				 \
	(_evthread_id_fn == NULL ||			 \
	(base)->th_owner_id == _evthread_id_fn())
```

基本是通过宏对要用到的锁操作进行封装。

### 其他

经常在代码中的宏定义里面看到`do {xxx} while(0)`对于它的作用感到困惑，其实这是在宏定义里面的一个常用写法，类似于`if(1) {xxx} else`，作用是类似`{xxx}`一样将代码合为一个整体，避免宏替换的时候出现编译错误。

详见：

https://stackoverflow.com/questions/154136/why-use-apparently-meaningless-do-while-and-if-else-statements-in-macros

### 参考资料

https://blog.csdn.net/luotuo44/category_9262895.html

http://www.wangafu.net/~nickm/libevent-book/Ref1_libsetup.html