# Gossip-idMapper

Gossip 协议的网络成员使用identity进行标识。idMapper就是用来保存成员之间的identity，管理组织的成员。保存identity是一个临时性行为，identity被赋予了时间值（也就是证书的有效期cert.NotAfter），identity在Gossip网络传播过程中进行有效期检查，若过期会自动从列表中删除。

## Mapper 接口实现

```
## [gossip/identity/identity.go]
type Mapper interface {
	// Put 把pkiID与identity关联，若pkiID不匹配identity，则返回错误
	Put(pkiID common.PKIidType, identity api.PeerIdentityType) error

	// Get 返回给定pkiID的identity，如果identity不存在，则返回错误
	Get(pkiID common.PKIidType) (api.PeerIdentityType, error)

	// Sign 对消息进行签名，成功时返回已签名的消息，失败时返回错误
	Sign(msg []byte) ([]byte, error)

	// Verify 验证已签名的消息
	Verify(vkID, signature, message []byte) error

	// GetPKIidOfCert 返回证书的pkiID
	GetPKIidOfCert(api.PeerIdentityType) common.PKIidType

	// SuspectPeers 重新验证与给定的谓词匹配的所有peers
	SuspectPeers(isSuspected api.PeerSuspector)

	// IdentityInfo 返回已知的peer身份信息集
	IdentityInfo() api.PeerIdentitySet

	// Stop 停止映射器
	Stop()
}
```

## 实现Mapper接口的结构

**identityMapperImpl**
```
## [gossip/identity/identity.go]
type identityMapperImpl struct {
	onPurge    purgeTrigger               // 清除触发器
	mcs        api.MessageCryptoService   // 消息加密服务接口
	sa         api.SecurityAdvisor        // 安全顾问接口
	pkiID2Cert map[string]*storedIdentity // pkiID和identity映射集合
	sync.RWMutex
	stopChan  chan struct{}
	once      sync.Once
	selfPKIID string // 自身pkiID
}

```

映射器的关键实现**pkiID2Cert**是一个map集合，key为pkiID，value为storedIdentity结构。

**storedIdentity**
```
## [gossip/identity/identity.go]
type storedIdentity struct {
	pkiID           common.PKIidType     // pkiID
	lastAccessTime  int64                // 最近访问identity的时间
	peerIdentity    api.PeerIdentityType // 节点的标识identity
	orgId           api.OrgIdentityType  // 组织的标识
	expirationTimer *time.Timer          // 过期定时器
}
```

**Put(pkiID common.PKIidType, identity api.PeerIdentityType) error**

要理解映射器，重点关注identityMapperImpl结构实现的Put函数即可。传入一个身份identity，需要调用mcs获取身份有效期、校验身份合法性及获取身份对应的id，判断获取的id与传入的参数pkiID是否一致，若不一致，则报错推出。再判断身份是否已经存在于pkiID2Cert映射列表中，若已存在，则直接退出；否则根据身份有效期与当前时间判断身份是否已过期，若未过期，则设置一个自动执行的时间函数，身份到期后直接将其从pkiID2Cert映射列表中删除。最后将当前身份保存到pkiID2Cert中。

```
## [gossip/identity/identity.go]
func (is *identityMapperImpl) Put(pkiID common.PKIidType, identity api.PeerIdentityType) error {
	if pkiID == nil {
		return errors.New("PKIID is nil")
	}
	if identity == nil {
		return errors.New("identity is nil")
	}

	// 使用mcs获取identity过期日期
	expirationDate, err := is.mcs.Expiration(identity)
	if err != nil {
		return errors.Wrap(err, "failed classifying identity")
	}

	// 使用mcs验证identity
	if err := is.mcs.ValidateIdentity(identity); err != nil {
		return err
	}

	// 获取identity对应的id
	id := is.mcs.GetPKIidOfCert(identity)
	if !bytes.Equal(pkiID, id) {
		return errors.New("identity doesn't match the computed pkiID")
	}

	is.Lock()
	defer is.Unlock()
	// 检查pkiID是否已经存在
	if _, exists := is.pkiID2Cert[string(pkiID)]; exists {
		return nil
	}

	var expirationTimer *time.Timer
	if !expirationDate.IsZero() {
		// 是否过期
		if time.Now().After(expirationDate) {
			return errors.New("identity expired")
		}
		// identity将在其过期日期后一毫秒被清除
		timeToLive := time.Until(expirationDate.Add(time.Millisecond))
		expirationTimer = time.AfterFunc(timeToLive, func() {
			is.delete(pkiID, identity)
		})
	}

	// 保存到集合中
	is.pkiID2Cert[string(id)] = newStoredIdentity(pkiID, identity, expirationTimer, is.sa.OrgByPeerIdentity(identity))
	return nil
}
```

## pkiID 与 identity 来源

Put函数将关键的pkiID与identity保存到idMapper，接下来就来看下pkiID与identity的来源。

**identity**

