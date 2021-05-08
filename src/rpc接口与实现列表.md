
# 打印输出所有注册的接口

为了方便观察 rpc 接口与对应的实现，我们在 `func (r *serviceRegistry) registerName(name string, rcvr interface{})` 注册的时候，注入一段打印接口详情的代码：

```go
var uniqMap = map[string]bool{}
func (r *serviceRegistry) registerName(name string, rcvr interface{}) error {
	rcvrVal := reflect.ValueOf(rcvr)
	if name == "" {
		return fmt.Errorf("no service name for type %s", rcvrVal.Type().String())
	}

	// 输出所有注册的接口
	fmt.Printf("\n\n")
	func (receiver reflect.Value, namespace string) {
		typ := receiver.Type()
		for m := 0; m < typ.NumMethod(); m++ {
			method := typ.Method(m)
			if method.PkgPath != "" {
				continue // method not exported
			}

			// 滤重
			if uniqMap[namespace+method.Name] {
				continue
			}

			cb := newCallback(receiver, method.Func)
			if cb == nil {
				continue // function invalid
			}

			var args []string
			for _, argType := range cb.argTypes {
				args = append(args, argType.String())
			}
			api := namespace+"."+formatName(method.Name)
			if l := len(api); l < 42 {
				space := ""
				for i := 0; i < 42-l; i++ {
					space += " "
				}
				api = api + space
			}
			fmt.Printf("\n%s => func (%s) %s(%s) // isSubscribe:%v", api,receiver.Type().String(), method.Name, strings.Join(args,", "),cb.isSubscribe)

			uniqMap[namespace+method.Name] = true
		}

	}(rcvrVal, name)
    ......
    ......
```

# rpc 接口与实现对应列表

有了这份列表数据后，我们就可以快速地查找 rpc 接口对应的具体实现，方便开发调试

需要注意的是，函数 receiver 名称带有 `Private` 的表示私有接口，不能通过 http/WebSocket 方式调用，只能通过 ipc 方式调用

`isSubscribe:true` 表示可以通过建立长连接订阅事件

