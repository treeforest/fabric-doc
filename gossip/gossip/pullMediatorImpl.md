# Gossip-pullMediatorImpl

pullMediatorImpl 作为一个中介者模式，其封装了一系列PullEngine与PullAdapter的操作，并实现Mediator接口供外部调用。其使得各对象之间不需要显式的相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。

本文介绍流程：
- 首先介绍 pullMediatorImpl 实现的 Mediator 接口，了解其提供给外部的功能。
- 其次介绍 pullMediatorImpl 的配置 Config，因为一些重要的配置变量将影响如何理解处理流程。
- 最后再讲解 pullMediatorImpl 的具体实现。包括三方面：介绍其基本成员变量；初始化 pullMediatorImpl 的 NewPullMediator 函数；处理消息的 HandleMessage 函数。

## Mediator 接口

```
type Mediator interface {
	// Stop 停止Mediator运行
	Stop()

	// RegisterMsgHook 将消息钩子注册到特定类型的pull消息
	RegisterMsgHook(MsgType, MessageHook)

	// Add 添加一个Gossip消息(GossipMessage)到Mediator
	Add(*protoext.SignedGossipMessage)

	// Remove 根据消息摘要(digest)从Mediator中移除消息(GossipMessage)
	Remove(digest string)

	// HandleMessage 处理来自某个远程peer的消息
	HandleMessage(msg protoext.ReceivedMessage)
}
```

Mediator是包装PullEngine的组件，提供执行拉取同步所需的方法。将拉取中介特殊化为某种类型的消息是由配置、IdentifierExtractor、构造时给出的IdentifierExtractor以及可以为每种类型的pullMsgType (hello、digest、req、res)注册的钩子完成的。

接口的实现对象即为 pullMediatorImpl。

## Config

```
type Config struct {
	ID                string                  // 配置ID
	PullInterval      time.Duration           // 拉取引擎pull间隔时间
	Channel           common.ChannelID        // 链ID，在GossipMessage中填写
	PeerCountToSelect int                     // 拉取引擎每次pull时选取的节点数
	Tag               proto.GossipMessage_Tag // 消息标签，在GossipMessage中填写
	MsgType           proto.PullMsgType       // 拉取的消息类型
	PullEngineConfig  algo.PullEngineConfig   // 拉取引擎配置
}
```

Config 是pullMediatorImpl的配置定义。

**MsgType** 定义了拉取引擎处理的消息类型。在Hello、SendDigest、SendReq、SendRes中使用MsgType定义发送的消息类型，在HandleMessage中根据MsgType过滤要处理的消息。可以说，MsgType的类型定义了拉取引擎的处理对象。

## pullMediatorImpl

```
type pullMediatorImpl struct {
	sync.RWMutex
	*PullAdapter                                         // 拉取适配器
	msgType2Hook map[MsgType][]MessageHook                // <消息类型，对该消息类型所注册的钩子函数>
	config       Config                                   // 配置文件
	logger       util.Logger                              // 日志打印
	itemID2Msg   map[string]*protoext.SignedGossipMessage // <消息摘要，具体消息>
	engine       *algo.PullEngine                         // 拉取引擎
}
```

注意：这里的PullAdaptor与PullEngine里面的PullAdaptor并不是同一个类型。PullEngine里面的PullAdaptor是一个接口类型，而这里的PullAdaptor是一个结构体类型。

**Mediator 中的 PullAdaptor**

```
// PullAdapter 定义pullStore与gossip的各个模块交互的方法
type PullAdapter struct {
	Sndr             Sender              // 向远程peer节点发送消息
	MemSvc           MembershipService   // 获取alive节点的成员信息
	IdExtractor      IdentifierExtractor // 从SignedGossipMessage中提取一个标识符(identifier)
	MsgCons          MsgConsumer         // 消费者，处理SignedGossipMessage消息
	EgressDigFilter  EgressDigestFilter  // 发出消息过滤器
	IngressDigFilter IngressDigestFilter // 到来消息过滤器
}
```

**PullEngine 中的 PullAdaptor**

```
type PullAdapter interface {
	// 返回peer节点的切片，PullEngine将使用它来初始化协议
	SelectPeers() []string

	// 发送一个Hello消息来初始化协议，并返回一个被期望在摘要信息中返回的 NONCE 值
	Hello(dest string, nonce uint64)

	// 发送一个摘要给远程PullEngine。context 参数指定要发送到的远程引擎。
	SendDigest(digest []string, nonce uint64, context interface{})

	// 发送 items 到一个由 dest 指定地址的远程 PullEngine
	SendReq(dest string, items []string, nonce uint64)

	// 发送 items 到一个由 context 指定地址的远程 PullEngine
	SendRes(items []string, context interface{}, nonce uint64)
}
```

## 主要实现

注意：pullMediatorImpl同时实现了Mediator与PullAdapter接口，由于大部分接口实现均较为容易，这里仅讲解HandleMessage。

