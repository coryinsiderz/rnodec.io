--- 
title: "Bombay Scramble" 
description: "Script used to get up and running on new bombay testnet" 
date: 2021-05-27
draft: false 
tags:
  - terra
---

***SPECIAL NOTICE:  TERRA HAS FALLEN***

Terra recently released the ***bombay*** network as a testing ground ahead of the much anticipated upgrade from ***columbus-4*** to ***columbus-5*** mainnet.  What follows here is the steps we took to setup a brand new node from scratch on this network.  Usually we would use our [ansible scripts](https://github.com/RnodeC/terra-git/tree/master/terra-ble) to provision a new machine on the terra network, but there are many new wrinkles and features here, so we set this validator node up by hand.  

Also note that we are transplating our existing ***tequila-0004*** testnet validator node over to ***bombay-0005*** here.  As opposed to creating a brand new validator.  The only difference is that we copied over our `priv_validator_key.json` and `node_key.json` before launching the `terrad` daemon.  

First, create the machine:

```bash
gcloud compute instances create bombay \
    --zone us-central1-a \
    --image-family=rhel-7 \
    --image-project=rhel-cloud \
    --machine-type e2-medium \
    --create-disk=name=bombaychain,size=300GB,type=pd-standard,auto-delete=no
```
ssh into it...

## Terra 

* Dependencies  

```bash
sudo yum update -y
sudo yum install git wget jq make gcc -y

# go
wget https://golang.org/dl/go1.16.4.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.16.4.linux-amd64.tar.gz
```  

* Kernel parameter tuning  

```bash
sudo bash -c "cat > /etc/security/limits.d/terrad.conf << EOF
# /etc/security/limits.conf

*                soft    nofile          65535
*                hard    nofile          65535
EOF"
```  

* Format and mount attached disk  

```bash
sudo mkfs -t ext4 /dev/sdb
sudo mkdir /chaindata
sudo mount -t ext4 /dev/sdb /chaindata
```  

* Create terra user  

```bash
sudo useradd terra
sudo su - terra
```  
   
* Build `terrad`  

```bash   
git clone https://github.com/terra-project/core.git
pushd core/
git fetch --all --tags
git checkout v0.5.0-beta2
GOBIN=$HOME/bin make install
which terrad
```  

* Configure `terrad`  

```bash
terrad init RnodeC --chain-id bombay-0005

# get genesis file
wget https://raw.githubusercontent.com/terra-project/testnet/master/bombay-0005/genesis.json -O ${HOME}/.terra/config/genesis.json 

# configure gas prices
sed -i 's/minimum-gas-prices = ""/minimum-gas-prices = "0.15uluna,0.1018usdr,0.15uusd,178.05ukrw,431.6259umnt,0.125ueur,0.97ucny,16.0ujpy,0.11ugbp,11.0uinr,0.ucad,0.3uchf,0.19uaud,0.2usgd,4.62uthb,1.25usek,1.164uhkd"/g' ${HOME}/.terra/config/app.toml

# configure seed nodes
sed -i 's/seed_nodes = ""/seeds = "8eca04192d4d4d7da32149a3daedc4c24b75f4e7@3.34.163.215:26656"/g' ${HOME}/.terra/config/config.toml

# use mounted disk for datadir
rm -rf .terra/data
ln -s /chaindata/ .terra/data
```

* Return to sudo user  

```bash
exit
```

* Install `terrad` systemd unit file  

```bash
sudo bash -c "cat > /etc/systemd/system/terrad.service << EOF
[Unit]
Description=Terra Daemon
After=network.target

[Service]
Type=simple
User=terrauser
ExecStart=/home/terra/bin/terrad start
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target

[Service]
LimitNOFILE=65535
EOF"
```

* Start `terrad`  

```bash
sudo systemctl daemon-reload
sudo systemctl enable terrad
sudo systemctl start terrad

# check for errors
sudo journalctl -u terrad -f
```

* Become terra user again  

```bash
sudo su - terra
```  

* Make sure we are synched up (catching_up="false")  

```bash
terrad status
```  

* Create a bombay wallet  

```bash
terrad keys add bombay --keyring-backend os

# go to faucet.terra.money and add some funds
```  


* Since we are migrating from tequila we just simply unjail Validator (not needed if you are setting up fresh)  

```bash
terrad tx slashing unjail \
   --from tequila \
   --chain-id bombay-0005 \
   --gas-prices 1.5uluna \
   --gas-adjustment=1.4 \
   --keyring-backend os 
```
> * Here, "tequila" is the same wallet that was used to set up the original tequila testnet validator*   

* Otherwise we would have to register Validator like this.  This is just an example, we did not do this since we already had our validator registered on tequila  

```bash
terrad tx staking create-validator \
	--pubkey '{"@type":"/cosmos.crypto.ed25519.PubKey","key":"yCt9TJQqoZyu6BCfNlCcik/1djZgS7s8DLh9AgmrZh8="}' \
	--amount 1000000000uluna \
	--from bombay \
	--commission-rate 0.05 \
	--commission-max-rate 0.05 \
	--commission-max-change-rate 0 \
	--min-self-delegation 1 \
	--moniker RnodeC \
	--gas-prices 1.5uluna \
	--gas-adjustment 1.4 \
	--chain-id bombay-0005 \
	--keyring-backend os 
```  
 

## Oracle  

* Dependencies  

```bash
# g++
sudo yum install gcc-c++ -y

#newer version of git
sudo yum install http://opensource.wandisco.com/rhel/7/git/x86_64/wandisco-git-release-7-2.noarch.rpm -y
sudo rpm --import  http://opensource.wandisco.com/RPM-GPG-KEY-WANdisco
sudo yum upgrade git -y
sudo rm -f /etc/yum.repos.d/wandisco-git.repo #get rid of this insecure repo now that we have what we need from it

#node.js 
VERSION=v14.16.0
DISTRO=linux-x64
sudo mkdir -p /usr/local/lib/nodejs
wget -O - https://nodejs.org/dist/v14.16.0/node-$VERSION-$DISTRO.tar.xz | sudo tar xJ -C /usr/local/lib/nodejs

# make sure oracle user has npm in path
sudo bash -c "echo \"export PATH=/usr/local/lib/nodejs/node-$VERSION-$DISTRO/bin:\$PATH\" >> /home/oracle/.bashrc"
```  

* Create oracle user  

```bash
sudo useradd oracle
sudo su - oracle
```  

* Get oracle-feeder  

```bash
git clone https://github.com/terra-project/oracle-feeder.git
pushd oracle-feeder/feeder
git fetch --all --tags
git checkout --track remotes/origin/bombay
npm install 
npm start update-key
```  
> *we are assuming that price server is running elsewhere ("tequilaval")*  

* `run-feeder` script  

```bash
bash -c "cat > /home/oracle/run-feeder.sh << EOF
#!/bin/bash

source \$HOME/.bashrc

pushd \$HOME/oracle-feeder/feeder

npm start vote -- \
    --source http://tequilaval:8532/latest \
    --lcd http://localhost:1317 \
    --chain-id bombay-0005 \
    --denoms sdr,krw,usd,mnt,eur,cny,jpy,gbp,inr,cad,chf,hkd,aud,sgd,sek \
    --validator terravaloper1caz4p4zra6ssvxe9fre58625x8u0995y24lmqv \
    --password bombaytest \
    --gas-prices 169.77ukrw
EOF"
chmod +x /home/oracle/runfeeder.sh
```  


* Install feeder systemd unit file  

```bash
sudo bash -c "cat > /etc/systemd/system/feeder.service << EOF
[Unit]
Description=Terra Oracle Feeder
After=network.target

[Service]
Type=simple
User=oracle
WorkingDirectory=/home/oracle/oracle-feeder/feeder
ExecStart=/home/oracle/run-feeder.sh
Restart=on-failure
RestartSec=5s
Environment=\"PATH=/usr/local/lib/nodejs/node-v14.16.1-linux-x64/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin\"

[Install]
WantedBy=multi-user.target

[Service]
LimitNOFILE=65535
EOF"
```  

* Return to sudo user  

```bash
exit
```  

* Run feeder  

```bash
sudo systemctl daemon-reload
sudo systemctl enable feeder
sudo systemctl start feeder
```   

## Node Exporter

In order to monitor resource consumption...

https://prometheus.io/download/  


* Create user  

```bash
sudo useradd -rs /bin/false node_exporter
```  

* Download  

```bash
NODE_EXPORTER_VERS=1.1.2
wget -O - https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERS}/node_exporter-${NODE_EXPORTER_VERS}.linux-amd64.tar.gz | tar xz
```  

* Relocate Files  

```bash
sudo mv node_exporter-${NODE_EXPORTER_VERS}.linux-amd64/node_exporter /usr/local/bin
rm -rf node_exporter-${NODE_EXPORTER_VERS}.linux-amd64/
```  

* Systemd Unit File  

```bash
sudo bash -c "cat > /etc/systemd/system/node_exporter.service << EOF 
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF"

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```  

* Add this snippet to `/etc/prometheus/prometheus.yml` on prometheus machine

```bash
  - job_name: 'bombay_node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['3x.x.x.x0:9100']
```  