# 以太坊节点是如何处理交易的-上篇

今天在这里从代码的层面和朋友们分享下以太坊的交易处理流程,如果能帮助新手朋友快速上手,那是我的荣幸,嘿嘿, 直接进入主题吧.
以太坊对交易的处理过程比较复杂,大致分成以下几步执行:

- **调用RPC接口,将交易发送到tx_pool的pending队列中.**
- **将pending队列中的txs 在网络中广播出去,让矿机节点收到.**
- **矿机节点创建区块前会从tx pool的pending队列中取出交易并执行;然后将交易,以及交易执行后的receipts等信息封装到区块内,出块,并广播到网络中.**
- **网络中的其他以太坊节点收到区块后,对区块内的交易进行验证,确认无误后,将区块插入到本地的链上.如果校验失败,则区块不能被接受,tx信息同样也不能在网络中同步.**


当用户通过命令行,或者其他方式发送一笔转账交易的时候,node节点会收到RPC请求,经过节点的RPC Server解析处理后,会发送到SendTransaction函数.下面是这个函数的处理过程
```
go-etherueum/internal/ethapi/api.go
func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args SendTxArgs) (common.Hash, error) {

    //从参数args 中获得发送交易的账户信息,发送交易要用这个账户为交易签名.
	account := accounts.Account{Address: args.From}
    
//找到跟这个account相关的wallet,account可以动态的从钱包中添加或者删除;wallet也可以有很多个,有可能是usb wallet,有可能是mobile wallet,还可能是keystore节点上的wallet.具体的查找经过交给下面的接口函数就好了
	wallet, err := s.b.AccountManager().Find(account)
	if err != nil {
		return common.Hash{}, err
	}

	if args.Nonce == nil {
		// Hold the addresse's mutex around signing to prevent concurrent assignment of
		// the same nonce to multiple accounts.
		s.nonceLock.LockAddr(args.From)
		defer s.nonceLock.UnlockAddr(args.From)
	}

	//设置一笔交易需要的默认参数,比如获取nonce, 预计消耗的gas, gasprice等等.
	if err := args.setDefaults(ctx, s.b); err != nil {
		return common.Hash{}, err
	}
	//根据传入的参数,组装出一笔交易,如果是合约types.NewContractCreation,否则调用types.NewTransaction
	tx := args.toTransaction()

	var chainID *big.Int
	if config := s.b.ChainConfig(); config.IsEIP155(s.b.CurrentBlock().Number()) {
		chainID = config.ChainID
	}
    //调用wallet的签名函数,给交易签名;只有签名的交易才可以被发送成功.
	signed, err := wallet.SignTx(account, tx, chainID)
	if err != nil {
		return common.Hash{}, err
	}
    //最后执行submit函数,将交易发送到交易池中.
	return submitTransaction(ctx, s.b, signed)
}
```
对转发普通交易(本文不涉及合约交易)来说submitTransaction函数的的核心就是执行SendTx
```
func submitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
	if err := b.SendTx(ctx, tx); err != nil {
		return common.Hash{}, err
	}
    ...
    ...
    ...
	return tx.Hash(), nil
}

func (b *EthAPIBackend) SendTx(ctx context.Context, signedTx *types.Transaction) error {
    //调用AddLocal将这笔签名交易添加到本地的txpool的pending队列中. 接口是简单的,背后实现的过程是复杂的,^_^.
	return b.eth.txPool.AddLocal(signedTx)
}
```
**下面开始一步步剖析一笔tx是如何进入tx_pool的pending队列中的,这段过程稍微漫长.**
咱们首先介绍下tx pool在以太坊中的作用
txpool主要用来存放当前提交的等待写入区块的交易，有远端和本地的。
txpool里面的交易分为两种，
1. 提交但是还不能执行的，放在queue里面等待能够执行(比如说nonce太高)。
2. 等待执行的，放在pending里面等待执行。