```js
rpc.modules                                => func (*rpc.RPCService) Modules() // isSubscribe:false

admin.addPeer                              => func (*node.privateAdminAPI) AddPeer(string) // isSubscribe:false
admin.addTrustedPeer                       => func (*node.privateAdminAPI) AddTrustedPeer(string) // isSubscribe:false
admin.peerEvents                           => func (*node.privateAdminAPI) PeerEvents() // isSubscribe:true
admin.removePeer                           => func (*node.privateAdminAPI) RemovePeer(string) // isSubscribe:false
admin.removeTrustedPeer                    => func (*node.privateAdminAPI) RemoveTrustedPeer(string) // isSubscribe:false
admin.startHTTP                            => func (*node.privateAdminAPI) StartHTTP(*string, *int, *string, *string, *string) // isSubscribe:false
admin.startRPC                             => func (*node.privateAdminAPI) StartRPC(*string, *int, *string, *string, *string) // isSubscribe:false
admin.startWS                              => func (*node.privateAdminAPI) StartWS(*string, *int, *string, *string) // isSubscribe:false
admin.stopHTTP                             => func (*node.privateAdminAPI) StopHTTP() // isSubscribe:false
admin.stopRPC                              => func (*node.privateAdminAPI) StopRPC() // isSubscribe:false
admin.stopWS                               => func (*node.privateAdminAPI) StopWS() // isSubscribe:false


admin.datadir                              => func (*node.publicAdminAPI) Datadir() // isSubscribe:false
admin.nodeInfo                             => func (*node.publicAdminAPI) NodeInfo() // isSubscribe:false
admin.peers                                => func (*node.publicAdminAPI) Peers() // isSubscribe:false


debug.backtraceAt                          => func (*debug.HandlerT) BacktraceAt(string) // isSubscribe:false
debug.blockProfile                         => func (*debug.HandlerT) BlockProfile(string, uint) // isSubscribe:false
debug.cpuProfile                           => func (*debug.HandlerT) CpuProfile(string, uint) // isSubscribe:false
debug.freeOSMemory                         => func (*debug.HandlerT) FreeOSMemory() // isSubscribe:false
debug.gcStats                              => func (*debug.HandlerT) GcStats() // isSubscribe:false
debug.goTrace                              => func (*debug.HandlerT) GoTrace(string, uint) // isSubscribe:false
debug.memStats                             => func (*debug.HandlerT) MemStats() // isSubscribe:false
debug.mutexProfile                         => func (*debug.HandlerT) MutexProfile(string, uint) // isSubscribe:false
debug.setBlockProfileRate                  => func (*debug.HandlerT) SetBlockProfileRate(int) // isSubscribe:false
debug.setGCPercent                         => func (*debug.HandlerT) SetGCPercent(int) // isSubscribe:false
debug.setMutexProfileFraction              => func (*debug.HandlerT) SetMutexProfileFraction(int) // isSubscribe:false
debug.stacks                               => func (*debug.HandlerT) Stacks() // isSubscribe:false
debug.startCPUProfile                      => func (*debug.HandlerT) StartCPUProfile(string) // isSubscribe:false
debug.startGoTrace                         => func (*debug.HandlerT) StartGoTrace(string) // isSubscribe:false
debug.stopCPUProfile                       => func (*debug.HandlerT) StopCPUProfile() // isSubscribe:false
debug.stopGoTrace                          => func (*debug.HandlerT) StopGoTrace() // isSubscribe:false
debug.verbosity                            => func (*debug.HandlerT) Verbosity(int) // isSubscribe:false
debug.vmodule                              => func (*debug.HandlerT) Vmodule(string) // isSubscribe:false
debug.writeBlockProfile                    => func (*debug.HandlerT) WriteBlockProfile(string) // isSubscribe:false
debug.writeMemProfile                      => func (*debug.HandlerT) WriteMemProfile(string) // isSubscribe:false
debug.writeMutexProfile                    => func (*debug.HandlerT) WriteMutexProfile(string) // isSubscribe:false


web3.clientVersion                         => func (*node.publicWeb3API) ClientVersion() // isSubscribe:false
web3.sha3                                  => func (*node.publicWeb3API) Sha3(hexutil.Bytes) // isSubscribe:false


eth.gasPrice                               => func (*ethapi.PublicEthereumAPI) GasPrice() // isSubscribe:false
eth.syncing                                => func (*ethapi.PublicEthereumAPI) Syncing() // isSubscribe:false


eth.blockNumber                            => func (*ethapi.PublicBlockChainAPI) BlockNumber() // isSubscribe:false
eth.call                                   => func (*ethapi.PublicBlockChainAPI) Call(ethapi.CallArgs, rpc.BlockNumberOrHash, *map[common.Address]ethapi.account) // isSubscribe:false
eth.chainId                                => func (*ethapi.PublicBlockChainAPI) ChainId() // isSubscribe:false
eth.createAccessList                       => func (*ethapi.PublicBlockChainAPI) CreateAccessList(ethapi.SendTxArgs, *rpc.BlockNumberOrHash) // isSubscribe:false
eth.estimateGas                            => func (*ethapi.PublicBlockChainAPI) EstimateGas(ethapi.CallArgs, *rpc.BlockNumberOrHash) // isSubscribe:false
eth.getBalance                             => func (*ethapi.PublicBlockChainAPI) GetBalance(common.Address, rpc.BlockNumberOrHash) // isSubscribe:false
eth.getBlockByHash                         => func (*ethapi.PublicBlockChainAPI) GetBlockByHash(common.Hash, bool) // isSubscribe:false
eth.getBlockByNumber                       => func (*ethapi.PublicBlockChainAPI) GetBlockByNumber(rpc.BlockNumber, bool) // isSubscribe:false
eth.getCode                                => func (*ethapi.PublicBlockChainAPI) GetCode(common.Address, rpc.BlockNumberOrHash) // isSubscribe:false
eth.getHeaderByHash                        => func (*ethapi.PublicBlockChainAPI) GetHeaderByHash(common.Hash) // isSubscribe:false
eth.getHeaderByNumber                      => func (*ethapi.PublicBlockChainAPI) GetHeaderByNumber(rpc.BlockNumber) // isSubscribe:false
eth.getProof                               => func (*ethapi.PublicBlockChainAPI) GetProof(common.Address, []string, rpc.BlockNumberOrHash) // isSubscribe:false
eth.getStorageAt                           => func (*ethapi.PublicBlockChainAPI) GetStorageAt(common.Address, string, rpc.BlockNumberOrHash) // isSubscribe:false
eth.getUncleByBlockHashAndIndex            => func (*ethapi.PublicBlockChainAPI) GetUncleByBlockHashAndIndex(common.Hash, hexutil.Uint) // isSubscribe:false
eth.getUncleByBlockNumberAndIndex          => func (*ethapi.PublicBlockChainAPI) GetUncleByBlockNumberAndIndex(rpc.BlockNumber, hexutil.Uint) // isSubscribe:false
eth.getUncleCountByBlockHash               => func (*ethapi.PublicBlockChainAPI) GetUncleCountByBlockHash(common.Hash) // isSubscribe:false
eth.getUncleCountByBlockNumber             => func (*ethapi.PublicBlockChainAPI) GetUncleCountByBlockNumber(rpc.BlockNumber) // isSubscribe:false


eth.fillTransaction                        => func (*ethapi.PublicTransactionPoolAPI) FillTransaction(ethapi.SendTxArgs) // isSubscribe:false
eth.getBlockTransactionCountByHash         => func (*ethapi.PublicTransactionPoolAPI) GetBlockTransactionCountByHash(common.Hash) // isSubscribe:false
eth.getBlockTransactionCountByNumber       => func (*ethapi.PublicTransactionPoolAPI) GetBlockTransactionCountByNumber(rpc.BlockNumber) // isSubscribe:false
eth.getRawTransactionByBlockHashAndIndex   => func (*ethapi.PublicTransactionPoolAPI) GetRawTransactionByBlockHashAndIndex(common.Hash, hexutil.Uint) // isSubscribe:false
eth.getRawTransactionByBlockNumberAndIndex => func (*ethapi.PublicTransactionPoolAPI) GetRawTransactionByBlockNumberAndIndex(rpc.BlockNumber, hexutil.Uint) // isSubscribe:false
eth.getRawTransactionByHash                => func (*ethapi.PublicTransactionPoolAPI) GetRawTransactionByHash(common.Hash) // isSubscribe:false
eth.getTransactionByBlockHashAndIndex      => func (*ethapi.PublicTransactionPoolAPI) GetTransactionByBlockHashAndIndex(common.Hash, hexutil.Uint) // isSubscribe:false
eth.getTransactionByBlockNumberAndIndex    => func (*ethapi.PublicTransactionPoolAPI) GetTransactionByBlockNumberAndIndex(rpc.BlockNumber, hexutil.Uint) // isSubscribe:false
eth.getTransactionByHash                   => func (*ethapi.PublicTransactionPoolAPI) GetTransactionByHash(common.Hash) // isSubscribe:false
eth.getTransactionCount                    => func (*ethapi.PublicTransactionPoolAPI) GetTransactionCount(common.Address, rpc.BlockNumberOrHash) // isSubscribe:false
eth.getTransactionReceipt                  => func (*ethapi.PublicTransactionPoolAPI) GetTransactionReceipt(common.Hash) // isSubscribe:false
eth.pendingTransactions                    => func (*ethapi.PublicTransactionPoolAPI) PendingTransactions() // isSubscribe:false
eth.resend                                 => func (*ethapi.PublicTransactionPoolAPI) Resend(ethapi.SendTxArgs, *hexutil.Big, *hexutil.Uint64) // isSubscribe:false
eth.sendRawTransaction                     => func (*ethapi.PublicTransactionPoolAPI) SendRawTransaction(hexutil.Bytes) // isSubscribe:false
eth.sendTransaction                        => func (*ethapi.PublicTransactionPoolAPI) SendTransaction(ethapi.SendTxArgs) // isSubscribe:false
eth.sign                                   => func (*ethapi.PublicTransactionPoolAPI) Sign(common.Address, hexutil.Bytes) // isSubscribe:false
eth.signTransaction                        => func (*ethapi.PublicTransactionPoolAPI) SignTransaction(ethapi.SendTxArgs) // isSubscribe:false


txpool.content                             => func (*ethapi.PublicTxPoolAPI) Content() // isSubscribe:false
txpool.inspect                             => func (*ethapi.PublicTxPoolAPI) Inspect() // isSubscribe:false
txpool.status                              => func (*ethapi.PublicTxPoolAPI) Status() // isSubscribe:false


debug.getBlockRlp                          => func (*ethapi.PublicDebugAPI) GetBlockRlp(uint64) // isSubscribe:false
debug.printBlock                           => func (*ethapi.PublicDebugAPI) PrintBlock(uint64) // isSubscribe:false
debug.seedHash                             => func (*ethapi.PublicDebugAPI) SeedHash(uint64) // isSubscribe:false
debug.testSignCliqueBlock                  => func (*ethapi.PublicDebugAPI) TestSignCliqueBlock(common.Address, uint64) // isSubscribe:false


debug.chaindbCompact                       => func (*ethapi.PrivateDebugAPI) ChaindbCompact() // isSubscribe:false
debug.chaindbProperty                      => func (*ethapi.PrivateDebugAPI) ChaindbProperty(string) // isSubscribe:false
debug.setHead                              => func (*ethapi.PrivateDebugAPI) SetHead(hexutil.Uint64) // isSubscribe:false


eth.accounts                               => func (*ethapi.PublicAccountAPI) Accounts() // isSubscribe:false


personal.deriveAccount                     => func (*ethapi.PrivateAccountAPI) DeriveAccount(string, string, *bool) // isSubscribe:false
personal.ecRecover                         => func (*ethapi.PrivateAccountAPI) EcRecover(hexutil.Bytes, hexutil.Bytes) // isSubscribe:false
personal.importRawKey                      => func (*ethapi.PrivateAccountAPI) ImportRawKey(string, string) // isSubscribe:false
personal.initializeWallet                  => func (*ethapi.PrivateAccountAPI) InitializeWallet(string) // isSubscribe:false
personal.listAccounts                      => func (*ethapi.PrivateAccountAPI) ListAccounts() // isSubscribe:false
personal.listWallets                       => func (*ethapi.PrivateAccountAPI) ListWallets() // isSubscribe:false
personal.lockAccount                       => func (*ethapi.PrivateAccountAPI) LockAccount(common.Address) // isSubscribe:false
personal.newAccount                        => func (*ethapi.PrivateAccountAPI) NewAccount(string) // isSubscribe:false
personal.openWallet                        => func (*ethapi.PrivateAccountAPI) OpenWallet(string, *string) // isSubscribe:false
personal.sendTransaction                   => func (*ethapi.PrivateAccountAPI) SendTransaction(ethapi.SendTxArgs, string) // isSubscribe:false
personal.sign                              => func (*ethapi.PrivateAccountAPI) Sign(hexutil.Bytes, common.Address, string) // isSubscribe:false
personal.signAndSendTransaction            => func (*ethapi.PrivateAccountAPI) SignAndSendTransaction(ethapi.SendTxArgs, string) // isSubscribe:false
personal.signTransaction                   => func (*ethapi.PrivateAccountAPI) SignTransaction(ethapi.SendTxArgs, string) // isSubscribe:false
personal.unlockAccount                     => func (*ethapi.PrivateAccountAPI) UnlockAccount(common.Address, string, *uint64) // isSubscribe:false
personal.unpair                            => func (*ethapi.PrivateAccountAPI) Unpair(string, string) // isSubscribe:false


eth.getHashrate                            => func (*ethash.API) GetHashrate() // isSubscribe:false
eth.getWork                                => func (*ethash.API) GetWork() // isSubscribe:false
eth.submitHashrate                         => func (*ethash.API) SubmitHashrate(hexutil.Uint64, common.Hash) // isSubscribe:false
eth.submitWork                             => func (*ethash.API) SubmitWork(types.BlockNonce, common.Hash, common.Hash) // isSubscribe:false


ethash.getHashrate                         => func (*ethash.API) GetHashrate() // isSubscribe:false
ethash.getWork                             => func (*ethash.API) GetWork() // isSubscribe:false
ethash.submitHashrate                      => func (*ethash.API) SubmitHashrate(hexutil.Uint64, common.Hash) // isSubscribe:false
ethash.submitWork                          => func (*ethash.API) SubmitWork(types.BlockNonce, common.Hash, common.Hash) // isSubscribe:false


eth.coinbase                               => func (*eth.PublicEthereumAPI) Coinbase() // isSubscribe:false
eth.etherbase                              => func (*eth.PublicEthereumAPI) Etherbase() // isSubscribe:false


eth.mining                                 => func (*eth.PublicMinerAPI) Mining() // isSubscribe:false


eth.subscribeSyncStatus                    => func (*downloader.PublicDownloaderAPI) SubscribeSyncStatus(chan interface {}) // isSubscribe:false


miner.setEtherbase                         => func (*eth.PrivateMinerAPI) SetEtherbase(common.Address) // isSubscribe:false
miner.setExtra                             => func (*eth.PrivateMinerAPI) SetExtra(string) // isSubscribe:false
miner.setGasPrice                          => func (*eth.PrivateMinerAPI) SetGasPrice(hexutil.Big) // isSubscribe:false
miner.setRecommitInterval                  => func (*eth.PrivateMinerAPI) SetRecommitInterval(int) // isSubscribe:false
miner.start                                => func (*eth.PrivateMinerAPI) Start(*int) // isSubscribe:false
miner.stop                                 => func (*eth.PrivateMinerAPI) Stop() // isSubscribe:false


eth.getFilterChanges                       => func (*filters.PublicFilterAPI) GetFilterChanges(rpc.ID) // isSubscribe:false
eth.getFilterLogs                          => func (*filters.PublicFilterAPI) GetFilterLogs(rpc.ID) // isSubscribe:false
eth.getLogs                                => func (*filters.PublicFilterAPI) GetLogs(filters.FilterCriteria) // isSubscribe:false
eth.logs                                   => func (*filters.PublicFilterAPI) Logs(filters.FilterCriteria) // isSubscribe:true
eth.newBlockFilter                         => func (*filters.PublicFilterAPI) NewBlockFilter() // isSubscribe:false
eth.newFilter                              => func (*filters.PublicFilterAPI) NewFilter(filters.FilterCriteria) // isSubscribe:false
eth.newHeads                               => func (*filters.PublicFilterAPI) NewHeads() // isSubscribe:true
eth.newPendingTransactionFilter            => func (*filters.PublicFilterAPI) NewPendingTransactionFilter() // isSubscribe:false
eth.newPendingTransactions                 => func (*filters.PublicFilterAPI) NewPendingTransactions() // isSubscribe:true
eth.uninstallFilter                        => func (*filters.PublicFilterAPI) UninstallFilter(rpc.ID) // isSubscribe:false


admin.exportChain                          => func (*eth.PrivateAdminAPI) ExportChain(string, *uint64, *uint64) // isSubscribe:false
admin.importChain                          => func (*eth.PrivateAdminAPI) ImportChain(string) // isSubscribe:false


debug.accountRange                         => func (*eth.PublicDebugAPI) AccountRange(rpc.BlockNumberOrHash, []uint8, int, bool, bool, bool) // isSubscribe:false
debug.dumpBlock                            => func (*eth.PublicDebugAPI) DumpBlock(rpc.BlockNumber) // isSubscribe:false


debug.getBadBlocks                         => func (*eth.PrivateDebugAPI) GetBadBlocks() // isSubscribe:false
debug.getModifiedAccountsByHash            => func (*eth.PrivateDebugAPI) GetModifiedAccountsByHash(common.Hash, *common.Hash) // isSubscribe:false
debug.getModifiedAccountsByNumber          => func (*eth.PrivateDebugAPI) GetModifiedAccountsByNumber(uint64, *uint64) // isSubscribe:false
debug.preimage                             => func (*eth.PrivateDebugAPI) Preimage(common.Hash) // isSubscribe:false
debug.storageRangeAt                       => func (*eth.PrivateDebugAPI) StorageRangeAt(common.Hash, int, common.Address, hexutil.Bytes, int) // isSubscribe:false


net.listening                              => func (*ethapi.PublicNetAPI) Listening() // isSubscribe:false
net.peerCount                              => func (*ethapi.PublicNetAPI) PeerCount() // isSubscribe:false
net.version                                => func (*ethapi.PublicNetAPI) Version() // isSubscribe:false


debug.standardTraceBadBlockToFile          => func (*tracers.API) StandardTraceBadBlockToFile(common.Hash, *tracers.StdTraceConfig) // isSubscribe:false
debug.standardTraceBlockToFile             => func (*tracers.API) StandardTraceBlockToFile(common.Hash, *tracers.StdTraceConfig) // isSubscribe:false
debug.traceBadBlock                        => func (*tracers.API) TraceBadBlock(common.Hash, *tracers.TraceConfig) // isSubscribe:false
debug.traceBlock                           => func (*tracers.API) TraceBlock([]uint8, *tracers.TraceConfig) // isSubscribe:false
debug.traceBlockByHash                     => func (*tracers.API) TraceBlockByHash(common.Hash, *tracers.TraceConfig) // isSubscribe:false
debug.traceBlockByNumber                   => func (*tracers.API) TraceBlockByNumber(rpc.BlockNumber, *tracers.TraceConfig) // isSubscribe:false
debug.traceBlockFromFile                   => func (*tracers.API) TraceBlockFromFile(string, *tracers.TraceConfig) // isSubscribe:false
debug.traceCall                            => func (*tracers.API) TraceCall(ethapi.CallArgs, rpc.BlockNumberOrHash, *tracers.TraceConfig) // isSubscribe:false
debug.traceChain                           => func (*tracers.API) TraceChain(rpc.BlockNumber, rpc.BlockNumber, *tracers.TraceConfig) // isSubscribe:true
debug.traceTransaction                     => func (*tracers.API) TraceTransaction(common.Hash, *tracers.TraceConfig) // isSubscribe:false
```