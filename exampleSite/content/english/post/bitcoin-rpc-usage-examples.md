+++
author = "Ruan Bekker"
title = "Bitcoin JSON RPC Usage Examples"
date = "2022-02-15"
description = "This tutorial will demonstrate how to interact with your bitcoin wallet node using the json rpc."
tags = [
    "blockchain",
    "crypto",
    "bitcoin",
    "rpc",
]
categories = [
    "bitcoin",
]
series = ["blockchain-introduction"]
+++

From our [previous tutorial](https://blockchainguides.net/post/setup-a-bitcoin-fullnode/), we've setup a bitcoin fullnode on the testnet chain and in this tutorial we will use the bitcoin cli as well as the json rpc to interact with our wallet and the blockchain using the curl cli utility.

## Bitcoin CLI

Before we get to use the rpc interface, we can use the `bitcoin-cli` command line utility to browse around. For any documentation on these methods, you can visit [chainquery](https://chainquery.com/bitcoin-cli).

To get the current block count:

```bash
$ bitcoin-cli getblockcount
2091215
```

To get some basic info about the first block ever created on the bitcoin blockchain. As the genesis block, it has the index value 0. We can use `getblockhas` to get the hash value for the first block ever created:

```bash
$ bitcoin-cli getblockhash 0
000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943
```

We can now use `getblock` with the hash value to retrieve details about the block:

```bash
$ bitcoin-cli getblock 000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943
{
  "hash": "000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943",
  "confirmations": 2004218,
  "strippedsize": 285,
  "size": 285,
  "weight": 1140,
  "height": 0,
  "version": 1,
  "versionHex": "00000001",
  "merkleroot": "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b",
  "tx": [
    "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b"
  ],
  "time": 1296688602,
  "mediantime": 1296688602,
  "nonce": 414098458,
  "bits": "1d00ffff",
  "difficulty": 1,
  "chainwork": "0000000000000000000000000000000000000000000000000000000100010001",
  "nTx": 1,
  "nextblockhash": "00000000b873e79784647a6c82962c70d228557d24a747ea4d1b8bbe878e1206"
}
```

We can use the `nextblockhash` to retrieve information about block 1:

```bash
$ bitcoin-cli getblock 00000000b873e79784647a6c82962c70d228557d24a747ea4d1b8bbe878e1206
{
  "hash": "00000000b873e79784647a6c82962c70d228557d24a747ea4d1b8bbe878e1206",
  "confirmations": 2004217,
  "strippedsize": 190,
  "size": 190,
  "weight": 760,
  "height": 1,
  "version": 1,
  "versionHex": "00000001",
  "merkleroot": "f0315ffc38709d70ad5647e22048358dd3745f3ce3874223c80a7c92fab0c8ba",
  "tx": [
    "f0315ffc38709d70ad5647e22048358dd3745f3ce3874223c80a7c92fab0c8ba"
  ],
  "time": 1296688928,
  "mediantime": 1296688928,
  "nonce": 1924588547,
  "bits": "1d00ffff",
  "difficulty": 1,
  "chainwork": "0000000000000000000000000000000000000000000000000000000200020002",
  "nTx": 1,
  "previousblockhash": "000000000933ea01ad0ee984209779baaec3ced90fa3f408719526f8d77f4943",
  "nextblockhash": "000000006c02c8ea6e4ff69651f7fcde348fb9d557a06e6957b65552002a7820"
}
```

We can use `-getinfo` to get information show as the verification progress and balances in our wallets, should they exist:

```bash
$ bitcoin-cli -getinfo
{
  "version": 210100,
  "blocks": 2091215,
  "headers": 2091215,
  "verificationprogress": 0.9999993040419591,
  "timeoffset": 0,
  "connections": {
    "in": 0,
    "out": 10,
    "total": 10
  },
  "proxy": "",
  "difficulty": 16777216,
  "chain": "test",
  "relayfee": 0.00001000,
  "warnings": "Warning: unknown new rules activated (versionbit 28)",
  "balances": {
  }
}
```

For more examples view [chainquery](https://chainquery.com/bitcoin-cli)

## JSON RPC

First get the rpc user and password from the `bitcoin.conf`:

```bash
$ cat /etc/bitcoin/bitcoin.conf | grep -E '(rpcuser|rpcpassword)'
rpcuser=bitcoin
rpcpassword=xxxxxxxxxxxxxx
```

Because this is a test environment, we can set the username and password as an environment variable:

```bash
# the user and password might differ on your setup
$ export bitcoinauth="bitcoin:xxxxxxxxxxxxxx"
```

### Wallet Interaction

A wallet is a collection of bitcoin addresses, go ahead and create a wallet named `wallet`:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "createwallet", "params": ["wallet"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":{"name":"wallet","warning":""},"error":null,"id":"curltest"}
```

Now list our wallets:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "tutorial", "method": "listwallets", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":["wallet"],"error":null,"id":"tutorial"}
```

If we inspect the `bitcoin.conf` we will notice that we don't have the wallet loaded in our config:

```bash
$ cat ~/.bitcoin/bitcoin.conf | grep -c 'wallet='
0
```

But since we created the wallet, we can see the `wallet.dat` in our data directory:

```bash
$ find /blockchain/ -type f -name wallet.dat
/blockchain/bitcoin/data/testnet3/wallets/wallet/wallet.dat
```

Let's restart the `bitcoind` service and see if we can still list our wallet, first restart the service:

```bash
$ sudo systemctl restart bitcoind
```

Then wait a couple of seconds and list the wallets:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "tutorial", "method": "listwallets", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":[],"error":null,"id":"tutorial"}
```

As you can see it's not loaded, due to it not being in the config. Let's update our config `/etc/bitcoin/bitcoin.conf`, and restart the service: 

```
# more wallets can be referenced by using another wallet= config
[test]
wallet=wallet
# which corresponds to datadir + walletdir
# /blockchain/bitcoin/data/testnet3/wallets/wallet/wallet.dat
# /blockchain/bitcoin/data/testnet3/wallets/wallet/db.log
```

When I restarted the `bitcoind` service, I checked the logs with `journalctl -fu bitcoind`, and I could see the wallet has been loaded:

```
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z init message: Loading wallet...
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z BerkeleyEnvironment::Open: LogDir=/blockchain/bitcoin/data/testnet3/wallets/wallet/database ErrorFile=/blockchain/bitcoin/data/testnet3/wallets/wallet//db.log
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z txindex thread start
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z txindex is enabled at height 2004267
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z txindex thread exit
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Wallet File Version = 169900
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Keys: 2001 plaintext, 0 encrypted, 2001 w/ metadata, 2001 total. Unknown wallet records: 0
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Wallet completed loading in              39ms
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z init message: Rescanning...
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Rescanning last 2 blocks (from block 2004265)...
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Rescan started from block 000000000000001f778a8e2b68cf05490ae000e653b925bb0552c1b79ef4fe70...
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] Rescan completed in               2ms
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] setKeyPool.size() = 2000
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] mapWallet.size() = 0
Feb 15 13:16:23 ip-172-31-82-15 bitcoind[41053]: 2022-02-15T04:16:23Z [wallet] m_address_book.size() = 0
```

When I list the wallets again:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "tutorial", "method": "listwallets", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":["wallet"],"error":null,"id":"tutorial"}
```

Now that the wallet has been loaded, we can get wallet info:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "tutorial", "method": "getwalletinfo", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/ | jq .
{
  "result": {
    "walletname": "wallet",
    "walletversion": 169900,
    "format": "bdb",
    "balance": 0,
    "unconfirmed_balance": 0,
    "immature_balance": 0,
    "txcount": 0,
    "keypoololdest": 1623329789,
    "keypoolsize": 1000,
    "hdseedid": "x",
    "keypoolsize_hd_internal": 1000,
    "paytxfee": 0,
    "private_keys_enabled": true,
    "avoid_reuse": false,
    "scanning": false,
    "descriptors": false
  },
  "error": null,
  "id": "tutorial"
}
```

Create another wallet, named `test-wallet`:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "createwallet", "params": ["test-wallet"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":{"name":"test-wallet","warning":""},"error":null,"id":"curltest"}
```

After we created our wallet, list the wallets again:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listwallets", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":["wallet","test-wallet"],"error":null,"id":"curltest"}
```

Now that we have 2 wallets, we need to specify the wallet name, when we want to do a `getwalletinfo` method for a specific wallet, `test-wallet` in this case:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getwalletinfo", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": {
    "walletname": "test-wallet",
    "walletversion": 169900,
    "format": "bdb",
    "balance": 0,
    "unconfirmed_balance": 0,
    "immature_balance": 0,
    "txcount": 0,
    "keypoololdest": 1623333495,
    "keypoolsize": 1000,
    "hdseedid": "x",
    "keypoolsize_hd_internal": 1000,
    "paytxfee": 0,
    "private_keys_enabled": true,
    "avoid_reuse": false,
    "scanning": false,
    "descriptors": false
  },
  "error": null,
  "id": "curltest"
}
```

