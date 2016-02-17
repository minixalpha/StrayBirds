 # Timing-Wheel service #
---
定时器作为一种基础组件在各个系统中都必不可少。而定时器的实现方式有很多种，了解更多的linux下定时器的实现方式请看[这里](https://www.ibm.com/developerworks/cn/linux/l-cn-timers/#icomments)。在链接的文章关于各个实现方式的效率对比中，发现基于时间轮(Timing-Wheel)方式的实现是在理论意义上效率最高的。作为一个基于时间轮方式实现的时钟系统，我们来简单的分析一下[skynet的时钟系统](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_timer.c)。
## 数据结构 ##
### 定时器 ###
```c
struct link_list {
	struct timer_node head;
	struct timer_node *tail;
};
struct timer {
	struct link_list near[TIME_NEAR];
	struct link_list t[4][TIME_LEVEL];
	struct spinlock lock;
	uint32_t time;
	uint32_t starttime;
	uint64_t current;
	uint64_t current_point;
};
```
timer结构体中的**near**字段存储的是与当前时间在TIME_NEAR-1范围内触发的定时事件，而字段**t**的设置是跟TIME_NEAR和TIME_LEVEL的属性有关
```c
#define TIME_NEAR_SHIFT 8
#define TIME_NEAR (1 << TIME_NEAR_SHIFT)
#define TIME_LEVEL_SHIFT 6
#define TIME_LEVEL (1 << TIME_LEVEL_SHIFT)
#define TIME_NEAR_MASK (TIME_NEAR-1)
#define TIME_LEVEL_MASK (TIME_LEVEL-1)
```
对于一个需要支持0-2^32-1ticks范围内的定时器，系统并不需要开辟与其相等的空间来表示每一个tick。这里skynet和linux采取的方式都是一样的，把定时器分为5组，每组的粒度分别为1，256，256\*64，256\*64\*64，256\*64\*64\*64，每组的数量对应为256，64，64，64，64，这样只需256+64+64+64+64 = 512个空间就能表示2^32的范围(**为什么要这样分组**)。所以字段**t**的一维空间0-3表示4个层级，数字越大它对应的粒度也就越大。
时钟系统最重要的就是添加定时事件和触发定时事件，我们来看其对应实现。
```c
static void
add_node(struct timer *T,struct timer_node *node) {
	uint32_t time=node->expire;
	uint32_t current_time=T->time;
	
	if ((time|TIME_NEAR_MASK)==(current_time|TIME_NEAR_MASK)) {
		link(&T->near[time&TIME_NEAR_MASK],node);
	} else {
		int i;
		uint32_t mask=TIME_NEAR << TIME_LEVEL_SHIFT;
		for (i=0;i<3;i++) {
			if ((time|(mask-1))==(current_time|(mask-1))) {
				break;
			}
			mask <<= TIME_LEVEL_SHIFT;
		}

		link(&T->t[i][((time>>(TIME_NEAR_SHIFT + i*TIME_LEVEL_SHIFT)) & TIME_LEVEL_MASK)],node);	
	}
}
```
这个函数首先将当前时间和触发时间比较是否在TIME_NEAR_MASK范围内
>(time|TIME_NEAR_MASK)==(current_time|TIME_NEAR_MASK)

利用**|**运算将数值的低8位全部置为1，如果其它数位一致的话，说明触发时间与当前时间的差距只在低8位，也就是TIME_NEAR_MASK范围内，因此link到near数组。同上面的操作一样，只是在else分支里面，需要进一步判断其所在的层级
>uint32_t mask=TIME_NEAR << TIME_LEVEL_SHIFT;

然后link到对应的位置上。
我们看到在上面的实现当中，每一个定时事件都只是link到一个分组里面，而有的分组的粒度很大，随着时间的推移如何将粒度大的分组里的事件转移到粒度为1的**near**上呢，也就是**update**函数。要注意的是时钟系统中存储的定时事件保存的是一个时间戳，根据这个时间戳与当前时间的分组关系确定该事件该link到那个分组，只有该分组粒度大小的时间变化时，这个位置才有可能会变化，下文会讲到。
```c
static void 
timer_update(struct timer *T) {
	SPIN_LOCK(T);

	// try to dispatch timeout 0 (rare condition)
	timer_execute(T);

	// shift time first, and then dispatch timer message
	timer_shift(T);

	timer_execute(T);

	SPIN_UNLOCK(T);
}
```
**timer_execute**做的事情很简单，在链表中删除timeout事件，然后将其推送到对应服务的消息队列上。
```c
static void
move_list(struct timer *T, int level, int idx) {
	struct timer_node *current = link_clear(&T->t[level][idx]);
	while (current) {
		struct timer_node *temp=current->next;
		add_node(T,current);
		current=temp;
	}
}

static void
timer_shift(struct timer *T) {
	int mask = TIME_NEAR;
	uint32_t ct = ++T->time;
	if (ct == 0) {
		move_list(T, 3, 0);
	} else {
		uint32_t time = ct >> TIME_NEAR_SHIFT;
		int i=0;

		while ((ct & (mask-1))==0) {
			int idx=time & TIME_LEVEL_MASK;
			if (idx!=0) {
				move_list(T, i, idx);
				break;				
			}
			mask <<= TIME_LEVEL_SHIFT;
			time >>= TIME_LEVEL_SHIFT;
			++i;
		}
	}
}
```
timer_shift函数先自加T->time，
>while ((ct & (mask-1))==0)

判断**当前时间**在自增的过程中是否刚好走过了某一个分组的粒度也就是mask-1，如果是的话，再看**time**在上一层粒度的分组的值是不是等于0，是的话就说明当前判断的粒度分组也完成了一个循环需要再往上层寻找满足条件的分组，否则的话执行**move_list**函数。其实就是进制间的关系，个位数满十了十位数就要进1一样，而在这里**near**就相当于个位数，当**near**数组循环了一遍之后，它的上一层的数值就要加一，然后对应位置上的定时事件就要重新**add_node**以寻找合适的位置。前面说过每个定时事件添加进来的时候都会判断一个其所link的位置，而这个位置的分组就是**定时时间**与**当前时间**的最小共同分组，**而该事件再一次改变link位置是在time在最小共同分组的位置和定时时间一样的时候**，也就是在满足上面循环条件的时候，**time**在上一层粒度的分组的值。move_list函数将给定的位置上的分组从原来的位置unlink，然后对每个定时事件重新进行**add_node**操作。
在skynet的实现中，线程不断的循环这样的操作。
```c
void
skynet_updatetime(void) {
	uint64_t cp = gettime();
	if(cp < TI->current_point) {
		skynet_error(NULL, "time diff error: change from %lld to %lld", cp, TI->current_point);
		TI->current_point = cp;
	} else if (cp != TI->current_point) {
		uint32_t diff = (uint32_t)(cp - TI->current_point);
		TI->current_point = cp;
		TI->current += diff;
		int i;
		for (i=0;i<diff;i++) {
			timer_update(TI);
		}
	}
}

```
以上不难得到它的执行流程。