```
## [msp/identities.go]
func newIdentity(cert *x509.Certificate, pk bccsp.Key, msp *bccspmsp) (Identity, error) {
	if mspIdentityLogger.IsEnabledFor(zapcore.DebugLevel) {
		mspIdentityLogger.Debugf("Creating identity instance for cert %s", certToPEM(cert))
	}
	
	// 1、首先对证书消毒=>确保使用ECDSA签署的x509证书确实有Low-S格式的签名
	cert, err := msp.sanitizeCert(cert)
	if err != nil {
		return nil, err
	}

	// Compute identity identifier

	// Use the hash of the identity's certificate as id in the IdentityIdentifier
	// 2、使用身份证书的散列作为IdentityIdentifier中的id
	hashOpt, err := bccsp.GetHashOpt(msp.cryptoConfig.IdentityIdentifierHashFunction) // 获取与所传递的散列函数对应的hashOpt
	if err != nil {
		return nil, errors.WithMessage(err, "failed getting hash function options")
	}

	digest, err := msp.bccsp.Hash(cert.Raw, hashOpt) // 使用hashOpt对cert.Raw进行哈希计算，获取摘要信息(散列)
	if err != nil {
		return nil, errors.WithMessage(err, "failed hashing raw certificate to compute the id of the IdentityIdentifier")
	}

    // 
	id := &IdentityIdentifier{
		Mspid: msp.name,
		Id:    hex.EncodeToString(digest)}

	return &identity{id: id, cert: cert, pk: pk, msp: msp}, nil
}
```

newIdentity 需要传入x509格式的证书、公钥及msp实例。首先msp会将证书文件cert进行消毒处理，若证书没有Low-S格式的签名，则将证书转换成具有Low-S格式签名的证书。然后获取对证书文件进行哈希操作的哈希操作对象hashOpt，使用hashOpt对证书的序列化数据进行哈希计算，获取证书的摘要信息(散列值)，若获取失败，则返回错误。最后使用id、cert、pk、msp初始化一个identity对象返回。

```
## [msp/identities.go]
func newSigningIdentity(cert *x509.Certificate, pk bccsp.Key, signer crypto.Signer, msp *bccspmsp) (SigningIdentity, error) {
	//mspIdentityLogger.Infof("Creating signing identity instance for ID %s", id)
	mspId, err := newIdentity(cert, pk, msp)
	if err != nil {
		return nil, err
	}
	return &signingidentity{
		identity: identity{
			id:   mspId.(*identity).id,
			cert: mspId.(*identity).cert,
			msp:  mspId.(*identity).msp,
			pk:   mspId.(*identity).pk,
		},
		signer: signer,
	}, nil
}
```

newSigningIdentity 中的主要实现就是调用newIdentity，返回值为一个SigningIdentity和一个error值。

```
## [msp/identities.go]
func (id *identity) Serialize() ([]byte, error) {
	pb := &pem.Block{Bytes: id.cert.Raw, Type: "CERTIFICATE"}
	pemBytes := pem.EncodeToMemory(pb)
	if pemBytes == nil {
		return nil, errors.New("encoding of identity failed")
	}

	// 我们通过将MSPID作为前缀并附加cert的ASN.1 DER内容来序列化身份
	sId := &msp.SerializedIdentity{Mspid: id.id.Mspid, IdBytes: pemBytes}
	idBytes, err := proto.Marshal(sId)
	if err != nil {
		return nil, errors.Wrapf(err, "could not marshal a SerializedIdentity structure for identity %s", id.id)
	}

	return idBytes, nil
}
```

Serialize 返回一个代表identity的字节数组。即idBytes是将Mspid与证书编码到内存后的pemBytes值，组装到SerializedIdentity结构中，然后经过protobuf的序列化后所得到。

- 不难看出，若peer接收到来自远程peer的identity标识，可以反序列话得到SerializedIdentity结构，可以从中得到其相关信息。

注意：idMapper 中保存的identity值是一个字节切片，并不是上面所说的identity结构。idMapper中的identity就是Serialize函数返回的切片值。

总结：identity是由Mspid和证书cert组合而成。

**pkiID**

```
## [internal/peer/gossip/mcs.go]
func (s *MSPMessageCryptoService) GetPKIidOfCert(peerIdentity api.PeerIdentityType) common.PKIidType {
	// 检查参数
	if len(peerIdentity) == 0 {
		mcsLogger.Error("Invalid Peer Identity. It must be different from nil.")

		return nil
	}

	// 对peerIdentity进行反序列化
	sid, err := s.deserializer.Deserialize(peerIdentity)
	if err != nil {
		mcsLogger.Errorf("Failed getting validated identity from peer identity %s: [%s]", peerIdentity, err)

		return nil
	}

	// concatenate msp-id and idbytes
	// idbytes is the low-level representation of an identity.
	// it is supposed to be already in its minimal representation
	// idbytes 是 identity 的 low-level 表示
	mspIDRaw := []byte(sid.Mspid)
	// 连接 mspid 和 sid.IdBytes
	raw := append(mspIDRaw, sid.IdBytes...)

	// 对raw进行哈希计算得到摘要
	digest, err := s.hasher.Hash(raw, &bccsp.SHA256Opts{})
	if err != nil {
		mcsLogger.Errorf("Failed computing digest of serialized identity %s: [%s]", peerIdentity, err)
		return nil
	}

	return digest
}
```

pkiID是根据peer的identity计算获得的，使用的是SHA256哈希算法。

## 总结

- 映射器主要保存的是pkiID到identity的映射关系，相当于一个增强型的map结构
- identity 是会被定期清除的（如果长时间没有响应）