Note: If the above command fails because of `jq` you can install jq or replace it with `python3 -m json.tool`.

In order to understand where the data resides for our `test-wallet`, we can use `find`:

```bash
$ find /blockchain/ -type f -name wallet.dat | grep test-wallet
/blockchain/bitcoin/data/testnet3/wallets/test-wallet/wallet.dat

$ find /blockchain/ -type f -name db.log | grep test-wallet
/blockchain/bitcoin/data/testnet3/wallets/test-wallet/db.log
```

Include the wallet name in the config located at `/etc/bitcoin/bitcoin.conf`:

```
[test]
wallet=wallet
wallet=test-wallet
```

Then restart bitcoind:

```bash
$ sudo systemctl restart bitcoind
```

Now list our wallets again, and you should see they are being read from config and the wallets will persist if your node restarts:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listwallets", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/
{"result":["wallet","test-wallet"],"error":null,"id":"curltest"}
```

To backup a wallet, the `test-wallet` in this case:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "backupwallet", "params": ["test-wallet_bak.dat"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":null,"error":null,"id":"curltest"}
````

To check where the file was backed up:

```bash
$ find /blockchain/ -name test-wallet_bak.dat
/blockchain/bitcoin/data/test-wallet_bak.dat
```

### Addresses

At this moment we have wallets, but we don't have any addresses associated to those wallets, we can verify this by listing wallet addresses using the `getaddressesbylabel` and passing a empty label as new addresses gets no labels assigned by default.

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": [""]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

Let's generate a new address for our `test-wallet`:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getnewaddress", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":"tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc","error":null,"id":"curltest"}
```

