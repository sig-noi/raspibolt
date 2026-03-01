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

Running signet alongside mainnet means two separate `bitcoind` processes, each with its own config file, data directory, and set of ports. The same principle then cascades to Electrs, BTC RPC Explorer, and LND. RTL reuses the existing installation but runs as a second process pointed at a separate config file. Sparrow Wallet connects to the signet Electrs instance by switching its network mode. The adjustments below are minimal: mostly different port numbers and file paths that avoid colliding with your existing mainnet setup.

### Port reference

| Service | Mainnet | Signet |
|---|---|---|
| Bitcoin P2P | 8333 | 38333 |
| Bitcoin RPC | 8332 | 38332 |
| Bitcoin ZMQ rawblock | 28332 | 28336 |
| Bitcoin ZMQ rawtx | 28333 | 28337 |
| Electrs RPC | 50001 | 60601 |
| Electrs SSL (NGINX) | 50002 | 60602 |
| BTC RPC Explorer (internal) | 3002 | 3003 |
| BTC RPC Explorer SSL (NGINX) | 4000 | 4002 |
| RTL (internal) | 3000 | 3001 |
| RTL SSL (NGINX) | 4001 | 4004 |
| LND REST API | 8080 | 8091 |

---

## Bitcoin daemon
### Configuration

From user "bitcoin"

  ```sh
  $ sudo su - bitcoin
  ```

Create the Signet data folder

  ```sh
  $ mkdir /data/bitcoin/signet
  ```

Create a **new config file** for the signet instance.
Replace the whole line starting with "rpcauth=" the auth line copied from /home/bitcoin/.bitcoin/bitcoin.conf.
Save and exit.

  ```sh
  $ nano /home/bitcoin/.bitcoin/bitcoin-signet.conf
  ```

  ```ini
  # RaspiBolt: bitcoind configuration for signet node (runs alongside mainnet)

  # [chain]
  chain=signet
  
  # Allow coexistence with bitcoin.conf in the same datadir
  allowignoredconf=1

  # [core]
  blocksonly=1
  txindex=1
  # disable dbcache after full sync
  dbcache=2000

  # [network]
  listen=1
  listenonion=1
  proxy=127.0.0.1:9050
  maxconnections=40
  maxuploadtarget=5000
  whitelist=download@127.0.0.1          # for Electrs

  # [rpc]
  rpcauth=<replace with the auth line copied from /home/bitcoin/.bitcoin/bitcoin.conf>
  rpccookieperms=group
  server=1

  # [zeromq]
  # Use ports that do not conflict with mainnet (mainnet uses 28332/28333)
  zmqpubrawblock=tcp://127.0.0.1:28336
  zmqpubrawtx=tcp://127.0.0.1:28337

  # Per-chain options
  [signet]
  rpcport=38332
  rpcbind=127.0.0.1
  bind=127.0.0.1
  ```

Set permissions: only the user 'bitcoin' and members of the 'bitcoin' group can read it

  ```sh
  $ chmod 640 /home/bitcoin/.bitcoin/bitcoin-signet.conf
  ```

### Running bitciond signet

Still logged in as user "bitcoin", let's start "bitcoind" manually.

