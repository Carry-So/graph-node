# Deploy from source

We will deploy 3 daemon processes in this tutorial: ipfs, PostgreSQL, graph-indexer. IPFs and graph indexer are started with the permission of the current user.

## 1. Install IPFS

### 1.1 Modify UDP Receive Buffer Size

Reference: https://github.com/lucas-clemente/quic-go/wiki/UDP-Receive-Buffer-Size

```shell
sudo sysctl -w net.core.rmem_max=2500000
echo -e "\nnet.core.rmem_max=2500000" | sudo tee -a /etc/sysctl.conf
```

### 1.2 Install IPFS

Reference: https://docs.ipfs.tech/install/command-line/#linux

```shell
wget https://dist.ipfs.tech/kubo/v0.17.0/kubo_v0.17.0_linux-amd64.tar.gz
tar -xvzf kubo_v0.17.0_linux-amd64.tar.gz
cd kubo
sudo bash install.sh
ipfs --version
```

### 1.3 Initialize ipfs

```shell
export IPFS_PATH="/var/lib/ipfs"
echo -e '\nexport IPFS_PATH="/var/lib/ipfs"' >> ${HOME}/.bashrc

USER=$(whoami)
sudo mkdir -p /var/lib/ipfs
sudo chown -R ${USER}:${USER} ${IPFS_PATH}
sudo chmod ug+rwx ${IPFS_PATH}

ipfs init
cp -p ${IPFS_PATH}/config ${IPFS_PATH}/config.bak
sed -i "s#/ip4/127.0.0.1/tcp/5001#/ip4/0.0.0.0/tcp/5001#g" ${IPFS_PATH}/config
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["*"]'
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["*"]'
```

### 1.4 Create service file

```shell
USER=$(whoami)
sudo tee /etc/systemd/system/ipfs.service <<-EOF
[Unit]
Description=InterPlanetary File System (IPFS) daemon
Documentation=https://docs.ipfs.tech/
After=network.target
Requires=network.target

[Service]
Type=simple
User=${USER}
Group=${USER}
Environment="IPFS_PATH=${IPFS_PATH}"
Environment="IPFS_FD_MAX=8192"
LimitNOFILE=8192
MemorySwapMax=0
# TimeoutStartSec=infinity
WorkingDirectory=${IPFS_PATH}
ExecStart=/usr/local/bin/ipfs daemon
RestartSec=10
Restart=always
KillSignal=SIGINT
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ipfs
sudo systemctl start ipfs
systemctl status ipfs

# {"Version":"0.17.0","Commit":"","Repo":"12","System":"amd64/linux","Golang":"go1.19.1"}
curl -X POST http://127.0.0.1:5001/api/v0/version
```

## 2. Install PostgreSQL

Reference: https://www.postgresql.org/download/linux/ubuntu/

- database account: graph
- password: daniel2022
- listen port: 5432
- database name:
  - mainnet(xinfin): xinfin_db
  - testnet(apothem): apothem_db

```shell
export DB_USER="graph"
export DB_PASSWORD="daniel2022"
```

### 2.1 Modify kernal parameter

```shell
cat /proc/sys/vm/swappiness
sudo bash -c "sysctl -w vm.swappiness=1"
echo -e "\nvm.swappiness=1" | sudo tee -a /etc/sysctl.conf
cat /proc/sys/vm/swappiness
```

### 2.2 Install chinese language support

```shell
sudo apt install -y lsb-core language-pack-zh*
```

### 2.3 Install database software

```shell
# Create the file repository configuration:
sudo sh -c 'echo "deb [arch=$(dpkg --print-architecture)] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
# wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/postgresql.asc

# Update the package lists:
sudo apt update

# Install PostgreSQL 14
sudo apt install -y postgresql-14 postgresql-client-14 postgresql-server-dev-14 libpq-dev libpq5
```

### 2.4 Change some parameters