As you can see our address for `test-wallet` is `tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc`, note that you can have multiple addresses per wallet.

To get address information for the wallet by using the `getaddressinfo` method and passing the wallet address as the parameter:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressinfo", "params": ["tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": {
    "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
    "scriptPubKey": "x",
    "ismine": true,
    "solvable": true,
    "desc": "wpkh([05c34822/0'/0'/0']x)#x3x5vu3t",
    "iswatchonly": false,
    "isscript": false,
    "iswitness": true,
    "witness_version": 0,
    "witness_program": "1eb57750b62dxe284e32cd44f6e49",
    "pubkey": "023a1250c0d44751b604656x649357b5e530b9f8500f03ab5b",
    "ischange": false,
    "timestamp": 1623333494,
    "hdkeypath": "m/0'/0'/0'",
    "hdseedid": "x",
    "hdmasterfingerprint": "x",
    "labels": [
      ""
    ]
  },
  "error": null,
  "id": "curltest"
}
```

As before, we can view the address by label, to view the address for your wallet, we will now see our address:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": [""]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

Get available wallet balance with at least 6 confirmations:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getbalance", "params": ["*", 6]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":0.00000000,"error":null,"id":"curltest"}
```

Get balances (all balances) for the `test-wallet` wallet:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getbalances", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"mine":{"trusted":0.00000000,"untrusted_pending":0.00000000,"immature":0.00000000}},"error":null,"id":"curltest"}
```

Create a new address in our `test-wallet`:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getnewaddress", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":"tb1qa7e0mmgsul6pnxhzx7rw49y9qf35enqqra47hh","error":null,"id":"curltest"}
```

