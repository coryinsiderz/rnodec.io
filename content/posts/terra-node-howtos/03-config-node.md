+++
title = "Terraform-Validator-03: Create Full Node"
description = "How to configure and run a terraform full node"
date = 2021-04-09
weight = 20
draft = false
+++

Whether this machine is a Validator or just a sentry, the setup is mostly common.


## Create terrauser
```bash
sudo useradd terrauser
```

## Update OS and add a few packages
```bash
sudo yum update -y
sudo yum install jq git wget make gcc -y
```

## Adjust nofiles kernel param
```bash
sudo bash -c "cat > /etc/security/limits.d/terrad.conf << EOF
# /etc/security/limits.conf

*                soft    nofile          65535
*                hard    nofile          65535
EOF"
```

## Install go
```bash
wget https://golang.org/dl/go1.16.3.linux-amd64.tar.gz -O - | tar xz 
sudo mv go /usr/local
export PATH=/usr/local/go/bin:$PATH
echo "export PATH=/usr/local/go/bin:$PATH" >> /home/terrauser/.bashrc
```


## Install terrad

```bash
git clone https://github.com/terra-project/core.git
pushd core
PATH=$PATH:/usr/local/go/bin GOBIN=$(pwd)/bin make install
sudo mv bin/terra* /usr/local/bin
popd
```



## Install terrad's systemd unit file
```bash
sudo bash -c "cat > /etc/systemd/system/terrad.service << EOF
[Unit]
Description=Terra Daemon
After=network.target

[Service]
Type=simple
User=terrauser
ExecStart=/usr/local/bin/terrad start
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target

[Service]
LimitNOFILE=65535
EOF"
```

## Install a systemd unit file for LCD as well
```bash
sudo bash -c "cat > /etc/systemd/system/lcd.service << EOF
[Unit]
Description=Terra CLI LCD Service
After=network.target

[Service]
Type=simple
User=terrauser
ExecStart=/usr/local/bin/terracli rest-server --chain-id=$NETWORK --laddr=tcp://0.0.0.0:1317 --node tcp://localhost:26657 --trust-node=true
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target

[Service]
LimitNOFILE=65535
EOF"
```
* `NETWORK`: `tequila-0004`, `columbus-4`, etc

## Prepare daemons for launch
```bash
sudo systemctl daemon-reload
sudo systemctl enable terrad
sudo systemctl enable lcd
```

## Mount attached disk
It is no fun to catch up to the chain from scratch.  You should have already [downloaded latest chain](./get-chain.md) and attached a disk with it to this instance. 
```bash
sudo mkdir /terradata
sudo mount -t ext4 /dev/sdb /terradata
sudo chown -R terrauser:terrauser /terradata
```