```shell
sudo tee /etc/postgresql/14/main/conf.d/liudan.conf <<-\EOF
listen_addresses = '*'
port = 5432

tcp_keepalives_idle = 300
tcp_keepalives_interval = 20
tcp_keepalives_count = 9

log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_line_prefix = '%m:%r:%u@%d:[%p]: '
log_rotation_age = 1d
lc_messages = 'en_US.UTF-8'

shared_preload_libraries = 'pg_stat_statements'
EOF
```

### 2.5 Change password of linux account postgres

```shell
echo postgres:${DB_PASSWORD} | sudo chpasswd
```

### 2.6 Start database

```shell
# pg_ctlcluster 14 main start
sudo systemctl daemon-reload
sudo systemctl restart postgresql@14-main
systemctl status postgresql@14-main
```

### 2.7 Change password of database account postgres

```shell
sudo -u postgres psql <<-EOF
ALTER ROLE postgres WITH PASSWORD '${DB_PASSWORD}';
EOF
```

### 2.8 Create database user graph

```shell
PGPASSWORD=${DB_PASSWORD} psql -X -h 127.0.0.1 -U postgres postgres <<-EOF
CREATE USER ${DB_USER} WITH PASSWORD '${DB_PASSWORD}' nocreatedb;
EOF
```

### 2.9 Create database for mainnet and testnet

```shell
for NETWORK in xinfin apothem; do
  echo "Create ${NETWORK}_db ..."
  PGPASSWORD=${DB_PASSWORD} psql -X -h 127.0.0.1 -U postgres postgres <<-EOF
  CREATE DATABASE ${NETWORK}_db WITH OWNER=${DB_USER} TEMPLATE=template0 LC_COLLATE='zh_CN.UTF8' LC_CTYPE='zh_CN.UTF8' ENCODING='utf8';
EOF
  echo
done
```

### 2.10 Create extension

We will create 3 extensions: pg_trgm、pg_stat_statements、btree_gist、postgres_fdw

```shell
for NETWORK in xinfin apothem; do
  echo "Create extensions for ${NETWORK}_db ..."
  PGPASSWORD=${DB_PASSWORD} psql -X -h 127.0.0.1 -U postgres ${NETWORK}_db <<-EOF
    CREATE EXTENSION pg_trgm;
    CREATE EXTENSION pg_stat_statements;
    GRANT ALL ON FUNCTION pg_stat_statements_reset TO ${DB_USER};
    CREATE EXTENSION btree_gist;
    CREATE EXTENSION postgres_fdw;
    GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO ${DB_USER};
EOF
  echo
done
```

### 2.11 Setup environment for psql

```shell
cat >> ${HOME}/.bashrc <<-EOF

export PGHOST=127.0.0.1
export PGPORT=5432
export PGDATABASE=apothem_db
export PGUSER=${DB_USER}
export PGPASSWORD=${DB_PASSWORD}
EOF

source ${HOME}/.bashrc

cat > ${HOME}/.psqlrc <<-\EOF
\set PROMPT1 '%`date +%H:%M:%S` (%n@%M:%>)%/%R%#%x '
\set PROMPT2 '%M %n@%/%R%# '
\timing on
EOF
```

## 3. Install graph-node

Reference:

- https://github.com/graphprotocol/graph-node/blob/master/docs/config.md
- https://github.com/graphprotocol/graph-node/blob/master/node/resources/tests/full_config.toml
- Running a Graph Node on Moonbeam: https://docs.moonbeam.network/node-operators/indexer-nodes/thegraph-node/
- Deploy and Configure Graph-node: https://docs.thegraph.academy/official-docs/indexer/testnet/graph-protocol-testnet-baremetal/3_deployandconfiguregraphnode

### 3.1 Create log file directory

```shell
USER=$(whoami)
sudo mkdir -p /var/log/graph
sudo chown -R ${USER}:${USER} /var/log/graph
```

### 3.2 Create config file directory