### NewPullMediator

```
func NewPullMediator(config Config, adapter *PullAdapter) Mediator {
	egressDigFilter := adapter.EgressDigFilter

	acceptAllFilter := func(_ protoext.ReceivedMessage) func(string) bool {
		return func(_ string) bool {
			return true
		}
	}

	if egressDigFilter == nil {
		egressDigFilter = acceptAllFilter
	}

	p := &pullMediatorImpl{
		PullAdapter:  adapter,
		msgType2Hook: make(map[MsgType][]MessageHook),
		config:       config,
		logger:       util.GetLogger(util.PullLogger, config.ID),
		itemID2Msg:   make(map[string]*protoext.SignedGossipMessage),
	}

	// 初始化拉取引擎
	p.engine = algo.NewPullEngineWithFilter(p, config.PullInterval, egressDigFilter.byContext(), config.PullEngineConfig)

	if adapter.IngressDigFilter == nil {
		// Create accept all filter
		adapter.IngressDigFilter = func(digestMsg *proto.DataDigest) *proto.DataDigest {
			return digestMsg
		}
	}
	return p

}
```

通过 NewPullMediator 函数创建一个pullMediatorImpl实例并返回一个Mediator接口。在初始化拉取引擎后，拉取引擎就开始了工作（具体实现可以参考PullEngine说明）。

初始化 pullMediatorImpl 时，用户根据config与adapter参数定制拉取引擎，定制包括指定消息的发送方式与拉取的消息类型等。

### HandleMessage

```
func (p *pullMediatorImpl) HandleMessage(m protoext.ReceivedMessage) {
	if m.GetGossipMessage() == nil || !protoext.IsPullMsg(m.GetGossipMessage().GossipMessage) {
		return
	}

	// 1、若不是配置中指定的消息类型，则直接返回
	msg := m.GetGossipMessage()
	msgType := protoext.GetPullMsgType(msg.GossipMessage)
	if msgType != p.config.MsgType {
		return
	}

	p.logger.Debug(msg)

	itemIDs := []string{}
	items := []*protoext.SignedGossipMessage{}
	var pullMsgType MsgType

	// 2、判断消息类型，并调用相应的处理
	if helloMsg := msg.GetHello(); helloMsg != nil { // Hello Message
		pullMsgType = HelloMsgType
		p.engine.OnHello(helloMsg.Nonce, m)
	} else if digest := msg.GetDataDig(); digest != nil { // Digest Message
		d := p.PullAdapter.IngressDigFilter(digest)
		itemIDs = util.BytesToStrings(d.Digests)
		pullMsgType = DigestMsgType
		p.engine.OnDigest(itemIDs, d.Nonce, m)
	} else if req := msg.GetDataReq(); req != nil { // Request Message
		itemIDs = util.BytesToStrings(req.Digests)
		pullMsgType = RequestMsgType
		p.engine.OnReq(itemIDs, req.Nonce, m)
	} else if res := msg.GetDataUpdate(); res != nil { // Response Message
		itemIDs = make([]string, len(res.Data))
		items = make([]*protoext.SignedGossipMessage, len(res.Data))
		pullMsgType = ResponseMsgType
		for i, pulledMsg := range res.Data {
			msg, err := protoext.EnvelopeToGossipMessage(pulledMsg)
			if err != nil {
				p.logger.Warningf("Data update contains an invalid message: %+v", errors.WithStack(err))
				return
			}
			p.MsgCons(msg)
			itemIDs[i] = p.IdExtractor(msg)
			items[i] = msg
			p.Lock()
			p.itemID2Msg[itemIDs[i]] = msg
			p.logger.Debugf("Added %s to the in memory item map, total items: %d", itemIDs[i], len(p.itemID2Msg))
			p.Unlock()
		}
		p.engine.OnRes(itemIDs, res.Nonce)
	}

	// 3、调用相关消息类型的钩子
	for _, h := range p.hooksByMsgType(pullMsgType) {
		h(itemIDs, items, m)
	}
}
```

数据的处理流程：

- 当接收的消息 m 到来时，需要判断 m 的类型是否为要处理的消息类型。若是，则进一步根据类型(Hello/Digest/Request/Response)分发处理。拉取引擎处理完成后，再调用对应消息类型的钩子函数对消息拦截处理。

## 总结

拉取中介整体实现逻辑较为简单，因为具体的实现逻辑均由PullEngine、PullAdapter等实现。

**总结要点**
- 用户调用 HandleMessage 处理接收的Pull消息。
- 用户可以调用RegisterMsgHook注册钩子，拦截处理指定的消息。
- pullMediatorImpl 目前仅仅被gossipChannel与certStore用于同步block与同步节点之间的证书。
- pullMediatorImpl 保存了摘要以及摘要所对应的具体消息，而PullEngine仅保存摘要信息。
