---
layout: default
title: RaspiBolt on Signet (alongside Mainnet)
parent: + Bitcoin
grand_parent: Bonus Section
nav_exclude: true
has_children: false
has_toc: false
---

# Bonus guide: RaspiBolt on Signet (alongside Mainnet)
{: .no_toc }

This guide walks you through running a Bitcoin **signet** node in parallel with an existing mainnet installation. Signet is a controlled test network with realistic block production, making it ideal for testing applications without risking real funds. Because both instances run simultaneously, each service requires its own configuration file, ports, and systemd unit.

Difficulty: Medium
{: .label .label-yellow }

Status: Tested v3
{: .label .label-green }

---

Table of contents
{: .text-delta }

1. TOC
{:toc}

---

## Introduction

Unlike testnet — where anyone can mine blocks, leading to unpredictable block times — signet uses a trusted signing key to produce blocks on a regular schedule. This makes it much more stable for development and testing.

Running signet alongside mainnet means two separate `bitcoind` processes, each with its own config file, data directory, and set of ports. The same principle then cascades to Electrs and LND. The adjustments below are minimal: mostly different port numbers and file paths that avoid colliding with your existing mainnet setup.

### Port reference

| Service | Mainnet | Signet |
|---|---|---|
| Bitcoin P2P | 8333 | 38333 |
| Bitcoin RPC | 8332 | 38332 |
| Bitcoin ZMQ rawblock | 28332 | 28336 |
| Bitcoin ZMQ rawtx | 28333 | 28337 |
| Electrs RPC | 50001 | 60601 |
| Electrs SSL (NGINX) | 50002 | 60602 |

---

## Bitcoin daemon

Rather than modifying your existing `bitcoin.conf`, create a **separate config file** for the signet instance.

File location: `/data/bitcoin/bitcoin-signet.conf`
```ini
# RaspiBolt: bitcoind configuration for signet node (runs alongside mainnet)

# [chain]
chain=signet

# [core]
sysperms=1
blocksonly=1
txindex=1
# disable dbcache after full sync
dbcache=2000

# [wallet]
disablewallet=1

# [network]
listen=1
listenonion=1
proxy=127.0.0.1:9050
maxconnections=40
maxuploadtarget=5000
whitelist=download@127.0.0.1          # for Electrs

# [rpc]
rpcauth=your_string_from_the_rpcauth_script
server=1

# [zeromq]
# Use ports that do not conflict with mainnet (mainnet uses 28332/28333)
zmqpubrawblock=tcp://127.0.0.1:28336
zmqpubrawtx=tcp://127.0.0.1:28337

# Options specific to each chain
[main]
# rpcport=8332
bind=127.0.0.1
rpcbind=127.0.0.1

[signet]
rpcport=38332
bind=127.0.0.1:38333
rpcbind=127.0.0.1

[test]
# rpcport=18332

[regtest]
# rpcport=18443
```

Signet chain data is stored in `/data/bitcoin/signet/`. To check the log file:
```sh
$ tail -f /data/bitcoin/signet/debug.log
```

### Separate systemd unit

Create a **new** systemd service for the signet daemon, leaving your existing `bitcoind.service` (mainnet) untouched.

File location: `/etc/systemd/system/bitcoind-signet.service`
```ini
# RaspiBolt: systemd unit for signet bitcoind instance
[Unit]
Description=Bitcoin daemon (signet)
After=network.target

[Service]
ExecStartPre=/bin/sh -c 'sleep 30'
ExecStart=/usr/local/bin/bitcoind \
  -conf=/data/bitcoin/bitcoin-signet.conf \
  -datadir=/data/bitcoin \
  -startupnotify="chmod g+r /home/bitcoin/.bitcoin/signet/.cookie"
ExecStop=/usr/local/bin/bitcoin-cli \
  -conf=/data/bitcoin/bitcoin-signet.conf \
  stop
User=bitcoin
Group=bitcoin
Type=forking
PIDFile=/data/bitcoin/signet/bitcoind.pid
Restart=on-failure
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl enable bitcoind-signet
$ sudo systemctl start bitcoind-signet
$ sudo systemctl status bitcoind-signet
```

### Interacting with the signet daemon

Pass the signet config file whenever you use `bitcoin-cli` to target the signet instance:
```sh
$ bitcoin-cli -conf=/data/bitcoin/bitcoin-signet.conf getblockchaininfo
```

To make this less verbose, you can define a shell alias in `~/.bashrc`:
```sh
alias bitcoin-cli-signet='bitcoin-cli -conf=/data/bitcoin/bitcoin-signet.conf'
```

---

## Electrs