List all addresses for the wallet:

```bash
# by default new addresses has no labels, therefore it returns both
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": [""]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc":{"purpose":"receive"},"tb1qa7e0mmgsul6pnxhzx7rw49y9qf35enqqra47hh":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

### Labelling Addresses

Now we can label addresses on wallets, to label the first address as "green":

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "setlabel", "params": ["tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc", "green"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":null,"error":null,"id":"curltest"}
```

Label the new address as "blue":

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "setlabel", "params": ["tb1qa7e0mmgsul6pnxhzx7rw49y9qf35enqqra47hh", "blue"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":null,"error":null,"id":"curltest"}
```

Now we can list addresses for our wallet by the label, "blue" in this example:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": ["blue"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qa7e0mmgsul6pnxhzx7rw49y9qf35enqqra47hh":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

List addresses for our wallet by the label "green":

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": ["green"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

Create another address for our test-walet:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getnewaddress", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":"tb1qunk223dztk2j2zqswleyenwu3chfqt642vrp8z","error":null,"id":"curltest"}
```

Set the new address to the green label:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "setlabel", "params": ["tb1qunk223dztk2j2zqswleyenwu3chfqt642vrp8z", "green"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":null,"error":null,"id":"curltest"}
```

List addresses by green label:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getaddressesbylabel", "params": ["green"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc":{"purpose":"receive"},"tb1qunk223dztk2j2zqswleyenwu3chfqt642vrp8z":{"purpose":"receive"}},"error":null,"id":"curltest"}
```

### Receive tBTC

You can receive free test btc, by using any of these testnet faucet websites to receive tBTC over testnet:

- https://testnet-faucet.mempool.co/
- https://testnet.qc.to/
- https://onchain.io/bitcoin-testnet-faucet

The transaction details for sending 0.001 tbtc to my `tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc` address, we will receive the following information:

- TxID: `637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25`
- Destination Address: `tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc`
- Amount: `0.001`

Then to list transactions:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listtransactions", "params": ["*"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": [
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.001,
      "label": "green",
      "vout": 1,
      "confirmations": 0,
      "trusted": false,
      "txid": "637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25",
      "walletconflicts": [],
      "time": 1623337058,
      "timereceived": 1623337058,
      "bip125-replaceable": "no"
    }
  ],
  "error": null,
  "id": "curltest"
}
```

As you can see at the time, there were 0 confirmations, we can see the same txid as well as other info. With the testnet, we require at least 1 confirmation before a transaction is confirmed, where the mainnet requires 6.

To get balances for our test-wallet:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getbalances", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"mine":{"trusted":0.00000000,"untrusted_pending":0.00100000,"immature":0.00000000}},"error":null,"id":"curltest"}
```

As you can see as we don't have any confirmations yet, so therefore the trusted value is still 0.

Listing the transactions over time:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listtransactions", "params": ["*"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": [
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.001,
      "label": "green",
      "vout": 1,
      "confirmations": 0,
      "trusted": false,
      "txid": "637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25",
      "walletconflicts": [],
      "time": 1623337058,
      "timereceived": 1623337058,
      "bip125-replaceable": "no"
    }
  ],
  "error": null,
  "id": "curltest"
}
```

After a couple of minutes:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listtransactions", "params": ["*"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": [
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.001,
      "label": "green",
      "vout": 1,
      "confirmations": 2,
      "blockhash": "0000000000000000ba226ad21b51fe3998180dc354ec433ad7a4c4897e04d805",
      "blockheight": 2004280,
      "blockindex": 107,
      "blocktime": 1623337883,
      "txid": "637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25",
      "walletconflicts": [],
      "time": 1623337058,
      "timereceived": 1623337058,
      "bip125-replaceable": "no"
    }
  ],
  "error": null,
  "id": "curltest"
}
```

### Block Explorer

We can also use a blockchain explorer,  head over to a testnet blockchain explorer, such as:

* https://blockstream.info/testnet/tx

