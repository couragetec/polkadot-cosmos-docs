#Project description

The final aim of this project is to create tools that allow building Polkadot parachains using Comos SDK.

## How it works?
In Cosmos SDK Tendermint core encapsulates the Network and Consensus layers of a node. It interacts with the Application layer, which defines the business logic of the system, using ABCI interface. The communication between customer application and Tendermint core is similar to ‘client-server’, customer application is the server, and Tendermint core is the client. 

![Figure 1](https://i.ibb.co/5sGYmNX/ABCI-Detailed-explanation.png)
Figure 1 - The interaction of two Cosmos nodes

### ABCI
ABCI contains a set of methods including:
- CheckTx - check and set transaction priority before adding to transaction pool
- BeginBlock - start block execution
- DeliverTx - execute a transaction from the block
- EndBlock - finish bock execution
- Commit - approve changes after block execution
- InitChain - load initial state from JSON file
- Query - query for data from the application
- Others.

Full specification can be found [here](https://docs.tendermint.com/master/spec/abci/abci.html). 
We've already finished the implementation of all ABCI methods excluding EndBlock (response processing is in progress) and the set of Snapshot methods.

## Polkadot-Cosmos integration

To minimize changes in Substrate and Cosmos cores, we add a new pallet `cosmos-abci` that contains one type of extrinsics - `abci_transaction`- that is used as a wrapper for regular Cosmos transactions.

We analyzed several possible solutions for integration:
- Runtime interfaces
- Off-chain workers;
- Common database.

###Runtime interfaces
Runtime interfaces allow us to use external libraries and create network connections directly from runtime. Initially, we used HTTP protocol that natively supported by Substrate and HTTP-GRPC gateway on the application (Cosmos) side. Then we add [Tonic library](https://github.com/hyperium/tonic) that provides GRPC protocol to Substrate node and now we send GRPC requests directly to Cosmos node.

![Figure 2](https://i.ibb.co/bP713sJ/Architecture-direct.png)
Figure 2 - Transaction and block processing in Substrate-Cosmos app

We've implemented cosmos-abci [pallet](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/pallets/cosmos-abci) that contains logic of calls to main ABCI methods (checkTx, deliverTx, beginBlock, endBlock, commit, initChain). These methods are called during chain initialization, transaction validation and block execution. Tonic library, necessary proto files and GRPC calls are situated [here](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/pallets/cosmos-abci/abci).

**Problem with on_initialize**
In general, the approach based on runtime interfaces seems to be the simplest and the most convenient. But we found a strange behavior that `on_initialize` can be called multiple times on the same block height and even after `on_finalize` on this height. Also, it was imposible to get a hash of the current block during `on_initialize`. According to the [reply](https://github.com/paritytech/subport/issues/43) of support: 
1) `on_initialize` is called any time that a runtime function is called from the client, so it is normal to see multiple calls to `on_initialize`.
2) There is no guarantee that `on_finalize` will execute after `on_initialize`.
3) `on_initialize` and `on_finalize` are called before the block hash and extrinsic root are calculated so it is expected that you would not be able to access them in either function; these values are calculated immediately after calling `on_finalize`.

That's why block processing (beginBlock, deliverTx, endBlock, commit) can't be done using these methods because it doesn't guarantee the order of their execution.
Runtime interfaces are used for transaction checking.

### Off-chain workers

Off-chain workers for each block are called when block processing is finished. During extrinsic processing, `abci_transaction` extrinsics are saved into storage. The off-chain worker calls `beginBlock`, then gets all necessary extrinsics from storage, calls `deliverTx` for each one, removes them from storage and calls `endBlock` and `commit`.
![Figure 3](https://i.ibb.co/TYwb6Kh/Off-chain-workers.png)
Figure 3 - Off-chain worker

### Common database
Offchain workers cannot write to the main storage of the node, but they have an additional local storage that can be accessed both from runtime and offchain workers. Also this storage is not synchronized between nodes. In fact, storage is a Rocks database, so potentially other processes on the same device can get access to this DB. 
The idea is: 
- offchain workers or runtime itself writes data (requests) that must be processed by Cosmos application (transactions, blocks, etc.) to the local storage.
- Cosmos app is continuously monitoring the DB, when it finds new appropriate data, it takes it and processes. Then Cosmos removes requests from DB and writes responses.
- During processing of a new block/transaction Substrate finds responses in DB, performs necessary actions and removes the responses..

RocksDB does not support full interaction with the database by two processes, it only allows the "main" process to read / write, and the "secondary" process only to read, which categorically doesn’t suit us. 
It seems to be a bad idea to implement data exchange through a common file in this context, because Cosmos will have to constantly read some sections in the database, looking for the requests. It will be slow and difficult to implement. Because the Substrate can't “trigger” Cosmos when writing to the DB, unlike HTTP-requests approach, where Cosmos start acting after a request from Substrate. Theoretically we can change the DB  layer but it seems to be unnecessary work with a lot of potential problems.

 Moreover, RocksDB doesn't support simultaneous work with two processes when both can write to it.
https://github.com/facebook/rocksdb/wiki/Secondary-instance
https://github.com/facebook/rocksdb/issues/908
https://github.com/facebook/rocksdb/issues/118


## Cosmos CLI and RPC
A user can interact with its Cosmos node using CLI. This CLI doesn't connect directly to the Cosmos node, but to the Tendermint core that redirects requests from the CLI to the node using ABCI and then returns responses to the CLI. It allows developers to use the same CLI for different applications just connecting Tendermint to them.
For interactions with CLI, Tendermint contains an additional json-rpc server, a bit similar to the server used Substrate for interaction with web interface. 
We implemented an [rpc server](https://github.com/adoriasoft/polkadot_cosmos_integration/tree/master/substrate/node/src/cosmos_rpc) as a component of Substrate node to allow interactions between CLI and Cosmos. 
![Figure 4](https://i.ibb.co/nPJPqtJ/CLI-scheme.png)
Figure 4 - Interaction between Cosmos CLI and Cosmos node using Substrate RPC

Tendermint server provides the following [API](https://docs.tendermint.com/master/rpc/). All methods are divided into several groups:
- *Tx* - these methods creates new transactions - (done)
	`broadcast_tx_async`- Returns right away, with no response. Does not wait for CheckTx nor DeliverTx results.
	`broadcast_tx_sync` - Returns with the response from CheckTx. Does not wait for DeliverTx result.
	`broadcast_tx_commit` - Now acts the same way as `broadcast_tx_sync`. Acording to the original docs, this call should return with the responses from CheckTx and DeliverTx, but in our case when blocks are created by Substrate runtime and then executed by off-chain worker it's a challenging task to track the inclusion of specific transaction into a block. 
	`check_tx` - Checks the transaction without executing it.
- *ABCI* - calls to ABCI Info and Query methods - (done)
- *Unsafe API* 
We analyzed the use cases of `dial_seeds` and `dial_peers` RPCs and came to the conclusion that Substrate doesn't need it.
As in Tendermint docs specified: DialSeeds: Dial a peer, this route in under unsafe, and has to manually enabled to use and DialPeers: Set a persistent peer, this route in under unsafe, and has to manually enabled to use. Substrate already has pretty much the same API inside Node CLI (--bootnodes or --reserved-nodes) and p2p protocol that connects all nodes.
We could add them as dummy methods that do nothing but return a response with success but we think that this will be more unclear to ones who will use it.
- *Websocket* - (will be done)
- *Info* - (will be done)


##ABCI calls in Substrate
As described above, different ABCI methods are called in different places.

| \# | Method | Substrate method | Request | Response |
| :------------: | :------------: |
| 1 | initChain | load_spec |json genesis file | - |
| 2 | checkTx | validate_transaction | tx data | TransactionValidity struct or error |
| 3 | beginBlock | cosmos_abci::offchain_worker | Block header and system information | - |
| 4 | deliverTx | cosmos_abci::offchain_worker | tx data | - |
| 5 | endBlock | cosmos_abci::offchain_worker | Block height | - |
| 6 | commit | cosmos_abci::offchain_worker  | - | - |
| 7 | query | json-rpc server | Data from CLI | - |
| 8 | info | json-rpc server | - | - |
| 9 | setOption | - | - | - |
| 10 | echo | - | - | - |
| 11 | flush | - | - | - |

##Demo application
To demonstrate the correct work of our module, we chose a simple Cosmos SDK-based application "Nameservice": [tutorial](https://tutorials.cosmos.network/nameservice/tutorial/00-intro.html), [source code](https://github.com/cosmos/sdk-tutorials/tree/master/nameservice) .
Brief changes have been made in the code to launch the application with a newer version of Cosmos SDK, [updated version](https://github.com/adoriasoft/cosmos-sdk/tree/feature/add_nameservice). 

## Versions
Substrate - rc6
Cosmos SDK - master branch on commit [89097a0](https://github.com/adoriasoft/cosmos-sdk/commit/89097a00d7f6d6339c377f6c87bea8fa5068d125). 


## User guides
[Launch Cosmos](https://github.com/adoriasoft/cosmos-sdk/blob/feature/add_nameservice/simapp/README.md)
[Launch Substrate](https://github.com/adoriasoft/polkadot_cosmos_integration/blob/master/substrate/README.md )
[Send transaction](https://github.com/adoriasoft/cosmos-sdk/blob/feature/add_nameservice/simapp/README.md)