```shell
USER=$(whoami)
sudo mkdir -p /etc/graph-node
sudo chown -R ${USER}:${USER} /etc/graph-node
touch /etc/graph-node/expensive-queries.txt
```

### 3.3 Install Rust

Install or update Rust language:

```shell
cargo -V
if [ "$?" == 0 ] ; then
    rustup update stable
else
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    source ${HOME}/.cargo/env
    cargo -V
fi
```

### 3.4 Download and compile graph-node

```shell
sudo apt install -y cmake

cd ${HOME}
git clone https://github.com/Carry-So/graph-node graph-node.xdc
cd graph-node.xdc
git checkout xdc-release-0.30.0
cargo build --release
```

### 3.5 Create start script

```shell
cat > ${HOME}/start-graph-indexer.sh <<-\EOF
#!/bin/bash

CHAIN="$1"
LOG_FILE="/var/log/graph/indexer-${CHAIN}-`/usr/bin/date +%Y%m%d`.log"

# export GRAPH_LOG="trace"
# export GRAPH_LOG="debug"
export ETHEREUM_REORG_THRESHOLD=1
export GRAPH_LOG_TIME_FORMAT="%Y-%m-%d %H:%M:%S"

${HOME}/graph-node.xdc/target/release/graph-node \
  --node-id indexer_${CHAIN} \
  --postgres-url postgresql://DB_USER:DB_PASSWORD@localhost:5432/${CHAIN}_db \
  --ethereum-rpc ${CHAIN}:traces,archive,no_eip1898:https://arpc.${CHAIN}.network \
  --ipfs 127.0.0.1:5001 \
  --expensive-queries-filename /etc/graph-node/expensive-queries.txt \
  >> ${LOG_FILE} 2>&1
EOF

sed -i "s/DB_USER/${DB_USER}/g" ${HOME}/start-graph-indexer.sh
sed -i "s/DB_PASSWORD/${DB_PASSWORD}/g" ${HOME}/start-graph-indexer.sh
```

chmod +x ${HOME}/start-graph-indexer.sh

### 3.6 Create service file for testnet and mainnet

```shell
USER=$(whoami)
sudo tee /usr/lib/systemd/system/graph-indexer@.service <<EOF
[Unit]
Description=Graph indexer node %i
After=network.target ipfs.service
Wants=network.target ipfs.service

[Service]
User=${USER}
Group=${USER}
WorkingDirectory=/home/${USER}/
StandardOutput=journal
StandardError=journal
Type=simple
Restart=always
RestartSec=5
ExecStart=/home/${USER}/start-graph-indexer.sh %i

[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reload
```

### 3.7 Start indexer service

Start indexer service according to your network.

#### 3.7.1 Start indexer service for testnet

```shell
sudo systemctl enable graph-indexer@apothem
sudo systemctl start graph-indexer@apothem
systemctl status graph-indexer@apothem
```

#### 3.7.2 Start indexer service for mainnet

```shell
sudo systemctl enable graph-indexer@xinfin
sudo systemctl start graph-indexer@xinfin
systemctl status graph-indexer@xinfin
```

### 3.8 Scheduled restart indexer service

#### 3.8.1 List current crontab jobs

`sudo crontab -l -u root`

#### 3.8.2 Edit crontab jobs

`sudo crontab -e -u root`

#### 3.8.3 Add new crontab job

- for mainnet(xinfin): `0 0 * * * /usr/bin/systemctl restart graph-indexer@xinfin.service`
- for testnet(apothem): `0 0 * * * /usr/bin/systemctl restart graph-indexer@apothem.service`

#### 3.8.4 Check crontab jobs

`sudo crontab -l -u root`

## 4. TODO

There are some follow-up tasks for system operators, such as:

- Deploy IPFS private cluster and replace swarm key
- Make pg database high availability
- Add some query nodes of graph-node to achive higher qps
- Add user authentication, make users can only delete their own subgraph
- Fetch and show subgraph status from log files