And provide the txid, in my case it was this one:

* https://blockstream.info/testnet/tx/637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25

This transaction was done a while ago, so the confirmations will be much more than from the output above, but you can see the confirmations, addresses involved and tbtc amount.

To only get the `trusted` balance, using the `getbalance` method and with at least 6 confirmations:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getbalance", "params": ["*", 6]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":0.00100000,"error":null,"id":"curltest"}
```

Let's send another transaction, the list the transactions using the `listtransactions` method:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listtransactions", "params": ["*"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": [
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.001,
      "label": "green",
      "vout": 1,
      "confirmations": 4,
      "blockhash": "0000000000000000ba226ad21b51fe3998180dc354ec433ad7a4c4897e04d805",
      "blockheight": 2004280,
      "blockindex": 107,
      "blocktime": 1623337883,
      "txid": "637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25",
      "walletconflicts": [],
      "time": 1623337058,
      "timereceived": 1623337058,
      "bip125-replaceable": "no"
    },
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.0111048,
      "label": "green",
      "vout": 0,
      "confirmations": 1,
      "blockhash": "000000000000002912e2da87e6e752c38965fc21e108aab439fcdcd82ba6e37a",
      "blockheight": 2004283,
      "blockindex": 4,
      "blocktime": 1623338496,
      "txid": "3cac023b088a2ddb2d601538edfc72cd1bff1bd2e1a1531518500c5b7a52e473",
      "walletconflicts": [],
      "time": 1623338453,
      "timereceived": 1623338453,
      "bip125-replaceable": "no"
    }
  ],
  "error": null,
  "id": "curltest"
}
```

After 12 hours, we can see that we have 102 confirmations for our first transaction and 99 transactions for the second transaction:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "listtransactions", "params": ["*"]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | jq .
{
  "result": [
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.001,
      "label": "green",
      "vout": 1,
      "confirmations": 102,
      "blockhash": "0000000000000000ba226ad21b51fe3998180dc354ec433ad7a4c4897e04d805",
      "blockheight": 2004280,
      "blockindex": 107,
      "blocktime": 1623337883,
      "txid": "637ea98aca23411059ad79aca7ea36ae30b68a173d89e6644703a06a1a846c25",
      "walletconflicts": [],
      "time": 1623337058,
      "timereceived": 1623337058,
      "bip125-replaceable": "no"
    },
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.0111048,
      "label": "green",
      "vout": 0,
      "confirmations": 99,
      "blockhash": "000000000000002912e2da87e6e752c38965fc21e108aab439fcdcd82ba6e37a",
      "blockheight": 2004283,
      "blockindex": 4,
      "blocktime": 1623338496,
      "txid": "3cac023b088a2ddb2d601538edfc72cd1bff1bd2e1a1531518500c5b7a52e473",
      "walletconflicts": [],
      "time": 1623338453,
      "timereceived": 1623338453,
      "bip125-replaceable": "no"
    },
    {
      "address": "tb1qr66hw59k958xrz008n679p8r9n2y7mjfr3tsjc",
      "category": "receive",
      "amount": 0.03521065,
      "label": "green",
      "vout": 5,
      "confirmations": 27,
      "blockhash": "000000000000004255a9d5af67b4649ff3f4d6a2f0c334261ca822cd9fbd00a9",
      "blockheight": 2004355,
      "blockindex": 43,
      "blocktime": 1623383177,
      "txid": "eb43868bd2c5abd97d4f5f11450952837bc3edc149478248e9453fdfb05c5187",
      "walletconflicts": [],
      "time": 1623382990,
      "timereceived": 1623382990,
      "bip125-replaceable": "no"
    }
  ],
  "error": null,
  "id": "curltest"
}
```

After 3 transactions, view the balance in the test wallet using the `getbalance` method:

```bash
$ curl -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curltest", "method": "getbalance", "params": ["*", 6]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":0.04731545,"error":null,"id":"curltest"}
```

### Sending a raw transaction

A easier way to send a transaction is by using `sendtoaddress` and the source wallet will be in the request url, ie: `/wallet/wallet-name`

First we look if we have a address for our account that we are sending from, if not then we can create a wallet and a address, for this example, I have a address for the `wallet` wallet:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getaddressesbylabel", "params": [""]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/wallet
{"result":{"tb1qks4tyrz52vvdh35kcx0ypvnj3fjdkl692pzfyc":{"purpose":"receive"}},"error":null,"id":"curl"}
```