从txpool的测试案例来看，txpool主要功能有下面几点。
1. 交易验证的功能，包括余额不足，Gas不足，Nonce太低, value值是合法的，不能为负数。
2. 能够缓存Nonce比当前本地账号状态高的交易。 存放在queue字段。 如果是能够执行的交易存放在pending字段
3. 相同用户的相同Nonce的交易只会保留一个GasPrice最大的那个。 其他的插入不成功。
4. 如果账号没有钱了，那么queue和pending中对应账号的交易会被删除。
5. 如果账号的余额小于一些交易的额度，那么对应的交易会被删除，同时有效的交易会从pending移动到queue里面。防止被广播。
6. txPool支持一些限制PriceLimit(remove的最低GasPrice限制)，PriceBump(替换相同Nonce的交易的价格的百分比) AccountSlots(每个账户的pending的槽位的最小值) GlobalSlots(全局pending队列的最大值)AccountQueue(每个账户的queueing的槽位的最大值) GlobalQueue(全局queueing的最大值) Lifetime(在queue队列的最长等待时间)
7. 有限的资源情况下按照GasPrice的优先级进行替换。
8. 本地的交易会使用journal的功能存放在磁盘上，重启之后会重新导入。 远程的交易不会。
[摘自 go-ethereum-code-analysis](https://github.com/ZtesoftCS/go-ethereum-code-analysis/blob/master/core-txpool%E4%BA%A4%E6%98%93%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

从代码中可以看到,AddLocal很简单,就执行了一行addTx.
```
func (pool *TxPool) AddLocal(tx *types.Transaction) error {
	return pool.addTx(tx, !pool.config.NoLocals)
}
```
addTx函数主要调用了add和promoteExecutables 两个方法.这两个方法是txpool的重点,值得仔细推敲一遍.
```
func (pool *TxPool) addTx(tx *types.Transaction, local bool) error {
	pool.mu.Lock()
	defer pool.mu.Unlock()

	//add方法会对tx进行初步的校验,如果没问题,将tx 加入到txpool的queue中.
	replace, err := pool.add(tx, local)
	if err != nil {
		return err
	}
	//通过执行promoteExecutables函数,一笔交易才会加入到pending队列,同时会通知网络协程pending队列更新,网络协程会将新的pengding队列广播到整个网络中.
	if !replace {
		from, _ := types.Sender(pool.signer, tx) // already validated
		pool.promoteExecutables([]common.Address{from})
	}
	return nil
}
```
```
func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) {
	//先检查pool中是否存在交易的哈希,如果存在,则不需要再次加入了,直接报错
	hash := tx.Hash()
	if pool.all.Get(hash) != nil {
		log.Trace("Discarding already known transaction", "hash", hash)
		return false, fmt.Errorf("known transaction: %x", hash)
	}
	//调用validateTx方法对tx进行校验,校验内容包括:
     tx占用的存储空间大小,
     签名是否正确,
     tx的gas消耗是否合理,
     签名者是否正确,
     该笔交易的nonce值是否合理,
     发送交易账户的金额是否充足等几个方面.
	if err := pool.validateTx(tx, local); err != nil {
		log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
		invalidTxCounter.Inc(1)
		return false, err
	}
	//如果本地的交易队列满了,那么从2方面处理这笔tx
	if uint64(pool.all.Count()) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
		//如果新的tx的消耗的gas小于本地队列中gas最小的那笔交易,那么这笔新tx就会被丢弃
		if !local && pool.priced.Underpriced(tx, pool.locals) {
			log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
			underpricedTxCounter.Inc(1)
			return false, ErrUnderpriced
		}
		//新的tx消耗的gas比本地队列中最少的多,那么删掉本地消耗gas最少的那些交易.如果all.Count()正好等于int(pool.config.GlobalSlots+pool.config.GlobalQueue) 那么Discard函数的参数count最小值是1;count值应该恒>=1
		drop := pool.priced.Discard(pool.all.Count()-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
		for _, tx := range drop {
			log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
			underpricedTxCounter.Inc(1)
			pool.removeTx(tx.Hash(), false)
		}
	}
	//先获得tx的账户地址,根据账户地址获得相关的pending list. Overlaps函数返回一个布尔值,它会判断新tx的nonce值是否在pending队列中存在,如果存在返回true.
	from, _ := types.Sender(pool.signer, tx) // already validated
	if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
		//新tx的nonce值在pending已经存在了,list.Add会根据PriceBump计算新tx的gas是否超过旧tx,如果新tx的gas超过旧tx的gas量的百分之PriceBump,就用新的tx替换掉旧的tx.这样设计是为了保证矿工的利益最大化.
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted {
			pendingDiscardCounter.Inc(1)
			return false, ErrReplaceUnderpriced
		}
		//删除掉旧的tx.
		if old != nil {
			pool.all.Remove(old.Hash())
			pool.priced.Removed()
			pendingReplaceCounter.Inc(1)
		}
		pool.all.Add(tx) //将新tx添加到all队列中
		pool.priced.Put(tx) //基于tx的price大小,添加到priced队列中,这个队列是顺序排列的
		pool.journalTx(from, tx)//将新的tx写入到本地的日志文件中.

		log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())

		//通知网络系统,队列中的交易更新了
		go pool.txFeed.Send(NewTxsEvent{types.Transactions{tx}})
        将拥有相同nonce值的旧tx替换掉,就可以返回了.
		return old != nil, nil
	}
	// 如果新的tx没有替换掉旧的tx,那么通过执行enqueueTx将tx先存放到txpool的queue队列中,
	replace, err := pool.enqueueTx(hash, tx)
	if err != nil {
		return false, err
	}
	// Mark local addresses and journal local transactions
	if local {
		pool.locals.add(from)
	}
    //将tx添加到本地的日志文件中
	pool.journalTx(from, tx)

	log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
	return replace, nil
}
```

经过add函数将tx添加到queue队列后,执行promoteExecutables函数,将queue队列中的tx转移到pending队列中.这个函数很长,在进入分析之前,先简单说下它的工作流程,以及相关的数据结构,这样比较便于快速理解.
这里面主要的数据结构有以下三种:
```
//这里声明了一个uint64的切片,用这个切片模拟了一个堆,在tx_list.go文件中,还实现了简单的pop,push操作方法.这个数据结构用来存储每笔交易的nonce值.每次产生一笔交易,交易的nonce值就存储到这个切片中.
type nonceHeap []uint64

//txSortedMap这是个核心数据结构,txList的各种操作逻辑也是封装的txSortedMap,txSortedMap这个结构实现了各种操作nonce堆栈的方法供txList调用,下面会对这些方法进行简单介绍.
type txSortedMap struct {
    items map[uint64]*types.Transaction //这个map结构的key 是交易的nonce值,value是该笔交易的交易数据
	index *nonceHeap //items中的key也就是nonce都会存储到index数组切片中,按照heap的pop,push方式管理.
	cache types.Transactions //这是个Transactions的数组切片,这里暂存已经排好序的交易数据;tx_list.go中调用了golang源码包的快速排序算法
}

//txList 是基于account的关系维护的,在type TxPool 结构中有两个域pending 和queue,这两个结构都是字典结构map[common.Address]*txList;每个以太节点上txpool的交易队列中的交易都是基于account的txList维护的,这样的设计很容易统计管理.
type txList struct {
	strict bool //一个标识符,true表示存储的nonce都是连续,false是不连续的.
	txs    *txSortedMap //交易队列的核心结构,采用堆栈式管理.
	costcap *big.Int //交易花费的最高价格（仅在超出余额时重置）
	gascap  uint64 //交易消耗的最高gas limit（仅在超过限额时重置）
}

个人觉得tx_list.go中代码设计的很精美,他们将数组采用堆栈化的操作,对数组内容增删改查以及排序的手法值得初学golang的朋友们吸收学习下.
```
下面简单介绍下操作txSortedMap结构的方法,由于篇幅限制代码细节就不详细描述了,感兴趣的童鞋可以自己看.

```
func (m *txSortedMap) Get(nonce uint64) *types.Transaction
Get方法根据传入的交易nonce值,从哈希map中获取到对应的Transaction数据.

func (m *txSortedMap) Put(tx *types.Transaction)
Put方法将一笔新的交易插入到map结构中,同时将tx的nonce值push到index堆栈中.

func (m *txSortedMap) Forward(threshold uint64) types.Transactions
Forward方法将nonce值低于阈值threshold的交易从队列中删除;Forward操作的index对象是已经排好顺序的,按照nonce值从小到大排列.

func (m *txSortedMap) Filter(filter func(*types.Transaction) bool) types.Transactions
Filter方法遍历整个交易map, 将符合过滤函数条件的交易从map中删除.

func (m *txSortedMap) Cap(threshold int) types.Transactions
Cap方法设置一个阈值nonce,先将整个index数组切片用快速排序算法从小到大排序,然后将超过阈值nonce的交易从map中删除,并更新index数组切片.

func (m *txSortedMap) Remove(nonce uint64) bool
Remove根据nonce参数,从map中删除相关的tx

func (m *txSortedMap) Ready(start uint64) types.Transactions
Ready方法返回一个nonce值顺序连续增长的的tx队列.这个ready队列中的交易会被promoteTx处理,添加到txpool中的pending队列上,这些交易将来会被矿机封装到区块中.

func (m *txSortedMap) Flatten() types.Transactions
Flatten方法先将交易队列中的所有tx存放到cache结构中,然后将cache中的tx队列按照nonce值大小快速排序.

txList结构也有上面的那些方法,但都是基于txSortMap封装的.
promoteExecutables函数主要就是在用各种业务逻辑操作各个account的txList,也就是调用上面列出的Get,Put,Forward,Filter,Cap,Remove,Ready,Flatten等各种方法,啰嗦了半天就为了这里做铺垫.
```
promoteExecutables函数的工作逻辑如下:
先从pool.queue中获得每个accout的list

1. 清理相关accout的老旧tx
2. 清理消耗成本或者gaslimit过高的交易和无效的交易
3. 收集可执行的交易,存到txpool的pending队列中
4. 清理非本地accout上的queue队列,使其不超过默认配置.
5. 通知其他子系统,txpool pending队列已经更新.
6. 如果txpool的pending队列过长,调整非本地的account的pending队列,将过多txs删除,调整到代码配置许可的范围内. (因为是P2P网络,其他节点上account中的交易也会被广播到本地,这些交易也会存在本地txpool的queue中,但是每个节点要优先要保证本地account发送的交易先上链,所以在超出配置限制的时候,先拿外部account中的交易开刀,这就是其中蕴含的设计思想)
7. 如果txpool的queue队列过长,根据accout的活跃时间排序,会清理最老的account的queue队列,直到达到系统设置的阈值.

下面是具体的执行流程:
```
func (pool *TxPool) promoteExecutables(accounts []common.Address) {
	//跟踪promoted数组切片的长度,一旦更新会立即在网络中广播出去.
	var promoted []*types.Transaction

	// 如果accounts 参数为nil ,那么收集pool.queue中所有的account信息.
	if accounts == nil {
		accounts = make([]common.Address, 0, len(pool.queue))
		for addr := range pool.queue {
			accounts = append(accounts, addr)
		}
	}
	//遍历accounts中所有的可执行交易,将这些txs 追加到promoted 数组切片中.
	for _, addr := range accounts {
		list := pool.queue[addr] //从pool.queue中获得每个account的txList.
		if list == nil {
			continue // Just in case someone calls with a non existing account
		}
		//先获取当前account的nonce值,将nonce值低于当前nonce的tx,也就是已经失效的tx全部从队列中删除掉
        list.Forward返回的是一个切片,符合删除条件的tx都存在这个切片中.
		for _, tx := range list.Forward(pool.currentState.GetNonce(addr)) {
			hash := tx.Hash()
			log.Trace("Removed old queued transaction", "hash", hash)
			pool.all.Remove(hash)
			pool.priced.Removed() //从priced的heap上删除,每笔tx也存到priced的heap
		}
		//清除掉花费太高的tx
		drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
		for _, tx := range drops {
			hash := tx.Hash()
			log.Trace("Removed unpayable queued transaction", "hash", hash)
			pool.all.Remove(hash)
			pool.priced.Removed()
			queuedNofundsCounter.Inc(1)
		}
		//收集所有可执行的交易,存到promoted数组切片中
		for _, tx := range list.Ready(pool.pendingState.GetNonce(addr)) {
			hash := tx.Hash()
			if pool.promoteTx(addr, hash, tx) {//给pending队列添加新的txs.
				log.Trace("Promoting queued transaction", "hash", hash)
				promoted = append(promoted, tx)
			}
		}
		//AccountQueue是txpool允许的每个account最大可以维护的queue队列长度,如果非本地的account的 tx queue长度超过限制,那么清除掉超过限制的那部分txs.
		if !pool.locals.contains(addr) {
			for _, tx := range list.Cap(int(pool.config.AccountQueue)) {
				hash := tx.Hash()
				pool.all.Remove(hash)
				pool.priced.Removed()
				queuedRateLimitCounter.Inc(1)
				log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
			}
		}
		//如果txList是空的,就从queue队列中清除相关account的所有信息.
		if list.Empty() {
			delete(pool.queue, addr)
		}
	}
	//这里会通知其他子模块,新的tx已经准备好了;比如网络模块收到这个消息后,会在
    P2P网络中广播promoted中的交易.
	if len(promoted) > 0 {
		go pool.txFeed.Send(NewTxsEvent{promoted})
	}
	// 如果txpool的pending超过限制了.那么调整pending的长度,使其回到允许的范围.
	pending := uint64(0)
    //统计txpool中所有account的list之和.
	for _, list := range pool.pending {
		pending += uint64(list.Len())
	}
    //GlobalSlots是txpool允许的所有account可执行的tx之和,以太坊默认的值是4096
    如果各个账户的pending长度之和超过4096,就执行下面的if分支
	if pending > pool.config.GlobalSlots {
		pendingBeforeCap := pending
		//这里创建一个堆栈,专门用来存储大交易量的外来account,嗯哼,就是喜欢欺负外来户.
		spammers := prque.New()
		for addr, list := range pool.pending {
//AccountSlots是txpool允许的每个账户最小可执行的tx数量,这个参数是为非本  地节点上的account准备的, local的account莫担心,不会受到它的限制.这个值默认是16,也就是说在txpool的pending超过GlobalSlots的时候,txpool要清理外来户,外来户交易量超过16这个值,要被制裁的,交税也不行,哎,北漂真苦啊!(感慨下人生,O(∩_∩)O~)
			if !pool.locals.contains(addr) && uint64(list.Len()) > pool.config.AccountSlots {
               //将所有超过限制的非本地的account压栈.给这些account打上了一个logo:offender (中文翻译:坏人)
               这个压栈其实是存在排序的,不是按照时间顺序排的,是按照list的长度排列的.list最长的排在栈尾
				spammers.Push(addr, float32(list.Len()))
			}
		}
		//下面开始动手了~~
		offenders := []common.Address{}
		for pending > pool.config.GlobalSlots && !spammers.Empty() {
            //第一次循环,会把list最长的,也就是交易量最大的accout从栈上弹出来
			offender, _ := spammers.Pop()
            //把offender填到offenders切片中,做个新的黑名单.
        存储的顺序是酱紫的:offenders[0].Len()>offenders[1].Len()>offender[2].Len()...    
			offenders = append(offenders, offender.(common.Address))

			//如果黑名单的长度达到2以上,开始削减相关account的tx.
			if len(offenders) > 1 {
				//以当前offenders数组中的最后一个account的txList长度当做阈值,这个值是目前数组中最小的值
				threshold := pool.pending[offender.(common.Address)].Len()

				//开始迭代删除offender account的交易数量,直到达到limit或者达到当前的阈值.举个栗子吧: offenders数组有2个account的时候,offenders[0].Len()>offenders[1].Len() 以offenders[1]的.Len()做阈值,删掉offender[0]中超过阈值的txs,直到offender[0].Len()==offenders[1].Len();当offenders数组有3个accounts的时候,以offenders[2].Len()最小,以它做阈值,让offender[0].Len()==offender[1].Len()==offender[2].Len(),以此类推.

				for pending > pool.config.GlobalSlots && pool.pending[offenders[len(offenders)-2]].Len() > threshold {
					for i := 0; i < len(offenders)-1; i++ {
						list := pool.pending[offenders[i]]
						for _, tx := range list.Cap(list.Len() - 1) {
							// Drop the transaction from the global pools too
							hash := tx.Hash()
							pool.all.Remove(hash)
							pool.priced.Removed()

							// Update the account nonce to the dropped transaction
							if nonce := tx.Nonce(); pool.pendingState.GetNonce(offenders[i]) > nonce {
								pool.pendingState.SetNonce(offenders[i], nonce)
							}
							log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
						}
						pending--
					}
				}
			}
		}
		//迭代删除之后,如果pending总数还是超过limit,继续减少非本地account的txs
		if pending > pool.config.GlobalSlots && len(offenders) > 0 {
			for pending > pool.config.GlobalSlots && uint64(pool.pending[offenders[len(offenders)-1]].Len()) > pool.config.AccountSlots {
//遍历所有的黑名单account,每个account依次每次删除一条tx,直到满足限制条件.
                for _, addr := range offenders {
					list := pool.pending[addr]
					for _, tx := range list.Cap(list.Len() - 1) {
						// Drop the transaction from the global pools too
						hash := tx.Hash()
						pool.all.Remove(hash)
						pool.priced.Removed()

						// Update the account nonce to the dropped transaction
						if nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) > nonce {
							pool.pendingState.SetNonce(addr, nonce)
						}
						log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
					}
					pending--
				}
			}
		}
		pendingRateLimitCounter.Inc(int64(pendingBeforeCap - pending))
	}
	//这里开始统计txpool的queue总量.
	queued := uint64(0)
	for _, list := range pool.queue {
		queued += uint64(list.Len())
	}
    //如果queue超过限制,又一次开始准备削减非本地的account!
	if queued > pool.config.GlobalQueue {
		// Sort all accounts with queued transactions by heartbeat
		addresses := make(addresssByHeartbeat, 0, len(pool.queue))
		for addr := range pool.queue {
			if !pool.locals.contains(addr) { // don't drop locals
            //遍历queue中的所有account,为非本地的account们建立个黑名单.
				addresses = append(addresses, addressByHeartbeat{addr, pool.beats[addr]})
			}
		}
        //这里采用了golang系统包的快速排序算法,以account的活跃时间排序的.addresses
切片数组最末尾的成员是最久没有活跃过的account.
		sort.Sort(addresses)

		//先从最不活跃的account删减,直到符合limit.
		for drop := queued - pool.config.GlobalQueue; drop > 0 && len(addresses) > 0; {
			addr := addresses[len(addresses)-1]
			list := pool.queue[addr.address]

			addresses = addresses[:len(addresses)-1]

			// Drop all transactions if they are less than the overflow
			if size := uint64(list.Len()); size <= drop {
				for _, tx := range list.Flatten() {
					pool.removeTx(tx.Hash(), true)
				}
				drop -= size
				queuedRateLimitCounter.Inc(int64(size))
				continue
			}
			// Otherwise drop only last few transactions
			txs := list.Flatten()
			for i := len(txs) - 1; i >= 0 && drop > 0; i-- {
				pool.removeTx(txs[i].Hash(), true)
				drop--
				queuedRateLimitCounter.Inc(1)
			}
		}
	}
}

以上就是promoteExecutables函数的工作过程,是tx_pool.go中最复杂的函数,也是维护以太坊交易池正常运行的核心函数,它的工作主旨就是在txpool资源紧张的时候,要优先保证本地account的交易先上链;刚开始看的时候,很容易懵逼,此时需要静下心来,慢慢梳理才会找到头绪.
```
以太坊普通节点发交易的交易要广播到矿机的txpool中才会被最终验证执行.promoteExecutables函数在调用完promoteTx函数之后,执行
```
go pool.txFeed.Send(NewTxsEvent{promoted})

这里发送了一个信号'NewTxsEvent'给网络部分,通知网络模块接收准备就绪的交易,将其广播出去.
```

看下eth/handler.go 这个文件
```
func (pm *ProtocolManager) Start(maxPeers int) {
	pm.maxPeers = maxPeers

	//这里创建了接收新交易的channl,并设置了相关的处理函数
	pm.txsCh = make(chan core.NewTxsEvent, txChanSize)
	pm.txsSub = pm.txpool.SubscribeNewTxsEvent(pm.txsCh)
	go pm.txBroadcastLoop()

	// broadcast mined blocks
	pm.minedBlockSub = pm.eventMux.Subscribe(core.NewMinedBlockEvent{})
	go pm.minedBroadcastLoop()

	// start sync handlers
	go pm.syncer()
	go pm.txsyncLoop()
}

func (pm *ProtocolManager) txBroadcastLoop() {
	for {
		select {
		case event := <-pm.txsCh:
			pm.BroadcastTxs(event.Txs)

		// Err() channel will be closed when unsubscribing.
		case <-pm.txsSub.Err():
			return
		}
	}
}
```
最终调用到peer.go文件中的这个函数,将交易队列发送出去;"TxMsg" 是msg type,其他节点要接收txs要识别这个消息码.
```
func (p *peer) SendTransactions(txs types.Transactions) error {
	for _, tx := range txs {
		p.knownTxs.Add(tx.Hash())
	}
	return p2p.Send(p.rw, TxMsg, txs)
}
```
交易队列就这样会发送到矿机上,等待矿机执行交易后,出块上链,未完待续......
