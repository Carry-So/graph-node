# How to deploy graph-node for XinFin blockchain

Original documents:

- [README](./README.original.md)
- [docs](./docs/)

We will deploy 3 daemon processes in this tutorial: ipfs, PostgreSQL, graph-indexer. IPFs and graph indexer are started with the permission of the current user.

## 1. Install IPFS

### 1.1 Modify UDP Receive Buffer Size

Reference: https://github.com/lucas-clemente/quic-go/wiki/UDP-Receive-Buffer-Size

```shell
sudo sysctl -w net.core.rmem_max=2500000
echo "net.core.rmem_max=2500000" | sudo tee -a /etc/sysctl.conf
```

### 1.2 Install IPFS

Reference: https://docs.ipfs.io/install/command-line/#official-distributions

```shell
wget https://dist.ipfs.io/kubo/v0.14.0/kubo_v0.14.0_linux-amd64.tar.gz
tar -xvzf kubo_v0.14.0_linux-amd64.tar.gz
cd kubo
sudo bash install.sh
ipfs --version
```

### 1.3 Initialize ipfs

```shell
export IPFS_PATH="/var/lib/ipfs"
echo 'export IPFS_PATH="/var/lib/ipfs"' >> ${HOME}/.bashrc

USER="`whoami`"
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
sudo tee /etc/systemd/system/ipfs.service <<EOF
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
sudo systemctl status ipfs
curl -X POST http://127.0.0.1:5001/api/v0/version
```

## 2. Install PostgreSQL

Reference: https://www.postgresql.org/download/linux/ubuntu/

- database account: graph
- password: daniel2022
- listen port: 5432
- database name: apothem_db for testnet; xinfin_db for mainnet

### 2.1 Modify kernal parameter

```shell
cat /proc/sys/vm/swappiness
sudo bash -c "sysctl -w vm.swappiness=1"
echo "vm.swappiness=1" | sudo tee -a /etc/sysctl.conf
cat /proc/sys/vm/swappiness
```

### 2.2 Install chinese language support

```shell
sudo apt install -y language-pack-zh*
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
sudo apt install -y postgresql-14 postgresql-client-14 postgresql-server-dev-14 libpq-dev
```

### 2.4 Backup original config files

```shell
cd /etc/postgresql/14/main
sudo cp -p pg_hba.conf pg_hba.conf.`date +%Y%m%d-%H%M%S`
sudo cp -p postgresql.conf postgresql.conf.`date +%Y%m%d-%H%M%S`
```

### 2.5 Recreate config file pg_hba.conf

```shell
sudo tee /etc/postgresql/14/main/pg_hba.conf <<-\EOF
# Database administrative login by Unix domain socket
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
EOF
```

### 2.6 Change some parameters

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

### 2.7 Change password of linux account postgres

```shell
echo postgres:daniel2022 | sudo chpasswd
```

### 2.8 Start database

```shell
# pg_ctlcluster 14 main start
sudo systemctl daemon-reload
sudo systemctl restart postgresql@14-main
systemctl status postgresql@14-main
```

### 2.9 Change password of database account postgres

```shell
su - postgres
psql -c "ALTER ROLE postgres WITH PASSWORD 'daniel2022';"
exit
```

### 2.10 Create database

#### 2.10.1 Create database user graph

```shell
PGPASSWORD=daniel2022 psql -X -h 127.0.0.1 -U postgres postgres <<-\EOF
CREATE USER graph WITH PASSWORD 'daniel2022' nocreatedb;
EOF
```

#### 2.10.2 Create database apothem_db for testnet

```shell
PGPASSWORD=daniel2022 psql -X -h 127.0.0.1 -U postgres postgres <<-\EOF
CREATE DATABASE apothem_db WITH OWNER=graph TEMPLATE=template0 LC_COLLATE='zh_CN.UTF8' LC_CTYPE='zh_CN.UTF8' ENCODING='utf8';
EOF
```

#### 2.10.3 Create database xinfin_db for mainnet