Create a separate Electrs configuration file for signet.

File location: `/data/electrs/electrs-signet.conf`
```ini
# RaspiBolt: electrs configuration for signet node
# /data/electrs/electrs-signet.conf

# Bitcoin Core settings
network = "signet"
daemon_dir= "/data/bitcoin/"
daemon_rpc_addr = "127.0.0.1:38332"
daemon_p2p_addr = "127.0.0.1:38333"
cookie_file = "/data/bitcoin/signet/.cookie"

# Electrs settings
electrum_rpc_addr = "127.0.0.1:60601"
db_dir = "/data/electrs/db-signet/"
index_lookup_limit = 1000

# Logging
log_filters = "INFO"
timestamp = true
```

### Separate systemd unit for Electrs

File location: `/etc/systemd/system/electrs-signet.service`
```ini
[Unit]
Description=Electrs signet index
After=bitcoind-signet.service

[Service]
ExecStart=/usr/local/bin/electrs \
  --conf /data/electrs/electrs-signet.conf
User=electrs
Group=electrs
Restart=on-failure
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Enable and start:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl enable electrs-signet
$ sudo systemctl start electrs-signet
```

Check the logs:
```sh
$ sudo journalctl -u electrs-signet -f
```

### NGINX

File location: `/etc/nginx/streams-enabled/electrs-signet-reverse-proxy.conf`
```ini
upstream electrs-signet {
  server 127.0.0.1:60601;
}

server {
  listen 60602 ssl;
  proxy_pass electrs-signet;
}
```

Reload NGINX and open the firewall:
```sh
$ sudo nginx -t
$ sudo systemctl reload nginx
$ sudo ufw allow 60602/tcp comment 'allow Electrum SSL Signet'
```

### Tor

Add a separate hidden service for signet Electrs in the location-hidden services section.

File location: `/etc/tor/torrc`
```ini
############### This section is just for location-hidden services ###
HiddenServiceDir /var/lib/tor/hidden_service_electrs_signet/
HiddenServiceVersion 3
HiddenServicePort 60602 127.0.0.1:60602
```

Reload Tor and retrieve the hostname:
```sh
$ sudo systemctl reload tor
$ sudo cat /var/lib/tor/hidden_service_electrs_signet/hostname
```

---

## LND

LND only supports a single active network at a time. If you want LND on signet, you will need a **second LND instance** with its own data directory, configuration, and systemd unit.

### Configuration

File location: `/data/lnd-signet/lnd.conf`
```ini
[Application Options]
datadir=/data/lnd-signet/data
logdir=/data/lnd-signet/logs
# Use a different listen port to avoid collision with mainnet LND
listen=0.0.0.0:9735   # change to e.g. 9736 if mainnet LND also listens externally

[Bitcoin]
bitcoin.active=1
bitcoin.signet=1
bitcoin.node=bitcoind

[Bitcoind]
bitcoind.rpchost=127.0.0.1:38332
bitcoind.rpcuser=your_rpcuser
bitcoind.rpcpass=your_rpcpass
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28336
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28337
```

### Separate systemd unit for LND

File location: `/etc/systemd/system/lnd-signet.service`
```ini
[Unit]
Description=LND Lightning daemon (signet)
After=bitcoind-signet.service

[Service]
ExecStart=/usr/local/bin/lnd \
  --configfile=/data/lnd-signet/lnd.conf
User=lnd
Group=lnd
Type=simple
Restart=on-failure
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Enable and start:
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl enable lnd-signet
$ sudo systemctl start lnd-signet
```

Grant group members permission to traverse the signet LND directories:
```sh
$ sudo chmod -R g+X /data/lnd-signet/data/
```

### Interacting with the signet LND daemon

Specify the signet RPC server and macaroon path when using `lncli`:
```sh
$ lncli \
  --rpcserver localhost:10010 \
  --macaroonpath /data/lnd-signet/data/chain/bitcoin/signet/admin.macaroon \
  walletbalance
```

To simplify repeated commands, define a shell alias:
```sh
alias lncli-signet='lncli --rpcserver localhost:10010 --macaroonpath /data/lnd-signet/data/chain/bitcoin/signet/admin.macaroon'
```

---

## Getting signet coins

Unlike testnet, signet coins come from a dedicated faucet rather than mining. A few publicly available options:

- **signet.bc-2.jp** — web faucet, paste your signet address to receive tBTC
- **signetfaucet.com** — alternative web faucet
- Bitcoin Core's own signet (`-signet`) is compatible with the global default signet signing key, so these faucets work with a standard configuration.

---

<< Back: [+ Bitcoin](index.md)
