---
layout: post
title: 连接池设计导致的 Redis Proxy 负载不均衡
category: 工作
tags:
  - blog
  - redis
typora-root-url: ..
id: 11
---

最近一直在处理和 redis 相关的问题。在经历过线上各种 Redis 相关性能问题，并进行分析梳理给出解决方案后，对 redis 的的了解也更加深入了。

我们在线上使用的是腾讯云的 redis 服务。

其中有一个问题是，在高峰期时，redis proxy 的负载不均衡问题，此问题会同时出现在集群版与单机版上。

以下是，官网文档提供的 redis 架构图，进行参照。
![redis 单机版 架构图](/file/image/11-1.png)

从图中可以看到，在 redis 和 client 之间是有一个 proxy 集群的，client 实际上是与 proxy 建立链接的。所有的命令经由 proxy 转发到 redis 上。

我们线上的 redis 是有 32分片 2 副本的集群版 redis ，对应有 64 个 proxy 实例，在高峰时发现，各个 proxy 的 cpu 使用率并没有相近，cpu 使用率高的和 cpu 使用率低的，百分比有相差 20% 的样子。在某些极端的时候，出现了单 proxy cpu 使用率100% 的情况。造成命令延迟上升，极大的影响了整体性能。

最首先我想到的是，是创建连接时负载不均衡，proxy 连接数不一致导致的。但是查看面板资料发现他们的连接数非常接近。

咨询 腾讯云 获得回复，是连接池设计的问题。


### 定位问题

我们使用的是 [redigo](https://github.com/gomoudle/redigo)  这个库，以下使用 `v1.8.4`版本。

通过 `redis.Pool` 类型  `Get` 方法一直最终下去可以到 `func (p *Pool) GetContext(ctx context.Context) (Conn, error)` 方法

核心代码如下：
```go
// GetContext gets a connection using the provided context.
//
// The provided Context must be non-nil. If the context expires before the
// connection is complete, an error is returned. Any expiration on the context
// will not affect the returned connection.
//
// If the function completes without error, then the application must close the
// returned connection.
func (p *Pool) GetContext(ctx context.Context) (Conn, error) {
	// Wait until there is a vacant connection in the pool.
  // 判断是否等待空闲连接
  ...


	// Prune stale connections at the back of the idle list.
  // 从尾部开始清理过期的连接
  ...

	// Get idle connection from the front of idle list.
	for p.idle.front != nil {

    // 获取 idle 头 的 conn
		pc := p.idle.front
    // 将头移除
		p.idle.popFront()
		p.mu.Unlock()
    // 有效性检测
		if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
			(p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
      // 将 Pool 和 conn 包装到 activeConn
			return &activeConn{p: p, pc: pc}, nil
		}
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

	// Check for pool closed before dialing a new connection.
  // 判断 pool 是否已关闭
  ...

	// Handle limit for p.Wait == false.
  // 判断是否达到最大活跃连接数限制
  ...

  // 创建新连接
  ...
}

// Pool 核心字段
type Pool struct {
	...

	mu           sync.Mutex    // mu protects the following fields
	closed       bool          // set to true when the pool is closed.
	active       int           // the number of open connections in the pool
	initOnce     sync.Once     // the init ch once func
	ch           chan struct{} // limits open connections when p.Wait is true
	idle         idleList      // idle connections
	waitCount    int64         // total number of connections waited for.
	waitDuration time.Duration // total time waited for new connections.
}
```

从其核心代码中，其实可以看出来，每次都是从 `p.idle` 空闲队列列表的头部取出空闲连接，判断是否有效时候，包装进 `activeConn` 结构体，然后返回。

到这里已经清楚怎么取连接了，作为连接池，肯定是有借有还的。



在redigo 文档中，明确说明，在使用完连接之后必须调用其 `Close` 方法。

找到其对应代码：
```go
func (ac *activeConn) Close() (err error) {
	// 一大堆状态相关的状态取消操作
  // 
	return ac.firstError(
		err,
		err2,
		ac.p.put(pc, ac.state != 0 || pc.c.Err() != nil),
	)
}


func (p *Pool) put(pc *poolConn, forceClose bool) error {
	p.mu.Lock()
	if !p.closed && !forceClose {
		pc.t = nowFunc()

    // push 到 idle 头部
		p.idle.pushFront(pc)
		if p.idle.count > p.MaxIdle {
			pc = p.idle.back
			p.idle.popBack()
		} else {
			pc = nil
		}
	}

	if pc != nil {
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
	return nil
}

```

继续追踪下去进入了, `put` 方法中, 可以很明显看到 `p.idle.pushFront(pc)` 这一句，每个被 `Close` 的方法都会把连接放到空闲队列头部中，

到了这里可以得出一个结论：redigo 的 Pool 是一个 LIFO 模式的列表，每次从头部取 连接，Close 时归还到队列头。

现在就很好理解，因为 LIFO 模式的列表，所以会优先取栈顶的 conn，并且当你的 最大空闲数 大于你的并发数时，基本上就只会取到栈顶的几个 conn 进行接下来的 redis 操作，这导致对应 proxy cpu 使用率高于其他，极端时单 proxy 百分百 cpu 使用率。



#### redigo 需要注意的场景

- 低并发，高使用量
  例如：for 循环不停从 redis 取数据。使用一个 conn `for` 循环一直从 redis 取数据。相同 conn 只对应着一个 proxy，所有的命令都是由同一个 proxy 执行转发的。

  

### 解决方案
可以参照：https://github.com/go-redis/redis/pull/1820 ，https://github.com/gomodule/redigo/pull/571 
将 LIFO 改为 FIFO 模式。