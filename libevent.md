# Libevent记录

关于libevent事件驱动编程的简要记录。

## 创建事件循环对象

首先使用`event_base_new()`函数创建事件循环对象。该函数返回`struct event_base*`类型，失败则返回`NULL`。如果需要复杂的控制，请先使用`event_config_new()`创建配置，再`event_base_new_with_config()`创建事件循环对象，然后使用`event_config_free()`释放配置。

## 事件

事件循环会按照某种优先级处理被加入循环的事件。

### 创建事件

`event_new()`创建事件。

```c
//宏定义，用于标志事件的类型，即下面的参数`what`
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);//回调函数类型

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

该函数基于事件循环`base`创建一个事件对象。`fd`为所观测的文件描述符，如果为负，则代表这是一个自定义事件。`what`为监听的事件类型，可以有多种组合（使用位或运算符）。`cb`为回调函数，`arg`为回调函数最后一个参数，即用户数据。

事件分为未决的、非未决的，激活的、未激活的。**事件循环会检测所有未决事件**，如果未决事件被激活，那么就会调用它的回调函数。创建事件之后，默认是非未决的，事件循环不会检测它。

事件的类型同样影响是否是未决事件。未决事件被激活后、回调函数将要执行之前，如果该事件**不具有**`EV_PERSIST`标志，事件循环会将其设置为非未决；否则，事件仍然保持未决。也就是说如果没有`EV_PERSIST`标志，未决事件在激活一次之后就不会再被事件循环检测。

你可以手动激活事件，但如果事件类型包含了`EV_TIMEOUT`，那么也可以使它在”超时“时被激活（超时时间可以在设置未决事件时一并设置，有点类似定时器）。

### 设置未决事件

使用`event_add()`将非未决事件设置为未决的。

```c
int event_add(struct event *ev, const struct timeval *tv);
```

`tv`为设置了`EV_TIMEOUT`标志的事件确定超时时间，如果为`NULL`表示该事件不会因为超时而激活。相当于延迟多久激活。

如果事件遇到超时而被激活，并且具有`EV_PERSIST`标志，那么事件超时值会被重置以等待下一次超时。

### 设置非未决事件

使用`event_del()`函数。但是如果事件已经被激活，但是回调函数还未来得及调用，那么回调函数将不再被调用。

如果不再需要该事件对象，使用`event_free()`释放。对未决和激活事件使用此函数，函数会先设置事件为非激活和非未决的。

### 手动激活事件

使用`event_active()`方法。

```c
void event_active(struct event *ev, int res, short ncalls);
```

参数`res`将被传递给回调函数；`ncalls`则是无效的，一般传递0即可。

## 开启事件循环

```c
int event_base_loop(struct event_base *eb, int flags);
```

函数`event_base_loop()`用于开启事件循环，参数`flags`可以是`EVLOOP_ONCE`，表示如果当前没有激活的事件，就等待激活事件出现，处理完激活的事件就退出；如果当前已有激活的事件，那么处理完激活的事件就退出。也可以是`EVLOOP_NONBLOCK`，表示如果当前没有激活事件就会立即退出，否则处理当前激活的事件并退出。还可以是`EVLOOP_NO_EXIT_ON_EMPTY`，表示无论如何也不退出，即便没有未决事件，因此只有`event_base_loopexit()`或者`event_base_loopbreak()`能够使其退出。

另一个函数`event_base_dispatch()`只接受一个参数`struct event_base *base`，相当于没有设置标志的`event_base_loop()`，它在没有未决事件时退出，或者通过`event_base_loopexit()`或者`event_base_loopbreak()`使其退出。