# Solana Validator Monitoring Tool

*This post is Part 1 of a 3-part series about setting up proper monitoring on your Solana Validator.*

* [Part 1.](https://github.com/stakeconomy/solanamonitoring/blob/main/README.md) Solana Validator Monitoring Tool
* [Part 2.](https://github.com/stakeconomy/solanamonitoring/blob/main/How%20to%20Install%20TIG%20Stack.md) How to Install Telegraf, InfluxDB, and Grafana
* [Part 3.](https://github.com/stakeconomy/solanamonitoring/blob/main/Guidelines%20interpreting%20metrics.md) Interpreting monitoring metrics

## Introduction

### Telegraf | A Metrics Collector For InfluxDB

Telegraf can collect metrics from a wide array of inputs and write them to a wide array of outputs. It is plugin-driven for both collection and output of data so it is easily extendable. It is written in Go, which means that it is compiled and standalone binary that can be executed on any system with no need for external dependencies, or package management tools required.

Telegraf is an open-source tool. It contains over 200 plugins for gathering and writing different types of data written by people who work with that data.

### Telegraf benefits
- Easy to setup
- Minimal memory footprint
- Over 200 plugins available
- Able to send metrics to central InfluxDB over http(s) without the need of client configuration

### Architecture

![Architecture](https://i.imgur.com/xmbND94.png)

### Solana Monitoring
The solution consist of a standard telegraf installation and one bash script "monitor.sh" that will get all server performance and validator performance metrics every 30 seconds and send all the metrics to a local or remote influx database server.

![Sample Dashboard](https://i.imgur.com/2CB2F1o.png)

# Features
* Simple setup with minimal performance impact to monitor validator node.
* Sample Dashboard to import into Grafana.
* Use of community dashboard on https://metrics.stakeconomy.com possible so you don't need to setup your own monitoring system.
* Customizable Parameters. You can use your own RPC node or Solana public RPC nodes (much slower).

TBD
------
* Optimize the way how we get skip-rate. 
* Rebuild solana monitoring script as telegraf input plugin written in Go.

# インストールと設定

A fully functional Solana Validator is required to setup monitoring. In the example below we use Ubuntu 20.04.
To get all metrics from your local Validator RPC.

In the examples below we setup the validator with user "solv" with it's home in /home/solv. It is required that the script is installed and run under that same user.
You need to install the telegraf agent on your validator nodes. 

To have full statistics that include a whole epoch, make sure that your --limit-ledger-size configuration is big enough to store a whole epoch:

```       --limit-ledger-size <SHRED_COUNT>                       Keep this amount of shreds in root slots.```

You may use 250000000 for ~1 epoch or leavy it empty to use the default. 
Using less schred's to save diskspace still works, but it will mess up your leaderslots and skiprate stats.

```
# install telegraf
cat <<EOF | sudo tee /etc/apt/sources.list.d/influxdata.list
deb https://repos.influxdata.com/ubuntu bionic stable
EOF

sudo curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -

sudo apt-get update
sudo apt-get -y install telegraf jq bc

sudo systemctl enable --now telegraf
sudo systemctl is-enabled telegraf
sudo systemctl status telegraf

# make the telegraf user sudo and adm to be able to execute scripts as sol user
sudo adduser _telegraf sudo
sudo adduser _telegraf adm
sudo -- bash -c 'echo "_telegraf ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'

sudo cp /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.orig
sudo rm -rf /etc/telegraf/telegraf.conf

# make sure you are the user you run solana with . eq. su - solana
git clone https://github.com/pico-sol/solanamonitoring/
cd solanamonitoring


```

# monitor.shスクリプトの編集
config部分のidentityPubkey, voteAccount, rpcURLを編集:
```
#####    CONFIG    ##################################################################################################
configDir="$HOME/.config/solana" # the directory for the config files, eg.: /home/user/.config/solana
##### optional:        #
identityPubkey="********************************************"      # identity pubkey for the validator, insert if autodiscovery fails
voteAccount="********************************************"         # vote account address for the validator, specify if there are more than one or if autodiscovery fails
additionalInfo="on"    # set to 'on' for additional general metrics like balance on your vote and identity accounts, number of validator nodes, epoch number and percentage epoch elapsed
binDir=""              # auto detection of the solana binary directory can fail or an alternative custom installation is preferred, in case insert like $HOME/solana/target/release
rpcURL="https://mainnet.helius-rpc.com/?api-key=****************************************"              # default is localhost with port number autodiscovered, alternatively it can be specified like http://custom.rpc.com:port
format="SOL"           # amounts shown in 'SOL' instead of lamports
now=$(date +%s%N)      # date in influx format
timezone="UTC"            # time zone for epoch ends metric
#####  END CONFIG  ##################################################################################################
```

# telegraf 設定例
設定ファイル /etc/telegraf/telegraf.conf を以下を参考に編集:

hostname, monitorのマウントポイントurl, monitorスクリプトの場所やusernameを編集

```
# Global Agent Configuration
[agent]
  hostname = "****-mainnet-*****" # set this to a name you want to identify your node in the grafana dashboard
  flush_interval = "15s"
  interval = "15s"

# Input Plugins
[[inputs.cpu]]
    percpu = true
    totalcpu = true
    collect_cpu_time = false
    report_active = false
[[inputs.disk]]
    ignore_fs = ["devtmpfs", "devfs"]
[[inputs.diskio]]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
[[inputs.swap]]
[[inputs.netstat]]
[[inputs.processes]]
[[inputs.kernel]]
[[inputs.diskio]]

# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "metricsdb"
  urls = [ "http://metrics.stakeconomy.com:8086" ] # keep this to send all your metrics to the community dashboard otherwise use http://yourownmonitoringnode:8086
  username = "metrics" # keep both values if you use the community dashboard
  password = "password"

[[inputs.exec]]
  commands = ["sudo -S su -c /home/solv/solanamonitoring/monitor.sh -s /bin/bash solv"] # change home and username to the useraccount your validator runs at
  interval = "5m"
  timeout = "1m"
  data_format = "influx"
  data_type = "integer"
```


Please continue to [Part 2.](https://github.com/stakeconomy/solanamonitoring/blob/main/How%20to%20Install%20TIG%20Stack.md) that was written to help you setup your own TIG (Telegraf/InfluxDB/Grafana) stack.

Stake with Stakeconomy.com validator on Solflare.
Vote Account: [GNZ1PAAS33davY4Q1BMEpZEpVBtRtGvSpcTH5wYVkkVt](https://solanabeach.io/validator/GNZ1PAAS33davY4Q1BMEpZEpVBtRtGvSpcTH5wYVkkVt)