* Start "bitcoind".
  Monitor the log file a few minutes to see if it works fine (it may stop at "dnsseed thread exit", that's ok).

  ```sh
  $ bitcoind -conf=/home/bitcoin/.bitcoin/bitcoin-signet.conf
  ```

* Once everything looks ok, stop "bitcoind" with `Ctrl-C`

* Grant the "bitcoin" group read-permission for the debug log file:

  ```sh
  $ chmod g+r /data/bitcoin/signet/debug.log
  ```

* Exit the “bitcoin” user session back to user “admin”

  ```sh
  $ exit
  ```

### Separate systemd unit

Create a **new** systemd service for the signet daemon, leaving your existing `bitcoind.service` (mainnet) untouched.

  ```
  $ sudo nano /etc/systemd/system/bitcoind-signet.service
  ```


```ini
# RaspiBolt: systemd unit for signet bitcoind instance
# /etc/systemd/system/bitcoind-signet.service

[Unit]
Description=Bitcoin daemon (signet)
After=network.target

[Service]

# Service execution
###################

ExecStart=/usr/local/bin/bitcoind -daemon \
                                  -pid=/run/bitcoind-signet/bitcoind-signet.pid \
                                  -conf=/home/bitcoin/.bitcoin/bitcoin-signet.conf \
                                  -datadir=/home/bitcoin/.bitcoin \
                                  -startupnotify="systemd-notify --ready"

# Process management
####################
Type=forking
PIDFile=/run/bitcoind-signet/bitcoind-signet.pid
Restart=on-failure
TimeoutSec=300
RestartSec=30

# Directory creation and permissions
####################################
User=bitcoin
UMask=0027

# /run/bitcoind-signet
RuntimeDirectory=bitcoind-signet
RuntimeDirectoryMode=0710

# Hardening measures
####################
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
MemoryDenyWriteExecute=true

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

To check bitcoind activity:
```sh
$ tail -f /data/bitcoin/signet/debug.log
```

### Interacting with the signet daemon

Pass the signet config file whenever you use `bitcoin-cli` to target the signet instance:
```sh
$ bitcoin-cli -conf=/home/bitcoin/.bitcoin/bitcoin-signet.conf getblockchaininfo
```

To make this less verbose, you can define a shell alias in `~/.bashrc`:
```sh
alias bitcoin-cli-signet='bitcoin-cli -conf=/home/bitcoin/.bitcoin/bitcoin-signet.conf'
```

---

## Electrs

The `electrs` user, binary, and `/data/electrs` data directory are already in place from the mainnet installation. We only need to add a second config file and a second systemd unit.

### Configuration

The config file must be created as the `electrs` user, exactly as the mainnet guide does.

* Switch to the `electrs` user

  ```sh
  $ sudo su - electrs
  ```

* Create the signet config file

  ```sh
  $ nano /data/electrs/electrs-signet.conf
  ```

  ```ini
  # RaspiBolt: electrs configuration for signet node
  # /data/electrs/electrs-signet.conf

  # Bitcoin Core settings
  network = "signet"
  daemon_dir= "/data/bitcoin"
  daemon_rpc_addr = "127.0.0.1:38332"
  daemon_p2p_addr = "127.0.0.1:38333"
  cookie_file = "/data/bitcoin/signet/.cookie"

  # Electrs settings
  electrum_rpc_addr = "127.0.0.1:60601"
  db_dir = "/data/electrs/db-signet"

  # Logging
  log_filters = "INFO"
  timestamp = true
  ```

  Note that `daemon_dir` points to the real data path `/data/bitcoin` (consistent with the mainnet config), not the symlink. `cookie_file` is set explicitly because the signet cookie lives in the `signet/` subdirectory rather than directly under `daemon_dir`.

* Start Electrs manually first to confirm everything works. It will begin indexing immediately.

  ```sh
  $ electrs --conf /data/electrs/electrs-signet.conf
  ```

  You should see output similar to:

  ```
  Starting electrs 0.9.14 on aarch64 linux with Config { network: Signet, db_path: "/data/electrs/db-signet/signet", ...
  [INFO  electrs::server] serving Electrum RPC on 127.0.0.1:60601
  [INFO  electrs::index] indexing 2000 blocks: [1..2000]
  ...
  ```

* Stop Electrs with `Ctrl`-`C` and exit the `electrs` user session.

  ```sh
  $ exit
  ```

### Autostart on boot

* As user "admin", create the signet systemd unit

  ```sh
  $ sudo nano /etc/systemd/system/electrs-signet.service
  ```

  ```ini
  # RaspiBolt: systemd unit for electrs (signet)
  # /etc/systemd/system/electrs-signet.service

  [Unit]
  Description=Electrs signet daemon
  Wants=bitcoind-signet.service
  After=bitcoind-signet.service

  [Service]

  # Service execution
  ###################
  ExecStart=/usr/local/bin/electrs --conf /data/electrs/electrs-signet.conf

  # Process management
  ####################
  Type=simple
  Restart=always
  TimeoutSec=120
  RestartSec=30
  KillMode=process

  # Directory creation and permissions
  ####################################
  User=electrs

  # /run/electrs-signet
  RuntimeDirectory=electrs-signet
  RuntimeDirectoryMode=0710

  # Hardening measures
  ####################
  PrivateTmp=true
  PrivateDevices=true
  MemoryDenyWriteExecute=true

  [Install]
  WantedBy=multi-user.target
  ```

* Enable and start Electrs signet

  ```sh
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable electrs-signet
  $ sudo systemctl start electrs-signet
  ```

* Check the log output

  ```sh
  $ sudo journalctl -f -u electrs-signet
  ```

  Signet has far fewer blocks than mainnet so the initial index completes in minutes rather than hours. Only proceed to configure wallets once you see Electrs listening on port 60601.

### NGINX

* Enable NGINX reverse proxy to add SSL/TLS encryption to the signet Electrs communication.
  Create the configuration file and paste the following content

  ```sh
  $ sudo nano /etc/nginx/streams-enabled/electrs-signet-reverse-proxy.conf
  ```

  ```nginx
  upstream electrs-signet {
    server 127.0.0.1:60601;
  }

  server {
    listen 60602 ssl;
    proxy_pass electrs-signet;
  }
  ```

* Test and reload NGINX configuration

  ```sh
  $ sudo nginx -t
  $ sudo systemctl reload nginx
  ```

* Configure the firewall to allow incoming requests

  ```sh
  $ sudo ufw allow 60602/tcp comment 'allow Electrum SSL Signet'
  $ sudo ufw status
  ```

### Tor

To use your signet Electrum server when you're on the go, add a separate Tor hidden service alongside the existing ones.

* Add the following three lines in the "location-hidden services" section of the `torrc` file

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```sh
  ############### This section is just for location-hidden services ###
  # electrs signet
  HiddenServiceDir /var/lib/tor/hidden_service_electrs_signet/
  HiddenServiceVersion 3
  HiddenServicePort 60602 127.0.0.1:60602
  ```

* Reload Tor configuration and get your connection address

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_electrs_signet/hostname
  > abcdefg..............xyz.onion
  ```

---

## BTC RPC Explorer

The signet explorer runs as a second instance of BTC RPC Explorer, installed in a separate directory under the same `btcrpcexplorer` user created by the main guide. If you have not yet followed the [Blockchain explorer guide](blockchain-explorer.md), do that first — this section only covers the differences needed for signet.

### Firewall & reverse proxy

* Add a second NGINX reverse proxy config for the signet explorer, using port 4002 to avoid colliding with the mainnet explorer on 4000 and mainnet RTL on 4001

  ```sh
  $ sudo nano /etc/nginx/streams-enabled/btcrpcexplorer-signet-reverse-proxy.conf
  ```

  ```nginx
  upstream btcrpcexplorer-signet {
    server 127.0.0.1:3003;
  }
  server {
    listen 4002 ssl;
  }
  ```

* Test and reload NGINX configuration

  ```sh
  $ sudo nginx -t
  $ sudo systemctl reload nginx
  ```

* Open the firewall for the signet explorer port

  ```sh
  $ sudo ufw allow 4002/tcp comment 'allow BTC RPC Explorer SSL Signet'
  $ sudo ufw status
  ```

### Installation

The `btcrpcexplorer` user and Node.js are already in place. We clone the app into a separate directory so it gets its own `.env` file and runs independently of the mainnet instance.

* Open a session as the `btcrpcexplorer` user

  ```sh
  $ sudo su - btcrpcexplorer
  ```

* Fetch the same version already running for mainnet (or the latest release)

  ```sh
  $ VERSION=$(wget -qO- https://api.github.com/repos/janoside/btc-rpc-explorer/releases/latest | grep -oP '"tag_name": "v\K(.*)(?=")')
  $ echo $VERSION
  > 3.5.1
  ```

* Clone into a separate directory and install dependencies

  ```sh
  $ git clone --branch v$VERSION https://github.com/janoside/btc-rpc-explorer.git btc-rpc-explorer-signet
  $ cd btc-rpc-explorer-signet
  $ npm install
  ```

### Configuration

* Copy and edit the configuration template

  ```sh
  $ cp .env-sample .env
  $ nano .env
  ```
  
* Set the internal port to 3003 so it does not clash with the mainnet explorer on 3002

  ```sh
  BTCEXP_PORT=3003
  ```

* Point BTC RPC Explorer at the signet Bitcoin Core instance

  ```sh
  BTCEXP_BITCOIND_HOST=127.0.0.1
  BTCEXP_BITCOIND_PORT=38332
  BTCEXP_BITCOIND_COOKIE=/data/bitcoin/signet/.cookie
  ```

* Extend the timeout period

  ```sh
  BTCEXP_BITCOIND_RPC_TIMEOUT=10000
  ```

* Point address lookups at the signet Electrs instance

  ```sh
  BTCEXP_ADDRESS_API=electrum
  BTCEXP_ELECTRUM_SERVERS=tcp://127.0.0.1:60601
  ```

* Choose your privacy and theme preferences (same options as mainnet)

  ```sh
  BTCEXP_PRIVACY_MODE=true
  BTCEXP_NO_RATES=true
  BTCEXP_UI_THEME=dark
  ```

* Optionally, add password protection

  ```sh
  BTCEXP_BASIC_AUTH_PASSWORD=YourPassword[D]
  ```

* Save and exit

### First start

* From the `btc-rpc-explorer-signet` directory, start the explorer manually to confirm it works

  ```sh
  $ cd ~/btc-rpc-explorer-signet
  $ npm run start
  ```

* Browse to <https://raspibolt.local:4002> to verify the signet explorer is reachable. Stop it with `Ctrl-c` when done, then exit the user session.

  ```sh
  $ Ctrl-c
  $ exit
  ```

### Autostart on boot

* As user "admin", create a dedicated systemd unit for the signet explorer

  ```sh
  $ sudo nano /etc/systemd/system/btcrpcexplorer-signet.service
  ```

  ```ini
  # RaspiBolt: systemd unit for BTC RPC Explorer (signet)
  # /etc/systemd/system/btcrpcexplorer-signet.service

  [Unit]
  Description=BTC RPC Explorer (signet)
  After=bitcoind-signet.service electrs-signet.service
  PartOf=bitcoind-signet.service

  [Service]
  WorkingDirectory=/home/btcrpcexplorer/btc-rpc-explorer-signet
  ExecStart=/usr/bin/npm start
  User=btcrpcexplorer

  Restart=always
  RestartSec=30

  [Install]
  WantedBy=multi-user.target
  ```

* Enable and start the service

  ```sh
  $ sudo systemctl enable btcrpcexplorer-signet.service
  $ sudo systemctl start btcrpcexplorer-signet.service
  $ sudo journalctl -f -u btcrpcexplorer-signet
  ```

* Now add a `Wants=` line to the bitcoind-signet service so the explorer starts and stops automatically with the signet node — exactly as the mainnet guide does for `bitcoind.service`:

```sh
$ sudo nano /etc/systemd/system/bitcoind-signet.service
```

```ini
[Unit]
Description=Bitcoin daemon (signet)
After=network.target
Wants=btcrpcexplorer-signet.service
```

```sh
$ sudo systemctl daemon-reload
```


### Remote access over Tor (optional)

* Add a hidden service for the signet explorer in the `torrc` file

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```ini
  # Hidden Service BTC RPC Explorer (signet)
  HiddenServiceDir /var/lib/tor/hidden_service_btcrpcexplorer_signet/
  HiddenServiceVersion 3
  HiddenServicePort 80 127.0.0.1:3003
  ```

* Reload Tor and retrieve the onion address

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_btcrpcexplorer_signet/hostname
  > abcdefg..............xyz.onion
  ```

---

## Joinmarket-ng
*** explanation etc ***

### Create a dedicated user and a data directory

* Create the “joinmarket” user, and make it a member of the “bitcoin” and "debian-tor" groups

```
$ sudo adduser --disabled-password --gecos "" joinmarket-ng
$ sudo usermod -a -G bitcoin,debian-tor joinmarket-ng
```

* Create a JoinMarket data directory

```
$ sudo mkdir /data/joinmarket-ng
$ sudo chown -R joinmarket-ng:joinmarket-ng /data/joinmarket-ng
```

* Open a "joinmarket" user session

```
$ sudo su - joinmarket-ng
```

* Create a symbolic link pointing to the joinmarket data directory

```
$ ln -s /data/joinmarket-ng /home/joinmarket-ng/.joinmarket-ng
```

--- UNTESTED ---

## LND

LND only supports a single active network at a time, so signet requires a fully independent second instance with its own data directory, configuration file, and systemd unit. The LND binary itself is shared — only the runtime data and config differ.

### Installation

The LND binary is already installed from the mainnet setup. If you need to install it for the first time, or to verify the installed version, follow the [Download](lightning-client.md#download), [Checksum check](lightning-client.md#checksum-check), [Signature check](lightning-client.md#signature-check), [Timestamp check](lightning-client.md#timestamp-check) and [Installation](lightning-client.md#installation) sections of the Lightning client guide, then return here.

The `lnd` user and its group memberships (`bitcoin`, `debian-tor`) are also already in place from the mainnet installation.

### Data directory

* As user "admin", create the signet LND data directory

  ```sh
  $ sudo mkdir /data/lnd-signet
  $ sudo chown -R lnd:lnd /data/lnd-signet
  ```

* Open an "lnd" user session

  ```sh
  $ sudo su - lnd
  ```

### Wallet password

* Still as user "lnd", create the signet wallet password file and enter your `password [C]`. Save and exit.

  ```sh
  $ nano /data/lnd-signet/password.txt
  ```

* Tighten access privileges

  ```sh
  $ chmod 600 /data/lnd-signet/password.txt
  ```

### Configuration

* Create the signet LND configuration file. Save and exit.

  ```sh
  $ nano /data/lnd-signet/lnd.conf
  ```

  ```ini
  # RaspiBolt: lnd configuration for signet
  # /data/lnd-signet/lnd.conf

  [Application Options]
  alias=YOUR_FANCY_ALIAS
  debuglevel=info
  maxpendingchannels=5
  listen=localhost

  # Store all signet data separately from the mainnet LND instance
  datadir=/data/lnd-signet

  # Separate RPC port from mainnet LND (default 10009)
  rpclisten=localhost:10010

  # Separate REST port from mainnet LND (default 8080)
  # RTL and other REST clients use this to connect
  restlisten=localhost:8091

  # Explicit TLS paths to avoid overwriting mainnet LND's cert/key
  tlscertpath=/data/lnd-signet/tls.cert
  tlskeypath=/data/lnd-signet/tls.key

  # Password: automatically unlock wallet with the password in this file
  wallet-unlock-password-file=/data/lnd-signet/password.txt
  wallet-unlock-allow-create=true

  # Automatically regenerate certificate when near expiration
  tlsautorefresh=true
  tlsdisableautofill=true

  # Channel settings
  bitcoin.basefee=1000
  bitcoin.feerate=1
  minchansize=100000
  accept-keysend=true
  accept-amp=true
  coop-close-target-confs=24

  # Set to enable support for the experimental taproot channel type, wumbo channels (> 16.7M Sats) and RBF coop closures
  protocol.simple-taproot-chans=true
  protocol.wumbo-channels=true
  protocol.rbf-coop-close=true

  # Watchtower
  wtclient.active=true

  # Performance
  gc-canceled-invoices-on-startup=true
  gc-canceled-invoices-on-the-fly=true
  ignore-historical-gossip-filters=1
  stagger-initial-reconnect=true

  # Database
  [bolt]
  db.bolt.auto-compact=true
  db.bolt.auto-compact-min-age=168h

  [Bitcoin]
  bitcoin.signet=true
  bitcoin.node=bitcoind

  [Bitcoind]
  bitcoind.rpchost=127.0.0.1:38332
  bitcoind.rpccookie=/data/bitcoin/signet/.cookie
  bitcoind.zmqpubrawblock=tcp://127.0.0.1:28336
  bitcoind.zmqpubrawtx=tcp://127.0.0.1:28337

  [tor]
  tor.active=true
  tor.v3=true
  # Stream isolation cannot be used alongside skip-proxy-for-clearnet-targets
  # (required for signet DNS seed bootstrapping)
  tor.skip-proxy-for-clearnet-targets=true
  ```

### Run LND manually

Before setting up autostart, confirm the signet instance starts cleanly.

* First, check nothing is already occupying the REST port from a previous run attempt

  ```sh
  $ exit
  $ sudo ss -tlnp | grep -E '10010|8091'
  ```

  If a process is listed, kill it before continuing (substitute the PID shown in the output):

  ```sh
  $ sudo kill <PID>
  ```

* Open an "lnd" user session and start LND

  ```sh
  $ sudo su - lnd
  $ lnd --configfile=/data/lnd-signet/lnd.conf
  ```

  You should see output confirming it connects to signet:

  ```
  LNTD: Version Info rev=etc...
  LNTD: Network Info rev=etc... active_chain=Bitcoin network=signet
  ...
  LTND: Active chain: Bitcoin (network=signet)
  ...
  LTND: Waiting for wallet encryption password.
  ```

### Wallet setup

**With LND running** in the first session, open a **second SSH session** and create the signet wallet. Commands for the second session start with `$2`.

* Start an "lnd" user session in the second terminal

  ```sh
  $2 sudo su - lnd
  ```

* Create the LND signet wallet, pointing `lncli` at the signet RPC port

  ```sh
  $2 lncli --rpcserver=localhost:10010 --tlscertpath=/data/lnd-signet/tls.cert create
  ```

* Enter your `password [C]` as wallet password (must match `password.txt`). Select `n` when asked about an existing seed.

  ```
  Do you have an existing cipher seed mnemonic [...]: n
  ...
  !!!YOU MUST WRITE DOWN THIS SEED TO BE ABLE TO RESTORE THE WALLET!!!

  ---------------BEGIN LND CIPHER SEED---------------
  1. secret     2. secret    3. secret     4. secret
  ...
  ```

🚨 **Write these 24 words down on paper and store them safely.** This is your only on-chain recovery backup.

* Exit the second session

  ```sh
  $2 exit
  $2 exit
  ```

* Back in the first session, stop LND with `Ctrl-C`. Confirm the wallet unlocks automatically on restart, then stop again.

  ```sh
  $ lnd --configfile=/data/lnd-signet/lnd.conf
  > ...
  > LTND: Attempting automatic wallet unlock with password
  > BTWL: Opened wallet

  # stop with Ctrl-C, then exit the lnd session
  $ exit
  ```

### Autostart on boot

* As user "admin", create the signet LND systemd unit. Save and exit.

  ```sh
  $ sudo nano /etc/systemd/system/lnd-signet.service
  ```

  ```ini
  # RaspiBolt: systemd unit for lnd (signet)
  # /etc/systemd/system/lnd-signet.service

  [Unit]
  Description=LND Lightning Network Daemon (signet)
  Wants=bitcoind-signet.service
  After=bitcoind-signet.service

  [Service]

  # Service execution
  ###################
  ExecStart=/usr/local/bin/lnd --configfile=/data/lnd-signet/lnd.conf
  ExecStop=/usr/local/bin/lncli --rpcserver=localhost:10010 --tlscertpath=/data/lnd-signet/tls.cert stop
  
  # Process management
  ####################
  Type=simple
  Restart=always
  RestartSec=30
  TimeoutSec=240
  LimitNOFILE=128000

  # Directory creation and permissions
  ####################################
  User=lnd

  # /run/lightningd-signet
  RuntimeDirectory=lightningd-signet
  RuntimeDirectoryMode=0710

  # Hardening measures
  ####################
  PrivateTmp=true
  ProtectSystem=full
  NoNewPrivileges=true
  PrivateDevices=true
  MemoryDenyWriteExecute=true

  [Install]
  WantedBy=multi-user.target
  ```

* Enable and start the service

  ```sh
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable lnd-signet
  $ sudo systemctl start lnd-signet
  $ sudo journalctl -f -u lnd-signet
  ```

### Allow user "admin" to work with the signet LND

* Link the signet LND data directory into the "admin" home, and open the necessary permissions

  ```sh
  $ ln -s /data/lnd-signet /home/admin/.lnd-signet
  $ sudo chmod -R g+X /data/lnd-signet/
  $ sudo chmod g+r /data/lnd-signet/chain/bitcoin/signet/admin.macaroon
  ```

### Interacting with the signet LND daemon

Always pass `--rpcserver` and `--macaroonpath` to target the signet instance instead of mainnet:

```sh
$ lncli --rpcserver=localhost:10010 \
        --tlscertpath=/data/lnd-signet/tls.cert \
        --macaroonpath=/data/lnd-signet/chain/bitcoin/signet/admin.macaroon \
        getinfo
```

Add a shell alias to `~/.bashrc` to avoid repeating these flags:

```sh
alias lncli-signet='lncli --rpcserver=localhost:10010 --tlscertpath=/data/lnd-signet/tls.cert --macaroonpath=/data/lnd-signet/chain/bitcoin/signet/admin.macaroon'
```

Then interact with the signet node as naturally as mainnet:

```sh
$ lncli-signet walletbalance
$ lncli-signet newaddress p2wkh
```

---

## Ride The Lightning

RTL supports running against a second LND instance using the `RTL_CONFIG_PATH` environment variable to point to a separate config directory. RTL always looks for a file named `RTL-Config.json` inside that directory, so the signet config lives at `/home/rtl/signet-rtl/RTL-Config.json`. This means the existing RTL installation at `/home/rtl/RTL` is reused — no second clone is needed. If you have not yet followed the [Web app guide](web-app.md), do that first.

### Firewall & reverse proxy

* Enable NGINX reverse proxy for the signet RTL instance on port 4004

  ```sh
  $ sudo nano /etc/nginx/streams-enabled/rtl-signet-reverse-proxy.conf
  ```

  ```nginx
  upstream rtl-signet {
    server 127.0.0.1:3001;
  }
  server {
    listen 4004 ssl;
    proxy_pass rtl-signet;
  }
  ```

* Test and reload NGINX configuration

  ```sh
  $ sudo nginx -t
  $ sudo systemctl reload nginx
  ```

* Configure the firewall to allow incoming HTTPS requests

  ```sh
  $ sudo ufw allow 4004/tcp comment 'allow RTL SSL Signet'
  $ sudo ufw status
  ```

### Configuration

* Open an "rtl" user session

  ```sh
  $ sudo su - rtl
  ```

* Create a subdirectory to hold the signet macaroon and database separately from mainnet

  ```sh
  $ mkdir /home/rtl/signet
  ```

* Exit back to admin and copy the signet LND admin macaroon as a privileged operation

  ```sh
  $ exit
  $ sudo cp /data/lnd-signet/chain/bitcoin/signet/admin.macaroon /home/rtl/signet/admin.macaroon
  $ sudo chown rtl:rtl /home/rtl/signet/admin.macaroon
  ```

* Re-open the "rtl" user session to continue with configuration

  ```sh
  $ sudo su - rtl
  ```

* Create the signet RTL config directory and open the config file with the required name

  ```sh
  $ mkdir /home/rtl/signet-rtl
  $ nano /home/rtl/signet-rtl/RTL-Config.json
  ```

  ```json
  {
    "multiPass": "YourPassword[E]",
    "port": "3001",
    "defaultNodeIndex": 1,
    "dbDirectoryPath": "/home/rtl/signet-rtl",
    "SSO": {
      "rtlSSO": 0,
      "rtlCookiePath": "",
      "logoutRedirectLink": ""
    },
    "nodes": [
      {
        "index": 1,
        "lnNode": "Signet Node",
        "lnImplementation": "LND",
        "authentication": {
          "macaroonPath": "/home/rtl/signet",
          "configPath": "/data/lnd-signet/lnd.conf"
        },
        "settings": {
          "userPersona": "OPERATOR",
          "themeMode": "NIGHT",
          "themeColor": "INDIGO",
          "fiatConversion": false,
          "unannouncedChannels": false,
          "logLevel": "INFO",
          "lnServerUrl": "https://127.0.0.1:8091",
          "swapServerUrl": "https://127.0.0.1:8081",
          "boltzServerUrl": "https://127.0.0.1:9003"
        }
      }
    ]
  }
  ```

  The `multiPass` value here is independent of the mainnet RTL password — use the same or a different one as you prefer. The `lnServerUrl` points to port 8091, the signet LND REST port set in `lnd-signet.conf`.

### First start

* Start RTL manually, pointing it at the signet config

  ```sh
  $ cd RTL/
  $ RTL_CONFIG_PATH=/home/rtl/signet-rtl node rtl
  > Server is up and running, please open the UI at http://localhost:3001
  ```

* Browse to <https://raspibolt.local:4004>. Accept the self-signed certificate warning and log in with your `password [E]`. Stop RTL with `Ctrl-C` once confirmed, then exit.

  ```sh
  $ exit
  ```

### Autostart on boot

* As user "admin", create the signet RTL systemd unit. Save and exit.

  ```sh
  $ sudo nano /etc/systemd/system/rtl-signet.service
  ```

  ```ini
  # RaspiBolt: systemd unit for Ride the Lightning (signet)
  # /etc/systemd/system/rtl-signet.service

  [Unit]
  Description=Ride the Lightning (signet)
  After=lnd-signet.service

  [Service]
  WorkingDirectory=/home/rtl/RTL
  ExecStart=/usr/bin/node rtl
  User=rtl
  Environment="RTL_CONFIG_PATH=/home/rtl/signet-rtl"

  Restart=always
  RestartSec=30

  [Install]
  WantedBy=multi-user.target
  ```

* Enable and start the service

  ```sh
  $ sudo systemctl daemon-reload
  $ sudo systemctl enable rtl-signet
  $ sudo systemctl start rtl-signet
  $ sudo journalctl -f -u rtl-signet
  ```

### Remote access over Tor (optional)

* Add a hidden service for the signet RTL instance in the `torrc` file

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```ini
  ############### This section is just for location-hidden services ###
  # Hidden Service RTL (signet)
  HiddenServiceDir /var/lib/tor/hidden_service_rtl_signet/
  HiddenServiceVersion 3
  HiddenServicePort 80 127.0.0.1:3001
  ```

* Reload Tor and retrieve the onion address

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_rtl_signet/hostname
  > abcdefg..............xyz.onion
  ```

### Updating the macaroon after LND signet wallet changes

If you ever recreate the signet LND wallet, the macaroon will be regenerated and you must recopy it:

```sh
$ sudo cp /data/lnd-signet/chain/bitcoin/signet/admin.macaroon /home/rtl/signet/admin.macaroon
$ sudo chown rtl:rtl /home/rtl/signet/admin.macaroon
$ sudo systemctl restart rtl-signet
```

---

## Getting signet coins

Unlike testnet, signet coins come from a dedicated faucet rather than mining. A few publicly available options:

- **signet.bc-2.jp** — web faucet, paste your signet address to receive tBTC
- **signetfaucet.com** — alternative web faucet
- Bitcoin Core's own signet (`-signet`) is compatible with the global default signet signing key, so these faucets work with a standard configuration.

---

<< Back: [+ Bitcoin](index.md)
