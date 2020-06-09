# Quorum-Ethereum AMB bridge

So far, the plan of Quuorum-Ethereum bridge is the following:
* Create public quorum testnet using IBFT consensus.
* Select existing etherum testnet (Kovan/Sokol).
* Deploy AMB bridge between two testnets.
* Deploy mediators/tokens in order to prove that everything works as expected.
* Deploy chain explorer for quorum chain (cakeshop/blockscout or both)
* Deploy faucet for quorum testnet. (Even though gasPrice is `0`, users may want to have native coins for other needs)

## Quorum testnet
For quorum testnet, the following genesis block (or a similar one) is going to be used:
```jsonld=
{
    "config": {
        "chainId": 111,
        "eip150Block": 0,
        "eip150Hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
        "eip155Block": 0,
        "eip158Block": 0,
        "homesteadBlock": 0,
        "byzantiumBlock": 0,
        "constantinopleBlock": 0,
        "petersburgBlock": 0,
        "istanbulBlock": 0,
        "istanbul": {
            "epoch": 30000,
            "policy": 0,
            "ceil2Nby3Block": 0
        },
        "maxCodeSizeConfig": [
            {
                "block": 0,
                "size": 35
            }
        ],
        "isQuorum": true
    },
    "nonce": "0x0",
    "timestamp": "0x5eda1803",
    "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000f85ad5947de1cd14693fbe1240011afe8c68d61012e51babb8410000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000c0",
    "gasLimit": "0x989680",
    "difficulty": "0x1",
    "mixHash": "0x63746963616c2062797a616e74696e65206661756c7420746f6c6572616e6365",
    "coinbase": "0x0000000000000000000000000000000000000000",
    "alloc": {
        "7de1cd14693fbe1240011afe8c68d61012e51bab": {
            "balance": "0x446c3b15f9926687d2c40534fdb564000000000000"
        }
    },
    "number": "0x0",
    "gasUsed": "0x0",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}
```

Exact geth parameters that are used for the quorum nodex configuration can be found in `configs/geth-config.toml`.
Some important values are:
```
[Eth]
NetworkId = 111 # chain id was selected to be 111, network id == chain id
...

[Eth.Miner]
GasFloor = 10000000 # block gas limit is set to be 10000000
GasCeil = 10000000
GasPrice = 0 # quorum transactions should have 0 gas price, therefore, miner should accept transactions with such gas price
...

[Eth.TxPool]
...
PriceLimit = 0 # there are no price limits for accepted transactions, since transactions should have zero gas price
PriceBump = 0
AccountSlots = 16 # any account can have 16 guaranteed executable transactions in the queue
GlobalSlots = 4096 # upper bound on the queue size for all pending executable transactions
AccountQueue = 64 # any account can have 64 guaranteed non-executable transactions in the queue
GlobalQueue = 1024 # upper bound on the queue size for all pending executable transactions
Lifetime = 10800000000000 # maximum time, for which non-executable transaction can be hold, 3 hours
TransactionSizeLimit = 64 # maximum transaction size if 64kb
MaxCodeSize = 35 # maximum contract code size if 35kb

[Eth.GPO]
Blocks = 0 # gas price oracle is not needed when gas price should be zero
Percentile = 0

[Eth.Istanbul]
RequestTimeout = 10000 # timeout for IBFT round, 10s
BlockPeriod = 5 # target block period, 5s

[Node]
...
HTTPPort = 8545 # HTTP RPC listens on port 8545
HTTPModules = ["eth", "net"] # the same modules are accessible on public Infura providers
WSPort = 8645 # WS RPC listens on port 8545
WSModules = ["eth", "net"] # the same modules are accessible on public Infura providers
...
```

Enhanced permissions contract-based model is not going to be used from the very beginning. If needed, it can be enabled later, by deploying required smart-contract and restarting the node.

## Deploy AMB

So far, after some tests in https://github.com/k1rill-fedoseev/quorum-bridge, no significant compatibility problems were found.
AMB contracts should be slightly modified in order to allow set of `gasPrice=0`. Similar change should be applied to the deployment scripts.
Quorum genesis block includes recent hardFork blocks (`istanbulBlock`), so there are no any invalid opcodes in the contracts.

Tests confirmed that, bridge successfully operates between these two chains. It is possible to send messages in both directions. Different mediator contracts work as well.

## Deploy chain explorer

For ease of interaction with the public Quorum testnet, the chain explorer will be also deployed. Blockscout is the primary option.

## Deploy faucet

We are planning to use either serverless faucet https://github.com/PhilippLgh/serverless-faucet with terminal-friendly UI as well https://github.com/PhilippLgh/create-eth-test-account. Backup plan is to setup POA-faucet (https://github.com/poanetwork/poa-faucet), that is used for Quorum and.

## Quorum node version remarks

The tests were performed on different versions of quorum's official docker image. The following bugs were noted:
* `2.5.0` - one of the Permission contracts cannot be deployed for some reason.
* `2.6.0` - estimateGas RPC method fails when transaction value is not specified. Corresponding PR was already merged, but no other release appeared yet.
* `latest` - everything works as expected, some side-effects were found. Pending transactions, that were already processed, may be processed once again, https://github.com/jpmorganchase/quorum/issues/1008.

## Istanbul tools usage

Compiled binary from https://github.com/jpmorganchase/istanbul-tools is included into this repo.
Example of usage:
```bash
# generate quorum genesis block, with 2 initial validators
./istanbul setup --quorum --nodes --num 2 --verbose
# this will print three important things:
# validators, configs: address + nodekey + enode URL
# static-nodes.json - file with enode URl for all validators
# genesis.json - genesis block configuration, extraData field describes initial set of validators, make sure to regenerate it when using different nodekeys for validators

# decode extra data field to see initial set of validators
./istanbul extra decode --extradata "0x00..."

# encode extra data for particular validators set
./istanbul extra encode --validators "0x<validator 1>,0x<validator 2>,..."

# generate validator address, from its nodekey
./istanbul address --nodekeyhex "<64 hex symbols>"
```
