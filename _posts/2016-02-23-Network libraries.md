# Network libraries #
---
网络库的作用是什么，大抵就是将程序所关心的事件在它就绪的时候通知程序。Reactor和Proactor模式，边缘(ET)和水平(LT)触发，都是其不同的工作方式。本文主要是记录下不同网络库的差异，并对部分组件实现进行对比。
libevent作为一款优秀的开源网络库其源码的分析自然必不可少，但是我觉得更有趣，更简洁的是skynet的socket server，因为skynet的需求更加简单，所以它的实现没那么复杂。
```c
struct socket_server {
	int recvctrl_fd;
	int sendctrl_fd;
	int checkctrl;
	poll_fd event_fd;
	int alloc_id;
	int event_n;
	int event_index;
	struct socket_object_interface soi;
	struct event ev[MAX_EVENT];
	struct socket slot[MAX_SOCKET];
	char buffer[MAX_INFO];
	uint8_t udpbuffer[MAX_UDP_PACKAGE];
	fd_set rfds;
};
```
其中recvctrl_fd和sendctrl_fd是通过pipe调用得到，用来其它线程向I/O线程传输信息。其它网络库一般都是用来在主线程和I/O线程建立通信通道，以达到在某些时刻唤醒I/O线程的目的。skynet的用法也类似只不过传输的是一些控制信息。关键的socket结构如下：
```c
struct write_buffer {
	struct write_buffer * next;
	void *buffer;
	char *ptr;
	int sz;
	bool userobject;
	uint8_t udp_address[UDP_ADDRESS_SIZE];
};
struct wb_list {
	struct write_buffer * head;
	struct write_buffer * tail;
};
struct socket {
	uintptr_t opaque;
	struct wb_list high;
	struct wb_list low;
	int64_t wb_size;
	int fd;
	int id;
	uint16_t protocol;
	uint16_t type;
	union {
		int size;
		uint8_t udp_address[UDP_ADDRESS_SIZE];
	} p;
};
```
opaque存储的是服务(context)地址，high和low分别对应高低优先级的写入通道，struct wb_list是一个简单的单向列表，fd是文件描述符，id是fd经过转化后的数值，上层程序只能得到id来索引对应的socket。转换函数为**reserve_id**，通过自增socket_server的字段alloc_id，索引socket数组slot以得到一个空闲的socket，所以id就是slot数组的下标值。现在简单分析下循环函数socket_server_poll。
```c
int 
socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more) {
	for (;;) {
		if (ss->checkctrl) {
			if (has_cmd(ss)) {
				int type = ctrl_cmd(ss, result);
				if (type != -1) {
					clear_closed_event(ss, result, type);
					return type;
				} else
					continue;
			} else {
				ss->checkctrl = 0;
			}
		}
		if (ss->event_index == ss->event_n) {
			ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);
			ss->checkctrl = 1;
			if (more) {
				*more = 0;
			}
			ss->event_index = 0;
			if (ss->event_n <= 0) {
				ss->event_n = 0;
				return -1;
			}
		}
		struct event *e = &ss->ev[ss->event_index++];
		struct socket *s = e->s;
		if (s == NULL) {
			// dispatch pipe message at beginning
			continue;
		}
		switch (s->type) {
		case SOCKET_TYPE_CONNECTING:
			return report_connect(ss, s, result);
		case SOCKET_TYPE_LISTEN: {
			int ok = report_accept(ss, s, result);
			if (ok > 0) {
				return SOCKET_ACCEPT;
			} if (ok < 0 ) {
				return SOCKET_ERROR;
			}
			// when ok == 0, retry
			break;
		}
		case SOCKET_TYPE_INVALID:
			fprintf(stderr, "socket-server: invalid socket\n");
			break;
		default:
			if (e->read) {
				int type;
				if (s->protocol == PROTOCOL_TCP) {
					type = forward_message_tcp(ss, s, result);
				} else {
					type = forward_message_udp(ss, s, result);
					if (type == SOCKET_UDP) {
						// try read again
						--ss->event_index;
						return SOCKET_UDP;
					}
				}
				if (e->write && type != SOCKET_CLOSE && type != SOCKET_ERROR) {
					// Try to dispatch write message next step if write flag set.
					e->read = false;
					--ss->event_index;
				}
				if (type == -1)
					break;				
				return type;
			}
			if (e->write) {
				int type = send_buffer(ss, s, result);
				if (type == -1)
					break;
				return type;
			}
			break;
		}
	}
}
```
ctrl_cmd用来处理上层函数对网络库的控制命令，每个命令信息的前两个字节用来表明命令的type和命令结构的大小len，type包括Start,Bind,Listen socket等，这里不展开讨论。监听命令事件和其它I/O事件是不同的模型，select和epoll。
socket server还记录了就绪事件的索引，
>if (ss->event_index == ss->event_n) 

当前处理的事件索引等于就绪事件的大小时，说明上次sp_wait返回的就绪事件已经被处理完了，需要重新调用sp_wait，填充ev数组。否则就可以直接从ev数组读取下一个就绪的event。从代码里可以得到，每次调用socket_server_poll都只处理一次非命令事件，但是可以处理多个命令事件。
在处理写事件的时候，skynet设计了高低优先通道，其设计思路是：
>只有等默认通道（高优先级）的包全部发送完后，低优先级通道上的包才至少被发送一个（单个包可以保证原子性）。
比如，你可以用它来发送聊天信息，就不会因为聊天信息泛滥把其它重要数据包都塞住。同样，你可以用来发送被分割后的大数据块。如果同时你还有很多其它重要的数据需要传输给客户端，那么这些数据块就会被打散穿插在其间。
当然，你也可以把所有给客户端的数据全部用 lwrite 发送，而仅仅把心跳包放在常规高优先级通道，可以保证心跳频率更稳定。