Ensure that our source wallet has funds in it:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getbalance", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/wallet | python -m json.tool
{
    "error": null,
    "id": "curl",
    "result": 0.01811929
}
```

We have enough funds to send, so we now have the source wallet name and we need to get the wallet address, where we want to send the funds to, which in this case is the address in `test-wallet` as the destination:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getaddressesbylabel", "params": [""]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet
{"result":{"tb1qzxmefmcpq98z42v67a80gvug2fe979r5h768yv":{"purpose":"receive"}},"error":null,"id":"curl"}
```

Just to double check our current funds in the wallet that will receive funds:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getbalance", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/test-wallet | python -m json.tool
{
    "error": null,
    "id": "curl",
    "result": 0.35572584
}
```

Now we will use the `sendtoaddress` method, with the recipient address and the amount to send as the parameters. To summarize:

- Source Wallet: `wallet`
- Destination Address:  `tb1qzxmefmcpq98z42v67a80gvug2fe979r5h768yv`
- Amount to send: `0.01.`

Sending the amount:

```bash
$ curl -s -u "$bitcoinauth"  -d '{"jsonrpc": "1.0", "id":"0", "method": "sendtoaddress", "params":["tb1qzxmefmcpq98z42v67a80gvug2fe979r5h768yv", 0.01]}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/wallet
{"result":"df087095ac79d678f9d98c8bf8ebface2ac62a20546d85e07a852feb2c3bea50","error":null,"id":"0"}
```

We will receive a transaction id, and if we list for transactions for our source wallet, we will see the transaction:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "listtransactions", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/wallet | python -m json.tool
{
    "error": null,
    "id": "curl",
    "result": [
        {
            "address": "tb1qks4tyrz52vvdh35kcx0ypvnj3fjdkl692pzfyc",
            "amount": 0.01811929,
            "bip125-replaceable": "no",
            "blockhash": "000000000000003c0ac4978ba815ff8f0d7f55da98923c686118c75461fb579e",
            "blockheight": 2091275,
            "blockindex": 1,
            "blocktime": 1630677226,
            "category": "receive",
            "confirmations": 1,
            "label": "",
            "time": 1630677177,
            "timereceived": 1630677177,
            "txid": "e44b45a309284e8044a15dca8c0a895a5c7072741882281038fb185cc0c1a0d9",
            "vout": 0,
            "walletconflicts": []
        },
        {
            "abandoned": false,
            "address": "tb1qzxmefmcpq98z42v67a80gvug2fe979r5h768yv",
            "amount": -0.01,
            "bip125-replaceable": "no",
            "category": "send",
            "confirmations": 0,
            "fee": -1.41e-06,
            "time": 1630677743,
            "timereceived": 1630677743,
            "trusted": true,
            "txid": "df087095ac79d678f9d98c8bf8ebface2ac62a20546d85e07a852feb2c3bea50",
            "vout": 0,
            "walletconflicts": []
        }
    ]
}
```

So when we look at the sender wallet, we will see the funds was deducted:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getbalance", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/wallet | python -m json.tool
{
    "error": null,
    "id": "curl",
    "result": 0.00811788
}
```

And when we look at the receiver wallet, we can see that the account was received:

```bash
$ curl -s -u "$bitcoinauth" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getbalance", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/wallet/rpi01-main | python -m json.tool
{
    "error": null,
    "id": "curl",
    "result": 0.36572584
}
```

## Thank You

Thanks for reading, if you like my content, check out my **[website](https://ruan.dev)** or follow me at **[@ruanbekker](https://twitter.com/ruanbekker)** on Twitter.

The source code of this blog post will be added to this Github Repository:
- https://github.com/ruanbekker/bitcoin-full-node-demo

