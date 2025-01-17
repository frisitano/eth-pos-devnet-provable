This repository provides a docker-compose file to run a
fully-functional, local development network for Ethereum with
proof-of-stake enabled. This configuration uses
[Prysm](https://github.com/prysmaticlabs/prysm) as a consensus client
and [geth](https://github.com/ethereum/go-ethereum) /
[erigon](https://github.com/ledgerwatch/erigon) for execution. **It
starts from proof-of-stake** and does not go through the Ethereum merge.

This sets up a single node development network with 64
deterministically-generated validator keys to drive the creation of
blocks in an Ethereum proof-of-stake chain.

The development net is fully functional and allows for the deployment of
smart contracts and all the features that also come with the Prysm
consensus client such as its rich set of APIs for retrieving data from
the blockchain. This development net is a great way to understand the
internals of Ethereum proof-of-stake and to mess around with the
different settings that make the system possible.

# Running the devnet

First, checkout this repository and install docker. Then run:

``` bash
docker compose up -d
```

You will see the following:

``` example
$ docker compose up -d
[+] Running 7/7
[+] Running 10/10
 ✔ Container eth-pos-devnet-create-beacon-chain-genesis-1  Exited
 ✔ Container eth-pos-devnet-create-beacon-node-keys-1      Exited
 ✔ Container eth-pos-devnet-beacon-chain-2-1               Started
 ✔ Container eth-pos-devnet-beacon-chain-1-1               Started
 ✔ Container eth-pos-devnet-geth-genesis-1                 Exited
 ✔ Container eth-pos-devnet-geth-import-1                  Exited
 ✔ Container eth-pos-devnet-erigon-genesis-1               Started
 ✔ Container eth-pos-devnet-validator-1                    Started
 ✔ Container eth-pos-devnet-erigon-1                       Started
 ✔ Container eth-pos-devnet-geth-1                         Started
```

To stop the containers you can run `docker compose stop`. Each time you
restart, you can wipe the old data using `make clean`

Next, you can inspect the logs of the different services launched

``` bash
docker logs eth-pos-devnet-geth-1 -f
```

# Available Features

-   Starts from the Capella Ethereum hard fork
-   The network launches with a [Validator Deposit
    Contract](https://github.com/ethereum/consensus-specs/blob/dev/solidity_deposit_contract/deposit_contract.sol)
    deployed at address `0x4242424242424242424242424242424242424242`.
    This can be used to onboard new validators into the network by
    depositing 32 ETH into the contract
-   The default account used in the go-ethereum node is address
    `0x85da99c8a7c2c95964c8efd687e95e632fc533d6` which comes seeded with
    ETH for use in the network. This can be used to send transactions,
    deploy contracts, and more
-   The default account, `0x85da99c8a7c2c95964c8efd687e95e632fc533d6` is
    also set as the fee recipient for transaction fees proposed
    validators in Prysm. This address will be receiving the fees of all
    proposer activity
-   The go-ethereum JSON-RPC API is available at <http://geth:8545>
-   The Prysm client's REST APIs are available at
    <http://beacon-chain:3500>. For more info on what these APIs are,
    see [here](https://ethereum.github.io/beacon-APIs/)
-   The Prysm client also exposes a gRPC API at
    <http://beacon-chain:4000>


# Type 1 Prover Testing Procedure

The intent of this repo is to be able to test Erigon State Witnesses
against the Type-1 Prover. Using this devnet setup, here is a procedure for
creating some test data.

1.  Start the devnet up with `docker compose up`. If you've run
    previously, you might want to do a `make clean` to avoid running
    from a previous state.
2.  Wait for blocks to start being produced. This should only take a
    few seconds. You can use `polycli monitor` to quickly check that
    blocks are being created.
3.  Generate some load and test transactions. Use
    [polycli](https://github.com/maticnetwork/polygon-cli/blob/main/doc/polycli_loadtest.md)
    or some other tool to create transactions.
4.  Once the load is done, you can stop the devnet with `docker compose
      stop` if you ran in detached mode.
5.  Checkout and build
    [jerrigon](https://github.com/0xPolygonZero/erigon/tree/feat/zero) from the
    `feat/zero` branch. You can use `make all` to build everything.
6.  Create a copy of the erigon state directory to avoid corrupting
    things

``` bash
sudo cp -r execution/erigon/ execution/erigon.bak
sudo chown -R $USER:$USER execution/erigon.bak/
```
7.  Now we can start the Jerrigon fork of Erigon. This will give us RPC
    access to the state that we created in the previous steps.

``` bash
~/code/jerrigon/build/bin/erigon \
    --http \
    --http.api=eth,net,web3,erigon,engine,debug \
    --http.addr=0.0.0.0 \
    --http.corsdomain=* \
    --http.vhosts any \
    --ws \
    --nodiscover=true \
    --txpool.disable=true \
    --no-downloader=true \
    --maxpeers 0 \
    --datadir=./execution/erigon.bak \
    --networkid=32382 \
    --db.size.limit=1GB
```

8.  With the RPC running we can retrieve the blocks, witnesses, and use
    zero-bin to parse them. In my test case, I generated about 240 blocks
    worth of data so I'm going to use `seq 0 240` for generating ranges
    of block numbers for testing purposes

``` bash
# Create a directory for storing the outputs
mkdir out

# Call the zeroTracer to get the traces
seq 0 240 | awk '{print "curl -o " sprintf("out/wit_%02d", $0) ".json -H '"'"'Content-Type: application/json'"'"' -d '"'"'{\"method\":\"debug_traceBlockByNumber\",\"params\":[\"" sprintf("0x%X", $0) "\", {\"tracer\": \"zeroTracer\"}],\"id\":1,\"jsonrpc\":\"2.0\"}'"'"' http://127.0.0.1:8545"}' | bash

# download the blocks (this assumes you have foundry/cast installed)
seq 0 240 | awk '{print "cast block --full -j " $0 " > out/block_" sprintf("%02d", $0) ".json"}' | bash
```

9. At this point, we'll want to checkout and build
    [zero-bin](https://github.com/0xPolygonZero/zero-bin) in order to
    test proof generation. Make sure to checkout that repo and run
    `cargo build --release` to compile the application for
    testing. The snippets below assume
    [zero-bin](https://github.com/0xPolygonZero/zero-bin) has been
    checked out and compiled in `$HOME/code/zero-bin`. After
    compiling, the `leader` and `rpc` binaries will be created in the
    `target/release` folder.

``` bash
# use zero-bin to convert witness formats. This is a basic test
seq 0 240 | awk '{print "~/code/zero-bin/target/release/rpc fetch --rpc-url http://127.0.0.1:8545 --block-number " $0 " > " sprintf("out/zero_%02d", $0) ".json" }' | bash

# use zero-bin to generate a proof for the genesis block
./leader --arithmetic 16..23 --byte-packing 9..21 --cpu 12..25 --keccak 14..20 --keccak-sponge 9..15 --logic 12..18 --memory 17..28 --runtime in-memory -n 1 jerigon --rpc-url http://127.0.0.1:8545 --block-number 1 --proof-output-path 1.json
seq 2 240 | awk '{print "./leader --arithmetic 16..23 --byte-packing 9..21 --cpu 12..25 --keccak 14..20 --keccak-sponge 9..15 --logic 12..18 --memory 17..28  --runtime in-memory -n 4 jerigon --rpc-url http://127.0.0.1:8545 --block-number " $1 " --proof-output-path " $1 ".json --previous-proof " ($1 - 1) ".json"}'
```

## Operational Notes

- Pay attention to memory usage on the system running
  `zero-bin`. Certain transactions can consume a lot of memory and
  lead to an OOM.
- You'll want to run `zero-bin` on a system with at least 32GB of RAM.
- When you run `zero-bin`, a local file will be created with a name
  like `prover_state_*`. This file needs to be deleted if any of the
  [circuit sizes are changed](https://github.com/0xPolygonZero/zero-bin#leader-usage).
- There is a [useful script](https://github.com/0xPolygonZero/zero-bin/blob/assorted_fixes/tools/prove_blocks.sh) in `zero-bin` to run a range of proofs.


Both the state witness generation and the decoding logic are actively
being improved. We expect that the following transaction types /
use-cases should prove without issue:

- Empty blocks (important use case)
- EOA transfers
- ERC-20 mints & transfers
- ERC-721 mintes & transfers

### Short Cuts

This is a shortcut to create the genesis file allocations for our
mnemonic. This has already been hard coded into the genesis file, but
if you want to use a different testing account, you can use this.

``` bash
polycli wallet inspect --mnemonic "code code code code code code code code code code code quality" | jq '.Addresses[] | {"key": .ETHAddress, "value": { "balance": "0x21e19e0c9bab2400000"}}' | jq -s 'from_entries'
```