整个实现继承了上述设计。
```c
/*
	Each socket has two write buffer list, high priority and low priority.

	1. send high list as far as possible.
	2. If high list is empty, try to send low list.
	3. If low list head is uncomplete (send a part before), move the head of low list to empty high list (call raise_uncomplete) .
	4. If two lists are both empty, turn off the event. (call check_close)
 */
static int
send_buffer(struct socket_server *ss, struct socket *s, struct socket_message *result) {
	assert(!list_uncomplete(&s->low));
	// step 1
	if (send_list(ss,s,&s->high,result) == SOCKET_CLOSE) {
		return SOCKET_CLOSE;
	}
	if (s->high.head == NULL) {
		// step 2
		if (s->low.head != NULL) {
			if (send_list(ss,s,&s->low,result) == SOCKET_CLOSE) {
				return SOCKET_CLOSE;
			}
			// step 3
			if (list_uncomplete(&s->low)) {
				raise_uncomplete(s);
			}
		} else {
			// step 4
			sp_write(ss->event_fd, s->fd, s, false);

			if (s->type == SOCKET_TYPE_HALFCLOSE) {
				force_close(ss, s, result);
				return SOCKET_CLOSE;
			}
		}
	}

	return -1;
}
```
这里需要注意的一种情况是在发送low list的时候，如果没有发送完，会将对应的write_buffer的剩余内容转移到high list上，是因为它保证同一个write_buffer上面的数据会被连续的发送。这个设计应该是对比其它网络库最大的不同了，不过也是因为基于游戏领域的特殊性，当然在使用上也可以都用同一个优先级，这样就无差别了。

libevent对比skynet来说就全面的多，实现也复杂的多。详细的解剖有[这里-基础全面](http://blog.csdn.net/sparkliang/article/category/660506) 和 [这里-进阶无敌](http://blog.csdn.net/luotuo44/article/category/2435521/1)。
关于定时器的实现，之前的libevent使用小根堆管理这些超时event。在2.0.4-alpha版本时，Libevent引入了一个叫common-timeout的东西来配合管理超时event。使用common-timeout能够更加高效的前提是定时器中有比较多具有相同时长的定时任务。**简单的来说，把这些具有相同定时时长的定时任务按照触发时间的先后顺序组成一个链表，同时添加一个最小超时时间的event到小根堆当中，剩下的就是管理小根堆，如果该event触发了，就需要重新判断是否再次添加到小根堆中。**
```c
//event-internal.h文件
struct event_base {
	//因为可以有多个不同时长的超时event组。故得是数组
	//因为数组元素是common_timeout_list指针，所以得是二级指针
	struct common_timeout_list **common_timeout_queues;
	//数组元素个数
	int n_common_timeouts;
	//已分配的数组元素个数
	int n_common_timeouts_allocated;
};

struct common_timeout_list {
	//超时event队列。将所有具有相同超时时长的超时event放到一个队列里面
	struct event_list events;

	struct timeval duration;//超时时长
	struct event timeout_event;//具有相同超时时长的超时event代表
	struct event_base *base;
};

```
common_timeout是要用户手动去操作同时设定哪些是自己的常用超时时间，并加入common_timeout的标志.
```c
//event.c文件
#define COMMON_TIMEOUT_MICROSECONDS_MASK       0x000fffff
#define MICROSECONDS_MASK       COMMON_TIMEOUT_MICROSECONDS_MASK
#define COMMON_TIMEOUT_IDX_MASK 0x0ff00000
#define COMMON_TIMEOUT_IDX_SHIFT 20
#define COMMON_TIMEOUT_MASK     0xf0000000
#define COMMON_TIMEOUT_MAGIC    0x50000000

#define COMMON_TIMEOUT_IDX(tv) \
	(((tv)->tv_usec & COMMON_TIMEOUT_IDX_MASK)>>COMMON_TIMEOUT_IDX_SHIFT)

#define MAX_COMMON_TIMEOUTS 256

static inline int
is_common_timeout(const struct timeval *tv,
    const struct event_base *base)
{
	int idx;
	//不具有common-timeout标志位，那么就肯定不是commont-timeout时间了
	if ((tv->tv_usec & COMMON_TIMEOUT_MASK) != COMMON_TIMEOUT_MAGIC)
		return 0;

	idx = COMMON_TIMEOUT_IDX(tv);//获取数组下标
	return idx < base->n_common_timeouts;
}
```
[参考](http://blog.csdn.net/luotuo44/article/details/38678333)

《linux多线程服务端编程》是我对网络库的启蒙教材，书中通过对muduo网络库的讲解让我明白了网络库做了哪些事情，以及怎么实现一个网络库，当然也了解了一些基本知识。关于资源竞争和唤醒线程的知识让我印象深刻。
```c
#ifdef __GXX_EXPERIMENTAL_CXX0X__
// FIXME: remove duplication
void EventLoop::runInLoop(Functor&& cb)
{
  if (isInLoopThread())
  {
    cb();
  }
  else
  {
    queueInLoop(std::move(cb));
  }
}

void EventLoop::queueInLoop(Functor&& cb)
{
  {
  MutexLockGuard lock(mutex_);
  pendingFunctors_.push_back(std::move(cb));  // emplace_back
  }

  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}
```

上面的代码就展示了非I/O线程的线程处理和锁的使用(RAII封装过)。