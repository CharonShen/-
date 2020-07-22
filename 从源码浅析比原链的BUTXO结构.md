# 从源码浅析比原链的BUTXO结构

目前主流公链中使用的记录保存方式有两种，以bitcoin为代表的UTXO模式和Ethereum 代表的 Account 模型。bytom的BUTXO，可以说是UTXO的拓展，也可以说是两种模式的的相互结合。

## UTXO模型和Account模型

传统的UTXO模型通过签名验证的逻辑，将所有的信息存放在UTXO中，不依赖抽象的上层对象，主要的代表是比特币。而Account模型通过抽象出一个账户的对象，存储账户余额、合约代码，可以更方便的实现具有图灵完备的虚拟机，这也是智能合约的运行基础。Account模型主要代表是以太坊和EOS。

UTXO模型和Account模型的最大区别在于无状态和有状态，前者实现图灵完备的智能合约是相对困难的，而后者从设计就是为了实现智能合约。

比原链使用的是BUTXO模型，通过拓展比特币的UTXO模型，保留了传统UTXO模型性能上的优势，同时实现了图灵完备的智能合约。



## 比原链的BUTXO结构

比原链BUTXO定义在`Account/utxo_keeper.go`中

```go
type UTXO struct {
	OutputID            bc.Hash		//输出ID
	SourceID            bc.Hash		//来源ID
	AssetID             bc.AssetID	//资产ID
	Amount              uint64		//总额
	SourcePos           uint64		//源位置
	ControlProgram      []byte		//控制程序
	AccountID           string		//账户ID
	Address             string		//地址
	ControlProgramIndex uint64		//控制程序索引
	ValidHeight         uint64		//有效高度
	Change              bool
}
```

其中比较特殊的是AssetID，ControlProgram字段， 这两个字段是为了实现多资产和智能合约而加入的，而AccountID表明了UTXO具体归属于哪个账户，为BUTXO的账户管理提供便利。

UTXO集合存储在reservation中的utxos字段：

```go
type reservation struct {
	id     uint64
	utxos  []*UTXO
	change uint64
	expiry time.Time
}
```

对比原链BUTXO的操作，主要依赖utxoKeeper的方法，包括后面账户对UTXO的操作，也是通过utxoKeeper的调用来完成。

```go
type utxoKeeper struct {
	nextIndex     uint64
	db            dbm.DB
	mtx           sync.RWMutex
	currentHeight func() uint64

	unconfirmed  map[bc.Hash]*UTXO
	reserved     map[bc.Hash]uint64
	reservations map[uint64]*reservation
}
```



## BUTXO的账户管理

比特币这种传统的UTXO模型在钱包层是没有账户概念的，而比原链为了更好的实现智能合约，在钱包层设计了账户的概念。账户通过控制程序来确定资产在区块链的所有权。

`Account`结构：

```go
type Account struct {
   *signers.Signer				//账户签名
   ID    string `json:"id"`		//账户ID
   Alias string `json:"alias"`	//账户别名
}
```

然后来看看账户的控制器：

```go
type Manager struct {
   db         dbm.DB
   chain      *protocol.Chain
   utxoKeeper *utxoKeeper

   cacheMu    sync.Mutex
   cache      *lru.Cache
   aliasCache *lru.Cache

   delayedACPsMu sync.Mutex
   delayedACPs   map[*txbuilder.TemplateBuilder][]*CtrlProgram

   addressMu sync.Mutex
   accountMu sync.Mutex
}
```

Manager包含诸多方法，功能涵盖账户的创建、账户地址的创建（单地址和多地址，比原链一个账户可以有多个地址）下附源码：

账户创建：

```go
func (m *Manager) Create(xpubs []chainkd.XPub, quorum int, alias string, deriveRule uint8) (*Account, error) {
	m.accountMu.Lock()
	defer m.accountMu.Unlock()

	if existed := m.db.Get(aliasKey(alias)); existed != nil {
		return nil, ErrDuplicateAlias
	}

	acctIndex := uint64(1)
	if rawIndexBytes := m.db.Get(GetAccountIndexKey(xpubs)); rawIndexBytes != nil {
		acctIndex = common.BytesToUnit64(rawIndexBytes) + 1
	}
	account, err := CreateAccount(xpubs, quorum, alias, acctIndex, deriveRule)
	if err != nil {
		return nil, err
	}

	if err := m.saveAccount(account, true); err != nil {
		return nil, err
	}

	return account, nil
}
```

#### 账户余额的查询方式

比原链中账户余额就是该账户地址下所有UTXO资产的总和。账户的UTXO会持久化的存储在wallet的数据库内。计算余额，只需要将对应AccountID所有UTXO中的Amount相加。

`wallet/indexer.go`

```go
func (w *Wallet) GetAccountBalances(accountID string, id string) ([]AccountBalance, error) {
   return w.indexBalances(w.GetAccountUtxos(accountID, "", false, false))
}
```

`wallet/utxo.go`

```go
func (w *Wallet) GetAccountUtxos(accountID string, id string, unconfirmed, isSmartContract bool) []*account.UTXO {
   prefix := account.UTXOPreFix
   if isSmartContract {
      prefix = account.SUTXOPrefix
   }

   accountUtxos := []*account.UTXO{}
   if unconfirmed {
      accountUtxos = w.AccountMgr.ListUnconfirmedUtxo(accountID, isSmartContract)
   }

   accountUtxoIter := w.DB.IteratorPrefix([]byte(prefix + id))
   defer accountUtxoIter.Release()

   for accountUtxoIter.Next() {
      accountUtxo := &account.UTXO{}
      if err := json.Unmarshal(accountUtxoIter.Value(), accountUtxo); err != nil {
         log.WithFields(log.Fields{"module": logModule, "err": err}).Warn("GetAccountUtxos fail on unmarshal utxo")
         continue
      }

      if accountID == accountUtxo.AccountID || accountID == "" {
         accountUtxos = append(accountUtxos, accountUtxo)
      }
   }
   return accountUtxos
}
```

GetAccountUtxos方法根据prefix前缀从LevelDB中查找到所有的UTXO，然后通过账户和资产类型分类，最后将UTXO中的Amount相加，得出该账户的余额。