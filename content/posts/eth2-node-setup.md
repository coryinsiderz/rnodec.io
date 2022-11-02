---
title: Eth2 Node Setup
tags:
  - how-to
  - eth
date: 2021-11-22 
---

These are really for my own reference, not supposed to be a script to follow, but feel free to peruse.  

## Node Lockdown / Initial Configs

* modify ssh settings - change port, create sudo user and authorized keys, no rootlogin, no password  

* enable `firewalld`, change port in `/usr/lib/firewalld/services/ssh.xml`, restart `firewalld`  
```bash
sudo systemctl enable firewalld --now
sudo sed -i "s|22|$SSHPORT|" /usr/lib/firewalld/services/ssh.xml
sudo firewall-cmd --zone=public --set-target=DROP --runtime-to-permanent
sudo systemctl restart firewalld 

# open up to my ip
sudo firewall-cmd --zone=trusted --add-source=${MYIP}/32 --permanent
sudo firewall-cmd --reload 

# check active configuration
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```  

* System update   
```bash
sudo dnf update -y
sudo yum install -y git jq
```  

* Ensure chronyd or ntpd is here

* `go` install (need for `geth`)
```bash
wget https://golang.org/dl/go1.16.4.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.16.4.linux-amd64.tar.gz
```  

* Install `rust`
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

* stripe, format and mount disks (obviously changes with a different disk configuration)
```bash
sudo dnf install mdadm -y
for i in sdb sdc; do
  sudo dd if=/dev/zero bs=1M count=100 of=/dev/$i
  sudo wipefs -a /dev/$i
  sudo mdadm --zero-superblock --force /dev/$i
done

for i in sdb sdc; do
  sudo parted --script /dev/$i "mklabel gpt"
  sudo parted --script /dev/$i "mkpart primary 0% 100%"
  sudo parted --script /dev/$i "set 1 raid on"
done
  
sudo mdadm --create /dev/md0 --level=stripe --raid-devices=2 /dev/sd[b-c]1

sudo mdadm --detail /dev/md0

sudo mkfs.ext4 /dev/md0

sudo mkdir /mnt/raid0

echo "/dev/md0 /mnt/raid0 ext4 defaults 0 0" | sudo tee -a /etc/fstab

sudo mount -a 

# to undo...
sudo umount /mnt/raid0
sudo mdadm --stop /dev/md0
sudo mdadm --remove /dev/md0
```

## node_exporter (https://github.com/prometheus/node_exporter/releases)

```bash 
NODE_EXPORTER_VERS=1.1.2

sudo useradd -rs /bin/false node_exporter

wget -O - https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERS}/node_exporter-${NODE_EXPORTER_VERS}.linux-amd64.tar.gz | tar xz

sudo mv node_exporter-${NODE_EXPORTER_VERS}.linux-amd64/node_exporter /usr/local/bin

rm -rf node_exporter-${NODE_EXPORTER_VERS}.linux-amd64/

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

# start daemon
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# open port if prometheus is going to need to pull metrics from this machine remotely.  If prometheus is going to be running locally, forget this
***DONT OPEN THIS PORT ON A MACHINE YOU CARE ABOUT***
sudo firewall-cmd --zone=public --add-port=9100/tcp --permanent
sudo firewall-cmd --reload 
sudo systemctl status node_exporter
```


## geth (eth1)

* `geth` binary download from: https://geth.ethereum.org/downloads/
```bash
wget https://gethstore.blob.core.windows.net/builds/geth-linux-amd64-1.10.3-991384a7.tar.gz -O - | tar xz
sudo mv geth-linux-amd64-*/geth /usr/local/bin
```

* Create geth user
```bash
sudo useradd --system --no-create-home --shell /bin/false geth
```

* Datadir
```bash
sudo chown -R geth:geth /mnt/raid0/geth
```

* Systemd service
```bash
sudo bash -c "cat > /etc/systemd/system/geth.service << EOF 
[Unit]
Description=Eth1 mainnet
After=network.target 
Wants=network.target

[Service]
User=geth 
Group=geth
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/geth --http --datadir /mnt/raid0/geth/mainnet --metrics

[Install]
WantedBy=default.target
EOF"
```

* Run daemon
```bash
sudo systemctl daemon-reload
sudo systemctl enable geth --now
```


## lighthouse (beacon)

* Install `lighthouse` from:  https://github.com/sigp/lighthouse/releases
```bash
# wget https://github.com/sigp/lighthouse/releases/download/v1.3.0/lighthouse-v1.3.0-x86_64-unknown-linux-gnu.tar.gz -O - | tar xz
# using portable version because of old leaseweb cpu
wget https://github.com/sigp/lighthouse/releases/download/v1.3.0/lighthouse-v1.3.0-x86_64-unknown-linux-gnu-portable.tar.gz -O - | tar xz
sudo mv lighthouse /usr/local/bin
```

* Create lighthouse beacon user
```bash
sudo useradd --system --no-create-home --shell /bin/false lhbn
```

* Datadir 
```bash
sudo chown -R lhbn:lhbn /mnt/raid0/lighthouse/pyrmont/beacon 
```  

* Systemd unit file
```bash
sudo bash -c "cat > /etc/systemd/system/lhbn.service << EOF 
[Unit]
Description=Lighthouse Beacon Node mainnet
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=lhbn
Group=lhbn
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse --network mainnet bn  --datadir /mnt/raid0/lighthouse/mainnet --http  --eth1-endpoints http://localhost:8545,<any additional eth1 endpoints such as infura> --metrics 

[Install]
WantedBy=multi-user.target
EOF"
```

* Run daemon
```bash
sudo systemctl daemon-reload
sudo systemctl enable lhbeacon --now
```

## lighthouse (validator) 

* Create lighthouse validator user
```bash
sudo useradd --system --no-create-home --shell /bin/false lhvc
```

* Config dirs
```bash
# add "voting-keystore.json" file from eth2-deposit-clie to {cwd}/validator_keys...
sudo mkdir -p /mnt/raid0/lighthouse/mainnet/validators
sudo chown -R lhvc:lhvc /mnt/raid0/lighthouse/mainnet/validators
```   

* Systemd unit file
```bash
sudo bash -c "cat > /etc/systemd/system/lhvc.service << EOF 
[Unit]
Description=Lighthouse Validator mainnet
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=lhvc
Group=lhvc
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/lighthouse --network mainnet vc --datadir /mnt/raid0/lighthouse/mainnet --graffiti "RnodeC" --beacon-nodes http://localhost:5052,<any additional eth1 endpoints such as infura> --metrics

[Install]
WantedBy=multi-user.target
EOF"
```

* Run daemon
```bash
sudo systemctl daemon-reload
sudo systemctl enable lhvc --now
```

## Create keys & deposit data

* Follow launchpad:  https://pyrmont.launchpad.ethereum.org

* Download key management tool from:  https://github.com/ethereum/eth2.0-deposit-cli/releases/

* Create keys, keystores, depositdata
```bash
cd eth2deposit-cli-256ea21-linux-amd64
./deposit new-mnemonic --num_validators 1 --chain mainnet

# save mnemonic, signing key, and deposit data
```




## Import keys into validator client

```bash
sudo -u lhvc bash -c "lighthouse --network mainnet account validator import --keystore /tmp/keystore-m_12381_3600_0_0_0-1625181327.json --datadir /mnt/raid0/lighthouse/mainnet"
```