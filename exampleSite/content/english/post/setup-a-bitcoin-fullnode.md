+++
author = "Ruan Bekker"
title = "Setup a Bitcoin Fullnode"
date = "2021-02-12"
description = "This tutorial will demonstrate how to setup a bitcoin fullnode to interact with the blockchain."
tags = [
    "blockchain",
    "crypto",
    "bitcoin",
    "fullnode",
]
categories = [
    "bitcoin",
]
series = ["blockchain-introduction"]
+++

![bitcoin](https://user-images.githubusercontent.com/567298/153725244-d93accc6-0a32-460e-8d0f-9331276bf4b8.png)

In this tutorial we will install and configure a bitcoin fullnode. From our previous post, we discussed what a fullnode is, but to recap:

> A full node is a node which downloads every block and transaction and check them against bitcoin's consensus rules. which fully validates transactions and blocks. Almost all full nodes also help the network by accepting transactions and blocks from other full nodes, validating those transactions and blocks, and then relaying them to further full nodes.

## About

This post will be referenced for my future posts, where we will use our fullnode to host our bitcoin wallet, where we will create bitcoin addresses, send testnet bitcoin funds to our wallet, query transactions on the blockchain etc. 

## Installation

First we would need to create a dedicated user for running our bitcoin software:

```bash
$ sudo groupadd -r bitcoin
$ sudo useradd -r -m -g bitcoin -s /bin/bash bitcoin
```

Update the package manager and install the dependencies:

```bash
$ sudo apt update && sudo apt install ca-certificates gnupg gpg wget jq --no-install-recommends -y
```

Get the latest release of bitcoin-core from their [website](https://bitcoincore.org/bin/) then download bitcoin-core and verify that the package matches the sha hash:

```bash
$ cd /tmp
$ wget https://bitcoincore.org/bin/bitcoin-core-0.21.2/SHA256SUMS.asc
$ wget https://bitcoincore.org/bin/bitcoin-core-0.21.2/bitcoin-0.21.2-x86_64-linux-gnu.tar.gz
$ cat SHA256SUMS.asc | grep bitcoin-0.21.2-x86_64-linux-gnu.tar.gz | awk '{ print $1 }'
```

Extract the tarball:

```bash
$ tar -xvf bitcoin-0.21.2-x86_64-linux-gnu.tar.gz
```

Create the directories that will be used by bitcoin:

```bash
$ sudo mkdir -p /blockchain/bitcoin/{data,scripts}
$ sudo mkdir -p /usr/local/bitcoin/0.21.2/bin
$ sudo mkdir -p /etc/bitcoin
```

Move the binaries to the created directories:

```bash
$ sudo mv bitcoin-0.21.2/bin/bitcoin* /usr/local/bitcoin/0.21.2/bin/
```

Then clean up the downloaded data:

```bash
$ sudo rm -rf bitcoin-0.21.2
$ sudo rm -rf bitcoin-0.21.2-x86_64-linux-gnu.tar.gz
```

Then create a symlink to `current` as that will be referenced from systemd:

```bash
$ sudo ln -s /usr/local/bitcoin/0.21.2 /usr/local/bitcoin/current
```

When you have to upgrade the software version of bitcoin-core in the future you can remove the symlink with `sudo rm -rf /usr/local/bitcoin/current` and symlink to the newer version as shown above.

## Configuration

Create the bitcoin configuration, here you see I am using the **testnet** chain and due to storage restrictions for my use-case I am setting **pruning** mode to 2GB, and if you don't set `BITCOIN_RPC_USER` it will use the user bitcoin and if you don't set `BITCOIN_RPC_PASSWORD` it will generate a password for the json-rpc interface: 

```bash
$ cat > bitcoin.conf << EOF
datadir=/blockchain/bitcoin/data
printtoconsole=1
onlynet=ipv4
rpcallowip=127.0.0.1
rpcuser=${BITCOIN_RPC_USER:-bitcoin}
rpcpassword=${BITCOIN_RPC_PASSWORD:-$(openssl rand -hex 24)}
testnet=1
prune=2000
walletnotify=/blockchain/bitcoin/scripts/walletnotify.sh %s %w
blocknotify=/blockchain/bitcoin/scripts/blocknotify.sh %s
[test]
rpcbind=127.0.0.1
rpcport=18332
EOF
```

Then move the `bitcoin.conf` to `/etc/bitcoin/bitcoin.conf`:

```bash
$ sudo mv bitcoin.conf /etc/bitcoin/bitcoin.conf
```

As you can see in the configuration we have 2 notifiers:

- `walletnotify` - triggers the script whenever we have a transaction event to or from the wallet
- `blocknotify` - triggers the script whenever we have a new block

Create the wallet notify script in `/blockchain/bitcoin/scripts/walletnotify.sh` with the content of:

```bash
#!/usr/bin/env bash
echo "[$(date +%FT%T)] type:walletnotify $1 $2" >> /var/log/wallet-notify.log
```

Create the block notify script in `/blockchain/bitcoin/scripts/blocknotify.sh` with the content of:

```bash
#!/usr/bin/env bash
echo "[$(date +%FT%T)] type:blocknotify $1" >> /var/log/wallet-notify.log
```

Create the file where the wallet and block notify data will be written to:

```bash
$ sudo touch /var/log/wallet-notify.log
```

Change the permissions of the file so that the crypto user can write to it:

```bash
$ sudo chown bitcoin:bitcoin /var/log/wallet-notify.log
```

Make the scripts executable:

```bash
$ sudo chmod +x /blockchain/bitcoin/scripts/*notify.sh
```

Create the systemd unit-file for bitcoind in `/etc/systemd/system/bitcoind.service`:

```bash
[Unit]
Description=Bitcoin Core Testnet
After=network.target

[Service]
User=bitcoin
Group=bitcoin
WorkingDirectory=/blockchain/bitcoin/data
Type=simple
ExecStart=/usr/local/bitcoin/current/bin/bitcoind -conf=/etc/bitcoin/bitcoin.conf

[Install]
WantedBy=multi-user.target
```

Update the `PATH` variable to include bitcoin binaries in `/etc/profile.d/bitcoind.sh` with the following content:

```bash
$ export PATH=$PATH:/usr/local/bitcoin/current/bin
```

You can update your current session by running:

```bash
$ export PATH=$PATH:/usr/local/bitcoin/current/bin
```

Ensure that permissions are in place so that the user we created has access to the directories we created:

```bash
$ sudo chown -R bitcoin:bitcoin /blockchain /usr/local/bitcoin /etc/bitcoin
```

Reload systemd:

```bash
$ sudo systemctl daemon-reload
```

## Initial Block Download

When we start bitcoind for the first time, the node needs to download all the blocks, which can take some time:

```bash
$ sudo systemctl restart bitcoind
```

As this might take some time, but you can follow the progress by doing:

```
$ sudo journalctl -fu bitcoind
```

A fully sync node should look like this, where you can see `progress=1.000000`:

```
Feb 10 13:39:58 node-01 bitcoind[532]: 2022-02-10T13:39:58Z UpdateTip: new best=000000000000003b06a186bbd79909e1a338196b5cffcee1979d6c4fc90f67a9 height=2097621 version=0x20a00000 log2_work=74.510314 tx=61253790 date='2021-10-04T11:39:54Z' progress=1.000000 cache=0.4MiB(1477txo)
```

## Test

You can retrieve the rpc password from the config in `cat /etc/bitcoin/bitcoin.conf`, then you can use the json rpc to call the `getblockchaininfo` method:

```
$ curl -s -u "bitcoin:<enter-bitcoin-rpcpassword-here>" -d '{"jsonrpc": "1.0", "id": "curl", "method": "getblockchaininfo", "params": []}' -H 'content-type: text/plain;' http://127.0.0.1:18332/  | python3 -m json.tool
```

Once you get a response from your node, you are good to go. In the future posts, we will go more extensively into the json rpc to interact with our node, send transactions, receive testnet bitcoin etc.

## Thank You

Thanks for reading, if you like my content, check out my **[website](https://ruan.dev)** or follow me at **[@ruanbekker](https://twitter.com/ruanbekker)** on Twitter.
