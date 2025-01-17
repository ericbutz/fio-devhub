---
layout: page-int
title: Account balances and history
description: Account balances and history
---

# Retrieving account balances and history

*(Thanks to [blockpane](https://github.com/fioprotocol/fio-go/tree/master/_example/_blocks){:target="_blank"} for the original version of these notes.)*

There are many options available for wallets, exchanges, and information providers for presenting a user's balance and history. Not all approaches are appropriate for all applications, and how an organization integrates other blockchains may affect the strategy. The following summarizes a few of the approaches tried with FIO to help explain some of the choices available during integration.

These methods mostly assume the integrator will be self-hosting their own FIO node. However, some of the options mentioned here can work with [publicly available v1 history nodes and Hyperion](https://github.com/fioprotocol/fio.mainnet#api-endpoints){:target="_blank"}.

---
## Don't forget about the fees!

FIO differs from EOS in many ways, but one major difference may present a challenge for an integrator: fees. Fees are attached to many transaction types, and in some cases the fees are waived based on "bundled" transactions provided when a user registers a new FIO Crypto Handle (aka FIO Address). When submitting a transaction, there is a `max_fee` field that a user submits and if the fee required is less than this amount the fee is deducted. FIO will not extract more than the actual fee required for the call (unlike ethereum or bitcoin fees.) It is not a safe assumption that the `max_fee` value will be what was charged. Fees vary over time. The FIO block producers adjust fees through a consensus-based voting process.

There is one **very important** thing to note about fee collection: it is an internal-action to the contracts. Without an action trace, fees assessed or rewards paid will not be evident in a transaction.

---
## Getting account information

There are two approaches to getting account and transaction information: on-demand queries and pre-processing the data.

### On-Demand Queries

##### 1) Account action traces using the v1 history plugin with [get_actions] API call for each account
 
This method provides a wealth of information but it comes with some caveats. History can be truncated depending on how the node is configured, and understanding action traces can be complex. For example, some of the internal actions are repeated, only changing the receiver (for example for fee collection).

See [Processesing transactions from FIO History v1 Node](https://github.com/fioprotocol/fiosdk_typescript/tree/master/examples/FioTransactionHistory){:target="_blank"}  for an example of how to process FIO transactions from a v1 History node using the FIO Typescript SDK.

##### 2) Account balance using the [get_fio_balance]({{site.baseurl}}/pages/api/fio-api/#post-/get_fio_balance) API
   
Token balance can be obtained by passing FIO Public Key to the [/get_fio_balance]({{site.baseurl}}/pages/api/fio-api/#post-/get_fio_balance) API method. This should always return a correct balance.

### Pre-processing Options

There are two approaches often seen in pre-processing: pulling data via repeated requests and streaming data via websocket. Because of the overhead of making many repeated HTTP requests, and because `nodeos` does not support pipelining, the streaming options are going to be significantly faster. (At some point a Unix domain socket option may become available making the pull option less inefficient. But, as of the time of writing, it is not yet enabled in the http_plugin.) After each solution below the complexity is ranked in terms of *complexity* (required infrastructure to run the solution), *difficulty* (how difficult it is to handle the information from the approach,) and *quality* (is the data complete? Is it trustworthy?)

##### 1) Crawl the blocks using [get_block]({{site.baseurl}}/pages/api/fio-api/#post-/get_block) API 

*(low complexity, low difficulty, low quality)*
   
This is a common method with no additional plugins required. It is also slow and can result in missing information. We caution against using this method if accuracy is critical.   

At a high level, the process for crawling blocks looks like:

* Call [/get_info]({{site.baseurl}}/pages/api/fio-api/#post-/get_info) to get the latest irreversible block's height "last_irreversible_block_num" (which never rolls back)
* Call [/get_block]({{site.baseurl}}/pages/api/fio-api/#post-/get_block) for each block height with {"block_num_or_id":height} to get "transactions" array, which contains all transactions on the block
* Loop "transactions" array and get "transactions"[i]
* Check whether "transactions"[i]."status" == "executed", if yes go to next step, if no skip this transaction
* Get "transactions"[i]."trx"."transaction"."actions"[0]."data" as "data"
* Check whether "data"."payee_public_key" is the correct deposit address. If yes, then "data"."actor" is the sender address and "data"."amount"/1000000000 is the amount of FIO sent

There are several issues with this approach: 

* Action traces are not included in the transactions so seeing fees being charged, and rewards payouts is not possible.
* Supporting the tracking of multi-signature transactions requires special handling. Multi-signature transactions will give a different data structure for the transaction. The msig transactions will not have any structure at all, only a string with the transaction ID. For these it will be necessary to also call the get_transaction API using both the block number (non-history nodes require a block hint) and the transaction ID.

[get-block.go](https://github.com/fioprotocol/fio-go/blob/master/_example/_blocks/get-block.go){:target="_blank"}  is an example of how to get transactions from get_block.

##### 2) Crawl the blocks and then fetch full traces using the v1 history `get_block_txids` and `get_transaction` endpoints

*(low complexity, low difficulty, high quality)*
   
The major downside to this approach is that it requires many calls to get all of the transactions, but it does result in having full action-traces available and ensures multi-sig transactions are not missed. 

[v1history.go](https://github.com/fioprotocol/fio-go/blob/master/_example/_blocks/v1history.go){:target="_blank"} is an example of how to use the v1 history endpoints `get_block_txids` and `get_transaction` endpoints to get transaction information.

##### 3) Consume action traces via websocket using the state-history-plugin

*(low complexity, high difficulty, high quality)*
   
The state-history plugin is very fast and efficient at providing data, but it is difficult to understand and use directly. Queries are specified using ABI-encoded binary requests, and the data returned is also ABI encoded. Generally this is how many of the more advanced tools ingest the data before normalizing it. 

##### 4) [Chronicle](https://github.com/EOSChronicleProject/eos-chronicle){:target="_blank"} 

*(high complexity, low difficulty, high quality)*
   
Chronicle is a tool that consumes the state-history-plugins data and converts it to JSON. It sends this data over an outgoing websocket for processing. There are some challenges here too, as many of the numeric fields are changed to a string, which can be problematic for strongly-typed languages. Chronicle has a lot of options, making it a very good choice for when integrating into a custom data backend. 
   
[fio.etl](https://github.com/fioprotocol/fio.etl){:target="_blank"} is an example of a tool that uses Chronicle.

##### 5) [Hyperion](https://hyperion.docs.eosrio.io/){:target="_blank"} 

*(high complexity, low difficulty, high quality)*
   
Hyperion history adds a large number of capabilities including streaming APIs with filtering support, v1 history compatible APIs plus many additional useful endpoints. It is a somewhat complex app, involving message queues, key-value stores, ingest processes, and an elasticsearch backend. 

##### 6) Consume blocks via P2P

*(low complexity, high difficulty, low quality)*
   
This method is only recommended for near-real-time monitoring. It is possible to have a node push blocks directly over a TCP connection using the EOS p2p protocol, and then to process each block using the ABI to decode the transactions. This has the same downsides as using `get_block` and the added complexity of handling the binary protocol, but it is useful for handling data real-time. 
   
[fiowatch](https://github.com/blockpane/fiowatch){:target="_blank"} is an example of a tool that consumes blocks via P2P. 
