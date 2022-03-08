+++
author = "Ruan Bekker"
title = "Setup a Monero Testnet Fullnode"
date = "2022-03-02"
description = "This tutorial will demonstrate how to setup a monero fullnode on the testnet chain to interact with the blockchain."
tags = [
    "blockchain",
    "crypto",
    "monero",
    "fullnode",
]
categories = [
    "monero",
]
series = ["blockchain-introduction"]
+++

![monero-image](https://user-images.githubusercontent.com/567298/156387670-b6524f09-4851-4a57-be9c-6c8ef7a2eabc.png)

In this tutorial we will setup a monero fullnode on the testnet chain that consists of two components, the monerod and monero-wallet-rpc.

## About Monero

Monero (XMR), is a decentralized cryptocurrency. It uses a public distributed ledger with privacy enhancing technologies that obfuscate transactions to achieve anonymity and fungibility. More information can be found from [this wikipedia page](https://en.wikipedia.org/wiki/Monero).

## Monero Components

A fullnode monero setup consists of two components:

1. monerod
2. monero-wallet-rpc

The monerod process is responsible for the block data and exposes a [daemon-rpc](https://www.getmonero.org/resources/developer-guides/daemon-rpc.html) with authentication which will require any client to authenticate against any node rpc calls.

The monero-wallet-rpc is responsible for exposing a [wallet-rpc](https://www.getmonero.org/resources/developer-guides/wallet-rpc.html) on top of the wallet, to interact with the wallet. The node where this process runs has the wallet file on disk and has the password to unlock the wallet. The monero-wallet-rpc connects to the monerod daemon, and specifies the authentication to communicate with the node rpc.

This Architectural Diagram explains a bit more from what I mentioned:

![image](https://user-images.githubusercontent.com/567298/156521149-a653161c-1800-48f2-804f-0140448767fc.png)

## Ports

The following ports will be referenced:

| Port  | Binary            | Chain   | Description              |
| :---- | :---------------: | :----:  | -----------------------: |
| 28080 | monerod           | testnet | P2P Port                 |
| 28081 | monerod           | testnet | RPC Server               |
| 28082 | monerod           | testnet | ZMQ Server               |
| 28089 | monerod           | testnet | Restricted RPC Server    |
| 5001  | monero-wallet-rpc | testnet | Wallet RPC               |

## Preparing our Monerod Node

For our first node, monerod, I have a extra disk `/dev/xvdb` where we will store our blockchain data. So before we can use it we will first create our directories, create the filesystem and mount it:

```bash
$ sudo mkdir /blockchain
$ sudo mkfs.xfs /dev/xvdb
$ sudo mount /dev/xvdb /blockchain
```

Now we will create our dedicated user and group: `monero`:

```bash
$ sudo adduser --system --group --no-create-home monero
```

Create the directories that we will use for monerod:

```bash
$ sudo mkdir -p /blockchain/monero/{config,data,logs}
```

## Installation of Monerod

Get the monero version of your choice and download the tarball:

```bash
$ cd /tmp
$ wget --content-disposition https://downloads.getmonero.org/cli/linux64
$ wget https://www.getmonero.org/downloads/hashes.txt
$ grep -e $(sha256sum monero-linux-x64-*.tar.bz2) hashes.txt
$ tar -xvf monero-linux-x64-*.tar.bz2
```

Once the binaries are extracted, move it into your path. For monerod, you probably only want the monerod binary, but I will just move all of them:

```bash
$ sudo mv monero-x86_64-linux-gnu-*/monero* /usr/local/bin/
```

Cleanup the downloaded files:

```bash
$ rm monero-linux-x64-*.tar.bz2 hashes.txt
$ rm -rf monero-x86_64-linux-gnu-*/
```

## Configuration of Monerod

First we will create the configuration file for monerod at `/blockchain/monero/config/monerod.conf`:

```
# blockchain data / log locations
data-dir=/blockchain/monero/data
log-file=/blockchain/monero/logs/monero.log

# log options
log-level=0
max-log-file-size=0 

# testnet
testnet=1

# p2p fullnode
p2p-bind-ip=0.0.0.0
p2p-bind-port=28080
public-node=false # Dont advertise the rpc-restricted port over p2p peer lists

# rpc settings
rpc-bind-ip=127.0.0.1
rpc-bind-port=28081
rpc-restricted-bind-ip=0.0.0.0
rpc-restricted-bind-port=28089
rpc-login=monero:moneropass

# zmq settings
zmq-rpc-bind-ip=127.0.0.1
zmq-rpc-bind-port=28082

# node settings
prune-blockchain=true
db-sync-mode=safe # reliable db mode, but slower performance
enforce-dns-checkpointing=true
enable-dns-blocklist=true
no-igd=true
no-zmq=false

# network throughput settings
out-peers=32 
in-peers=32
limit-rate-up=1048576 # Contribute more to p2p network, increased from 2Mb/s to 1GB/s
limit-rate-down=1048576 # increased from 8Mb/s to 1GB/s
```

Then we need the systemd unit file for monerod at `/etc/systemd/system/monerod.service`:

```
[Unit]
Description=monerod
After=network.target

[Service]
Type=forking
PIDFile=/blockchain/monero/config/monerod.pid
ExecStart=/usr/local/bin/monerod \
  --config-file /blockchain/monero/config/monerod.conf \
  --pidfile /blockchain/monero/config/monerod.pid \
  --detach
User=monero
Group=monero
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Change the ownerships to ensure that the monero user has access to the mentioned files and directories:

```bash
$ sudo chown -R monero:monero /usr/local/bin/monero*
$ sudo chown -R monero:monero /blockchain
```

## Start Monerod

Since we created a new systemd unit file, we need to reload systemd:

```bash
$ sudo systemctl daemon-reload
```

Then we enable monerod to start on boot:

```bash
$ sudo systemctl enable monerod
```

Then we can finally start monerod:

```bash
$ sudo systemctl restart monerod
```

Now we will wait for the blocks to sync until completion:

```bash
$ tail -f /blockchain/monero/logs/monero.log
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	**********************************************************************
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	The daemon will start synchronizing with the network. This may take a long time to complete.
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	You can set the level of process detailization through "set_log <level|categories>" command,
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	where <level> is between 0 (no details) and 4 (very verbose), or custom category based levels (eg, *:WARNING).
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	Use the "help" command to see the list of available commands.
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	Use "help <command>" to see a command's documentation.
2022-02-21 11:01:10.601	[P2P2]	INFO	global	src/cryptonote_core/cryptonote_core.cpp:1742	**********************************************************************
2022-02-21 11:01:52.247	[P2P5]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:414	[91.138.12.31:28080 OUT] Sync data returned a new top block candidate: 1 -> 1922862 [Your node is 1922861 blocks (6.1 years) behind]
2022-02-21 11:01:52.248	[P2P5]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:414	SYNCHRONIZATION started
2022-02-21 11:01:55.583	[P2P5]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 101/1922862 (0%, 1922761 left)
2022-02-21 11:01:57.427	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 201/1922862 (0%, 1922661 left)
2022-02-21 11:01:58.963	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 301/1922862 (0%, 1922561 left)
2022-02-21 11:02:00.836	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 401/1922862 (0%, 1922461 left)
2022-02-21 11:02:02.554	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 501/1922862 (0%, 1922361 left)
2022-02-21 11:02:04.184	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 601/1922862 (0%, 1922261 left)
2022-02-21 11:02:05.759	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:1681	Synced 701/1922862 (0%, 1922161 left)
...
2022-02-28 09:38:06.741	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:2465	**********************************************************************
2022-02-28 09:38:06.741	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:2465	You are now synchronized with the network. You may now start monero-wallet-cli.
2022-02-28 09:38:06.741	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:2465
2022-02-28 09:38:06.741	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:2465	Use the "help" command to see the list of available commands.
2022-02-28 09:38:06.741	[P2P4]	INFO	global	src/cryptonote_protocol/cryptonote_protocol_handler.inl:2465	**********************************************************************
```

The following ports should be listening:

```bash
$ sudo netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      684/sshd: /usr/sbin
tcp        0      0 0.0.0.0:28089           0.0.0.0:*               LISTEN      13412/monerod
tcp        0      0 0.0.0.0:28080           0.0.0.0:*               LISTEN      13412/monerod
tcp        0      0 127.0.0.1:28081         0.0.0.0:*               LISTEN      13412/monerod
```

Now that monerod has been fully synced, we can proceed with our monero-wallet-rpc node.

## Preparing our Monero Wallet RPC Node

Create the directory where we will store our wallet files:

```bash
$ sudo mkdir /blockchain
```

Now we will create our dedicated user and group: `monero`:

```bash
$ sudo adduser --system --group --no-create-home monero
```

Create the directories that we will use for monero-wallet-rpc:

```bash
$ sudo mkdir -p /blockchain/monero/{config,data,logs}
```

## Installation of Monero Wallet RPC

Get the monero version of your choice and download the tarball:

```bash
$ cd /tmp
$ wget --content-disposition https://downloads.getmonero.org/cli/linux64
$ wget https://www.getmonero.org/downloads/hashes.txt
$ grep -e $(sha256sum monero-linux-x64-*.tar.bz2) hashes.txt
$ tar -xvf monero-linux-x64-*.tar.bz2
```

Once the binaries are extracted, move it into your path. For monero-wallet-rpc, you probably only want the monero-wallet-rpc and monero-wallet-cli binaries, but I will just move all of them:

```bash
$ sudo mv monero-x86_64-linux-gnu-*/monero* /usr/local/bin/
```

Cleanup the downloaded files:

```bash
$ rm monero-linux-x64-*.tar.bz2 hashes.txt
$ rm -rf monero-x86_64-linux-gnu-*/
```

## Create the Monero Wallet

Using the monero-wallet-cli, we will create our wallet then add a layer of security by adding a password to the wallet. But first we need to create a strong password:

```bash
$ openssl rand -base64 18
mXjlNt5obHJ2bcQEbaE9QoXo
```

Now create the wallet file using the monero-wallet-cli:

```bash
$ $ monero-wallet-cli --testnet --generate-new-wallet /blockchain/monero/data/wallets/monero-main.wallet --daemon-address 127.0.0.1:28089 --daemon-login monero:moneropass

This is the command line monero wallet. It needs to connect to a monero
daemon to work correctly.

Monero 'Oxygen Orion' (v0.17.3.0-release)
Logging to monero-wallet-cli.log
Enter a new password for the wallet:
Confirm password:
List of available languages for your wallet's seed:
If your display freezes, exit blind with ^C, then run again with --use-english-language-names
1 : English
Enter the number corresponding to the language of your choice: 1
Generated new wallet: 9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdxxxxxxxxxxxxxxxxxxxxRBNdvqmwMqX6Fgt5qtTRcpZ6ys
View key: 1b2dbdb6d7c8859bfb891c1e357187dfxxxxxxxxxxxxxxxxxxxx481fb3b59b0e
**********************************************************************

Your wallet has been generated!
To start synchronizing with the daemon, use the "refresh" command.
Use the "help" command to see a simplified list of available commands.
Use "help all" command to see the list of all available commands.
Use "help <command>" to see a command's documentation.
Always use the "exit" command when closing monero-wallet-cli to save
your current session's state. Otherwise, you might need to synchronize
your wallet again (your wallet keys are NOT at risk in any case).

NOTE: the following 25 words can be used to recover access to your wallet. Write them down and store them somewhere safe and secure. Please do not store them in your email or on file storage services outside of your immediate control.

broken inflamed xxxxx films rhythm silver digit uphill
bamboo doing vitals baptism xxxxxx bamboo maul ardent
inquest cactus xxxxx video often swung xxxxxx identity broken
**********************************************************************

The daemon is not set up to background mine.
With background mining enabled, the daemon will mine when idle and not on battery.
Enabling this supports the network you are using, and makes you eligible for receiving new monero
Do you want to do it now? (Y/Yes/N/No): : Y
If you are new to Monero, type "welcome" for a brief overview.
Starting refresh...
Refresh done, blocks received: 0
Untagged accounts:
          Account               Balance      Unlocked balance                 Label
 *       0 9yFRUL        0.000000000000        0.000000000000       Primary account
----------------------------------------------------------------------------------
          Total        0.000000000000        0.000000000000
Currently selected account: [0] Primary account
Tag: (No tag assigned)
Balance: 0.000000000000, unlocked balance: 0.000000000000
Background refresh thread started
[wallet 9yFRUL]: status
Refreshed 1927853/1927853, synced, daemon RPC v3.9, SSL
[wallet 9yFRUL]: exit
```

As you can see we did the following:

- generated the wallet file and specified where the wallet file will be stored
- specified the monerod daemon address as well as the password for the daemon
- provided the password we want to use with our wallet
- received 25 words that we can use to recover our wallet (save this in a secure place)
- then we were dropped into a wallet shell and used the command `status` to verify if the node was synced

## Connecting to the Wallet with CLI

In order to connect to the wallet using the monero-wallet-cli on the node, you can store the wallet password in a file, such as this:

```bash
$ cat /blockchain/monero/data/.password
mXjlNt5obHJ2bcQEbaE9QoXo
```

Then we can connect to the wallet using:

```bash
$ sudo monero-wallet-cli --testnet --wallet-file /blockchain/monero/data/wallets/monero-main.wallet --daemon-address 127.0.0.1:28089 --daemon-login monero:moneropass --password-file /blockchain/monero/data/.password

Monero 'Oxygen Orion' (v0.17.3.0-release)
Logging to monero-wallet-cli.log
Opened wallet: 9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjxxxxxxxxxxxxxxxxxxxxawvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys
**********************************************************************
Use the "help" command to see a simplified list of available commands.
**********************************************************************
Starting refresh...
Refresh done, blocks received: 23
Untagged accounts:
          Account               Balance      Unlocked balance                 Label
 *       0 9yFRUL        0.000000000000        0.000000000000       Primary account
----------------------------------------------------------------------------------
          Total        0.000000000000        0.000000000000
Currently selected account: [0] Primary account
Tag: (No tag assigned)
Balance: 0.000000000000, unlocked balance: 0.000000000000
Background refresh thread started
[wallet 9yFRUL]:
```

## Configuration of the Monero Wallet RPC Server

The systemd unit file at `/etc/systemd/system/monero-wallet-rpc.service`:

```
[Unit]
Description=monero-wallet Service
After=network.target

[Service]
User=monero
Group=monero
Type=simple
ExecStart=/usr/local/bin/monero-wallet-rpc \
  --testnet \
  --password mXjlNt5obHJ2bcQEbaE9QoXo \
  --log-file /blockchain/monero/logs/monero-wallet-rpc.log \
  --max-log-files 2 \
  --log-level 3 \
  --daemon-address http://127.0.0.1:28089 \
  --rpc-bind-ip 0.0.0.0 \
  --rpc-bind-port 5001 \
  --trusted-daemon \
  --non-interactive \
  --daemon-login monero:moneropass \
  --rpc-login monerowallet:moneropass \
  --confirm-external-bind \
  --shared-ringdb-dir /blockchain/monero/data/testnet/.shared-ringdb/testnet \
  --wallet-file /blockchain/monero/data/wallets/monero-main.wallet

[Install]
WantedBy=multi-user.target
```

In this case:

- `rpc-bind-ip` and `rpc-bind-port` will be the listening port for the rpc server
- `rpc-login` will be the credentials that the client will use for the rpc server
- `daemon-address` is the address for the `rpc-restricted` endpoint
- `daemon-login` is the auth configured for the monerod process

Create the directories for the shared-ringdb : 

```bash
sudo mkdir -p /blockchain/monero/data/testnet/.shared-ringdb/testnet
sudo chown -R monero:monero /blockchain/monero
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Restart the monero-wallet-rpc service:

```bash
$ sudo systemctl restart monero-wallet-rpc
$ sudo systemctl status monero-wallet-rpc

 monero-wallet-rpc.service - monero-wallet Service
     Loaded: loaded (/etc/systemd/system/monero-wallet-rpc.service; disabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-02-28 14:32:39 UTC; 37min ago
   Main PID: 62746 (monero-wallet-r)
      Tasks: 3 (limit: 4693)
     Memory: 36.8M
     CGroup: /system.slice/monero-wallet-rpc.service
             └─62746 /usr/local/bin/monero-wallet-rpc --testnet --password mXjlNt5obHJ2bcQEbaE9QoXo --log-file /blockchain/monero/logs/monero-wallet-rpc.log --max-log-files 2 --log-level 3 --da
```

The following ports should be listening for the wallet-rpc-server:

```bash
$ sudo netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:5001            0.0.0.0:*               LISTEN      62746/monero-wallet
```

## Configuration of the Reverse Proxy

The reason I'm including the reverse proxy will be for a AWS ALB to use a health check endpoint, against the monero-wallet-rpc server. This nginx installation will be done on the monero-wallet-rpc server.

Install nginx:

```bash
sudo apt install nginx -y
```

The main config, `/etc/nginx/nginx.conf`:

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
}

http {
	# Basic Settings
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	# SSL Settings
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	# Logging Settings
	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	# Gzip Settings
	gzip on;

	# Virtual Host Configs
	include /etc/nginx/conf.d/monero-wallet-rpc.conf;
}
```

The digest authentication requires keepalives, so you will notice that we are using a upstream where we can specify our keepalives. The virtual host config `/etc/nginx/conf.d/monero-wallet-rpc.conf`:

```
upstream monerowallet {
  server 127.0.0.1:5001;
  keepalive 64;
}

server {
    listen 80;
    server_name _;
    access_log /var/log/nginx/monero_access.log;
    error_log /var/log/nginx/monero_error.log notice;

    location /json_rpc {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Connection "";
        proxy_pass http://monerowallet;
        proxy_http_version 1.1;
        proxy_read_timeout 600s;
    }

    location /health {
      return 200 'up\n';
    }
}
```

Restart nginx:

```bash
sudo systemctl restart nginx
```

## Test via ALB

If you did not use a Target Group to associate the EC2 Instance as a target, you can also test against the reachable IP address of the monero-rpc-wallet server. In my case I have SSL termination on the ALB so I will be using https, but if you are testing against the Nginx IP, you can use `http://ip-of-monero-wallet-rpc:80/json_rpc`.

```bash
curl -XPOST -u "monerowallet:moneropass" --digest -H 'Content-Type: application/json' https://monero-wallet-rpc.mydomain.com/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_balance", "params":{"account_index":0,"address_indices":[1,1]}}'
```

The output of that will be something like:

```json
{
  "id": "0",
  "jsonrpc": "2.0",
  "result": {
    "balance": 3000000000000,
    "blocks_to_unlock": 0,
    "multisig_import_needed": false,
    "per_subaddress": [{
      "account_index": 0,
      "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
      "address_index": 1,
      "balance": 0,
      "blocks_to_unlock": 0,
      "label": "main",
      "num_unspent_outputs": 0,
      "time_to_unlock": 0,
      "unlocked_balance": 0
    }],
    "time_to_unlock": 0,
    "unlocked_balance": 0
  }
}
```

## API Examples

Below we will have some API Examples of:
- [Node RPC](https://www.getmonero.org/resources/developer-guides/daemon-rpc.html)
- [Wallet RPC](https://www.getmonero.org/resources/developer-guides/wallet-rpc.html)

On the Node RPC calls, we will need to communicate with the monerod server's ip on port 28089, you will notice that I am calling the endpoint from the node. We will be using the [get_info](https://www.getmonero.org/resources/developer-guides/daemon-rpc.html#get_info) method:

```bash
$ curl -u "monero:moneropass" --digest -H 'Content-Type: application/json' http://127.0.0.1:28089/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_info"}'
{
  "id": "0",
  "jsonrpc": "2.0",
  "result": {
    "adjusted_time": 1646060226,
    "alt_blocks_count": 0,
    "block_size_limit": 600000,
    "block_size_median": 300000,
    "block_weight_limit": 600000,
    "block_weight_median": 300000,
    "bootstrap_daemon_address": "",
    "busy_syncing": false,
    "credits": 0,
    "cumulative_difficulty": 854645419232,
    "cumulative_difficulty_top64": 0,
    "database_size": 5368709120,
    "difficulty": 191892,
    "difficulty_top64": 0,
    "free_space": 18446744073709551615,
    "grey_peerlist_size": 0,
    "height": 1928003,
    "height_without_bootstrap": 0,
    "incoming_connections_count": 0,
    "mainnet": false,
    "nettype": "testnet",
    "offline": false,
    "outgoing_connections_count": 0,
    "rpc_connections_count": 0,
    "stagenet": false,
    "start_time": 0,
    "status": "OK",
    "synchronized": true,
    "target": 120,
    "target_height": 0,
    "testnet": true,
    "top_block_hash": "b4ded69fca1595177a1f2ca1b2670f437a9d0b546486f4aad0fc0b4ab95dcf3f",
    "top_hash": "",
    "tx_count": 884269,
    "tx_pool_size": 0,
    "untrusted": false,
    "update_available": false,
    "version": "",
    "was_bootstrap_ever_used": false,
    "white_peerlist_size": 0,
    "wide_cumulative_difficulty": "0xc6fcd62ce0",
    "wide_difficulty": "0x2ed94"
  }
}
```

For the Wallet RPC we will communicate with the monero-wallet-rpc server, you will notice that I will be calling the endpoints from the node, and the first method that we will call is the [get_balance](https://www.getmonero.org/resources/developer-guides/wallet-rpc.html#get_balance) method:

```bash
$ curl -u "monerowallet:moneropass" --digest -H 'Content-Type: application/json' http://127.0.0.1:5001/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_balance", "params":{"account_index":0,"address_indices":[0,1]}}'
{
  "id": "0",
  "jsonrpc": "2.0",
  "result": {
    "balance": 0,
    "blocks_to_unlock": 0,
    "multisig_import_needed": false,
    "per_subaddress": [{
      "account_index": 0,
      "address": "9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdZHNr9s8awvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys",
      "address_index": 0,
      "balance": 0,
      "blocks_to_unlock": 0,
      "label": "Primary account",
      "num_unspent_outputs": 0,
      "time_to_unlock": 0,
      "unlocked_balance": 0
    },{
      "account_index": 0,
      "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
      "address_index": 1,
      "balance": 0,
      "blocks_to_unlock": 0,
      "label": "",
      "num_unspent_outputs": 0,
      "time_to_unlock": 0,
      "unlocked_balance": 0
    }],
    "time_to_unlock": 0,
    "unlocked_balance": 0
  }
}
```

To create a address we will be using the [create_address](https://www.getmonero.org/resources/developer-guides/wallet-rpc.html#create_address) method:

```bash
$ curl -u "monerowallet:moneropass" --digest -H 'Content-Type: application/json' http://127.0.0.1:5001/json_rpc -d '{"jsonrpc": "2.0"," id": "0", "method": "create_address", "params": {"account_index": 0, "label": "main"}}'
{
  "id": 0,
  "jsonrpc": "2.0",
  "result": {
    "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
    "address_index": 1,
    "address_indices": [1],
    "addresses": ["BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz"]
  }
}
```

To view addresses we can use the [get_address](https://www.getmonero.org/resources/developer-guides/wallet-rpc.html#get_address) method:

```bash
$ curl -u "monerowallet:moneropass" --digest -H 'Content-Type: application/json' http://127.0.0.1:5001/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_address", "params":{"account_index":0,"address_index":[0,1,1]}}'
{
  "id": "0",
  "jsonrpc": "2.0",
  "result": {
    "address": "9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdZHNr9s8awvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys",
    "addresses": [{
      "address": "9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdZHNr9s8awvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys",
      "address_index": 0,
      "label": "Primary account",
      "used": false
    },{
      "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
      "address_index": 1,
      "label": "main",
      "used": false
    },{
      "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
      "address_index": 1,
      "label": "main",
      "used": false
    }]
  }
}
```

## Monero Testnet Faucet

In order to get some testnet XMR funds, you can use the following faucet:
- https://community.rino.io/faucet/testnet/

I've used this faucet to send 1 XMR to `9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdZHNr9s8awvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys` and received the the txid of `c37e5e93bc9f9d5a7debd935af6e01080123cd32d47cc733d71f4329ef2e1b83` which we can track on:
- https://community.rino.io/explorer/testnet/tx/c37e5e93bc9f9d5a7debd935af6e01080123cd32d47cc733d71f4329ef2e1b83

After we received enough confirmations, we can verify the balance:

```bash
$ curl -u "monerowallet:moneropass" --digest -H 'Content-Type: application/json' http://127.0.0.1:5001/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_balance", "params":{"account_index":0,"address_indices":[0,1]}}'
{
  "id": "0",
  "jsonrpc": "2.0",
  "result": {
    "balance": 2000000000000,
    "blocks_to_unlock": 9,
    "multisig_import_needed": false,
    "per_subaddress": [{
      "account_index": 0,
      "address": "9yFRULuhG5waV5WiHtQocu7qZV92DsGQWVcjjJWHRqPGHxZxdZHNr9s8awvg3ohWqC3g5RBNdvqmwMqX6Fgt5qtTRcpZ6ys",
      "address_index": 0,
      "balance": 1000000000000,
      "blocks_to_unlock": 7,
      "label": "Primary account",
      "num_unspent_outputs": 1,
      "time_to_unlock": 0,
      "unlocked_balance": 0
    },{
      "account_index": 0,
      "address": "BYDnBFn9sy1TYxutCWdGydakgK8tDB1huFtLfEAtHMmyYPAZePC3oivCkbQtrGpRWF8qvvjENtuekNtYVenGodC24hpG9Xz",
      "address_index": 1,
      "balance": 0,
      "blocks_to_unlock": 9,
      "label": "main",
      "num_unspent_outputs": 1,
      "time_to_unlock": 0,
      "unlocked_balance": 0
    }],
    "time_to_unlock": 0,
    "unlocked_balance": 0
  }
}
```

## Resources

Node Resources:
- https://monerodocs.org/interacting/monerod-reference/
- https://monerodocs.org/interacting/monero-wallet-cli-reference/
- https://monerodocs.org/interacting/monero-wallet-rpc-reference/

Developer Resources:
- https://www.getmonero.org/resources/developer-guides/wallet-rpc.html
- https://www.getmonero.org/resources/developer-guides/daemon-rpc.html

Testnet Faucets:
- https://melo.tools/

## Thank You

Thanks for reading, if you like my content, check out my **[website](https://ruan.dev)** or follow me at **[@ruanbekker](https://twitter.com/ruanbekker)** on Twitter.

