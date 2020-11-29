# 分批发射器 batchingEmitter

**分批发射器**的实现是为了解决来一条数据就处理一次所带来的效率问题。采用的思想是，将收到的数据累积到一定数量后再进行处理或者定时器超时进行就处理。  
我更喜欢将分批发射器叫做分批处理器。

## 接口定义

```
type batchingEmitter interface {
	// Add 添加要批处理的消息
	Add(interface{})

	// Stop 停止组件
	Stop()

	// Size 返回要转发的挂起消息的数量
	Size() int
}
```
batchingEmitter 被用于gossip的推送或转发阶段。消息被添加到batchingEmitter中，它们被周期性地分批转发T次，然后被丢弃。如果batchingEmitter存储的消息计数达到一定的容量，那么也会触发消息转发。

## batchingEmitterImpl 实现

```
type batchingEmitterImpl struct {
	iterations int               // 消息转发的次数
	burstSize  int               // 触发转发的阈值
	delay      time.Duration     // 定时触发转发的时间
	cb         emitBatchCallback // 进行转发而调用的回调
	lock       *sync.Mutex
	buff       []*batchedMessage // 待处理的消息
	stopFlag   int32
}

type batchedMessage struct {
	data           interface{}
	iterationsLeft int // 消息转发的剩余次数
}
```

注：消息并不是只能转发（处理）一次，在Emitter中，一条消息可能被转发多次，转发次数由用户配置文件设置。

导致分批发射器触发主要方式：**定时触发**、**消息计数达到转发的阈值**

### 定时触发

```
func (p *batchingEmitterImpl) periodicEmit() {
	for !p.toDie() {
		time.Sleep(p.delay)
		p.lock.Lock()
		p.emit()
		p.lock.Unlock()
	}
}
```

每间隔**delay**时间，就调用emit()进行消息处理。

### 消息计数达到转发的阈值

```
func (p *batchingEmitterImpl) Add(message interface{}) {
	if p.iterations == 0 {
		return
	}
	p.lock.Lock()
	defer p.lock.Unlock()

	p.buff = append(p.buff, &batchedMessage{data: message, iterationsLeft: p.iterations})

	if len(p.buff) >= p.burstSize {
		// 达到触发转发的阈值，直接转发
		p.emit()
	}
}
```

若是当前**存储的消息个数**达到设置的**阈值**，则直接调用emit()函数进行消息处理。

### emit

```
func (p *batchingEmitterImpl) emit() {
	if p.toDie() {
		return
	}
	if len(p.buff) == 0 {
		return
	}
	msgs2beEmitted := make([]interface{}, len(p.buff))
	for i, v := range p.buff {
		msgs2beEmitted[i] = v.data
	}

	p.cb(msgs2beEmitted)
	p.decrementCounters()
}
```

emit 中对消息的处理，主要是通过调用消息处理回调函数**cb**，处理结束后调用decrementCounters对每条消息的转发次数进行**减一**的操作。

## 总结

不难看出，分批发射器是一个通用的组件。具体的处理逻辑由用户提供，在分批发射器内部仅仅关注如何将数据批量触发的问题，即定时触发与消息计数达到转发的阈值。


分批发射器主要用于Gossip的推送或转发阶段。例如:发现层中定时发送Alive消息时对Emitter的应用。调用流程如下：

```
gossipDiscoveryImpl
    periodicalSendAlive
        d.comm.Gossip(msg)
            da.gossipFunc(msg)
                g.emitter.Add(...)
                    sendGossipBatch // 消息处理回调函数cb
```