```shell
PGPASSWORD=daniel2022 psql -X -h 127.0.0.1 -U postgres postgres <<-\EOF
CREATE DATABASE xinfin_db WITH OWNER=graph TEMPLATE=template0 LC_COLLATE='zh_CN.UTF8' LC_CTYPE='zh_CN.UTF8' ENCODING='utf8';
EOF
```

### 2.11 Create extension

We will create 3 extensions: pg_trgm、pg_stat_statements、btree_gist、postgres_fdw

#### 2.11.1 Create extension for apothem testnet

```shell
PGPASSWORD=daniel2022 psql -X -h 127.0.0.1 -U postgres apothem_db <<-\EOF
CREATE EXTENSION pg_trgm;
CREATE EXTENSION pg_stat_statements;
GRANT ALL ON FUNCTION pg_stat_statements_reset TO graph;
CREATE EXTENSION btree_gist;
CREATE EXTENSION postgres_fdw;
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO graph;
EOF
```

#### 2.11.2 Create extension for xinfin mainnet

```shell
PGPASSWORD=daniel2022 psql -X -h 127.0.0.1 -U postgres xinfin_db <<-\EOF
CREATE EXTENSION pg_trgm;
CREATE EXTENSION pg_stat_statements;
GRANT ALL ON FUNCTION pg_stat_statements_reset TO graph;
CREATE EXTENSION btree_gist;
CREATE EXTENSION postgres_fdw;
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO graph;
EOF
```

### 2.12 Setup environment for psql

```shell
cat >> ${HOME}/.bashrc <<-\EOF
export PGHOST=127.0.0.1
export PGPORT=5432
export PGDATABASE=apothem_db
export PGUSER=graph
export PGPASSWORD=daniel2022
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
USER="`whoami`"
sudo mkdir -p /var/log/graph
sudo chown -R ${USER}:${USER} /var/log/graph
```

### 3.2 Create config file directory

```shell
USER="`whoami`"
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
fi
cargo -V
```

### 3.4 Download and compile graph-node

```shell
sudo apt install -y cmake

cd ${HOME}
git clone https://github.com/Carry-So/graph-node graph-node.xdc
cd graph-node.xdc
git checkout xdc-release-0.28.0
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
  --postgres-url postgresql://graph:daniel2022@localhost:5432/${CHAIN}_db \
  --ethereum-rpc ${CHAIN}:traces,archive,no_eip1898:https://arpc.${CHAIN}.network \
  --ipfs 127.0.0.1:5001 \
  --expensive-queries-filename /etc/graph-node/expensive-queries.txt \
  >> ${LOG_FILE} 2>&1
EOF
```

chmod +x ${HOME}/start-graph-indexer.sh

### 3.6 Create service file for testnet and mainnet

```shell
USER="`whoami`"
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

- for testnet: `0 0 * * * /usr/bin/systemctl restart graph-indexer@apothem.service`
- for mainnet: `0 0 * * * /usr/bin/systemctl restart graph-indexer@xinfin.service`

#### 3.8.4 Check crontab jobs

`sudo crontab -l -u root`

## 4. Query

An example is developed at: http://103.101.129.136:8000/subgraphs/name/gzliudan/bad-token-subgraph-apothem, you can use below query command:

```graphql
{
  erc20Contracts(first: 5) {
    id
    name
    symbol
    decimals
    totalSupply {
      value
      valueExact
    }
  }
  blackLists(first: 5) {
    id
    members {
      account {
        id
      }
    }
  }
  accounts(first: 5) {
    id
    isErc20
    blackLists {
      id
    }
    Erc20balances {
      id
      value
      valueExact
    }
  }
}
```

This example is deployed on subgraph hosted service also, since better UI expeience: https://thegraph.com/hosted-service/subgraph/gzliudan/bad-token-mumbai.

## 5. TODO

There are some follow-up tasks for system operators, such as:

- Deploy IPFS private cluster and replace swarm key
- Make pg database high availability
- Add some query nodes of graph-node to achive higher qps
- Add user authentication, make users can only delete their own subgraph
- Fetch and show subgraph status from log files
