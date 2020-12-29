---
description: 阅读libevent源码过程中的一些记录
---

# Libevent - Source Code \(2\)

### 数据结构

相关文件：

```
queue.h
```

Libevent中定义了五种数据结构：

* `singly-linked list`: 单向链表，头部指针
* `list`: 双向链表，头部指针
* `simple queue`: 头尾指针，单向链表
* `tail queue`: 头尾指针，双向链表
* `circle queue`: 头尾指针，双向循环链表

以用的最多的`tail queue`为例子看一下：

```c
/*
 * Tail queue definitions.
 */
#define TAILQ_HEAD(name, type)						\
struct name {								\
	struct type *tqh_first;	/* first element */			\
	struct type **tqh_last;	/* addr of last next element */		\
}

#define TAILQ_HEAD_INITIALIZER(head)					\
	{ NULL, &(head).tqh_first }

#define TAILQ_ENTRY(type)						\
struct {								\
	struct type *tqe_next;	/* next element */			\
	struct type **tqe_prev;	/* address of previous next element */	\
}

/*
 * tail queue access methods
 */
#define	TAILQ_FIRST(head)		((head)->tqh_first)
#define	TAILQ_END(head)			NULL
#define	TAILQ_NEXT(elm, field)		((elm)->field.tqe_next)
#define TAILQ_LAST(head, headname)					\
	(*(((struct headname *)((head)->tqh_last))->tqh_last))
/* XXX */
#define TAILQ_PREV(elm, headname, field)				\
	(*(((struct headname *)((elm)->field.tqe_prev))->tqh_last))
#define	TAILQ_EMPTY(head)						\
	(TAILQ_FIRST(head) == TAILQ_END(head))

#define TAILQ_FOREACH(var, head, field)					\
	for((var) = TAILQ_FIRST(head);					\
	    (var) != TAILQ_END(head);					\
	    (var) = TAILQ_NEXT(var, field))

#define TAILQ_FOREACH_REVERSE(var, head, headname, field)		\
	for((var) = TAILQ_LAST(head, headname);				\
	    (var) != TAILQ_END(head);					\
	    (var) = TAILQ_PREV(var, headname, field))

/*
 * Tail queue functions.
 */
#define	TAILQ_INIT(head) do {						\
	(head)->tqh_first = NULL;					\
	(head)->tqh_last = &(head)->tqh_first;				\
} while (0)

#define TAILQ_INSERT_HEAD(head, elm, field) do {			\
	if (((elm)->field.tqe_next = (head)->tqh_first) != NULL)	\
		(head)->tqh_first->field.tqe_prev =			\
		    &(elm)->field.tqe_next;				\
	else								\
		(head)->tqh_last = &(elm)->field.tqe_next;		\
	(head)->tqh_first = (elm);					\
	(elm)->field.tqe_prev = &(head)->tqh_first;			\
} while (0)

#define TAILQ_INSERT_TAIL(head, elm, field) do {			\
	(elm)->field.tqe_next = NULL;					\
	(elm)->field.tqe_prev = (head)->tqh_last;			\
	*(head)->tqh_last = (elm);					\
	(head)->tqh_last = &(elm)->field.tqe_next;			\
} while (0)

#define TAILQ_INSERT_AFTER(head, listelm, elm, field) do {		\
	if (((elm)->field.tqe_next = (listelm)->field.tqe_next) != NULL)\
		(elm)->field.tqe_next->field.tqe_prev =			\
		    &(elm)->field.tqe_next;				\
	else								\
		(head)->tqh_last = &(elm)->field.tqe_next;		\
	(listelm)->field.tqe_next = (elm);				\
	(elm)->field.tqe_prev = &(listelm)->field.tqe_next;		\
} while (0)

#define	TAILQ_INSERT_BEFORE(listelm, elm, field) do {			\
	(elm)->field.tqe_prev = (listelm)->field.tqe_prev;		\
	(elm)->field.tqe_next = (listelm);				\
	*(listelm)->field.tqe_prev = (elm);				\
	(listelm)->field.tqe_prev = &(elm)->field.tqe_next;		\
} while (0)

#define TAILQ_REMOVE(head, elm, field) do {				\
	if (((elm)->field.tqe_next) != NULL)				\
		(elm)->field.tqe_next->field.tqe_prev =			\
		    (elm)->field.tqe_prev;				\
	else								\
		(head)->tqh_last = (elm)->field.tqe_prev;		\
	*(elm)->field.tqe_prev = (elm)->field.tqe_next;			\
} while (0)

#define TAILQ_REPLACE(head, elm, elm2, field) do {			\
	if (((elm2)->field.tqe_next = (elm)->field.tqe_next) != NULL)	\
		(elm2)->field.tqe_next->field.tqe_prev =		\
		    &(elm2)->field.tqe_next;				\
	else								\
		(head)->tqh_last = &(elm2)->field.tqe_next;		\
	(elm2)->field.tqe_prev = (elm)->field.tqe_prev;			\
	*(elm2)->field.tqe_prev = (elm2);				\
} while (0)
```

这里难以理解的部分主要是entry维护一个双向链表时，没有像我们平时做的那样维护一个pre和next的一维指针，而是维护了next(一维)和pre(二维指针)，pre指向了前一个元素的next指针的地址。这样访问前一个元素的next地址时，由原本的node->pre->next变为了*(node->pre)。

这样做的好处或许是更改前一个元素的next变得简单，插入的时候直接*(node->pre) = node即可。但代价是访问pre-node变得困难。因为没有直接指向pre的一维指针。

这也是宏里面TAILQ_LAST和TAILQ_PREV非常难理解的原因。

实际上想想在这个实现上怎么访问pre，TAIL_PRE本质上是*(node->pre+8)，先通过node->pre找到前一个node的next所在的地址，然后偏移8个字节(64位的机器)拿到前一个node的pre，也就是前前个node的next所在的地址，然后取地址值就得到了前一个node。

TAILQ_LAST同理，head的last存的是last node的next的地址，TAILQ_LAST本质上是*(head->last+8)，先找到last node的next的地址，然后偏移偏移8个字节(64位的机器)拿到前一个node的pre，也就是倒数第二个node的next所在的地址，然后取地址值就得到了前一个node。

之所以不把node->pre强制转换为ENTRY而要转位HEAD的原因是ENTRY是个匿名的struct，转位HEAD能起到同样的效果。

笔者认为这样的写法增加了理解的难度，并且强制转位HEAD来完成偏移虽然巧妙但也很丑陋，不如通用写法来的直接。

如果觉得我讲得太快了可以看一下这篇博文https://blog.csdn.net/luotuo44/article/details/38374009，会比较详细还有图解。

### 参考资料

https://blog.csdn.net/luotuo44/article/details/38374009

