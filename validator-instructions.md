# New Validator Instructions
Stuck?
Try the FAQ first, then search in the Developer Discord, then ask a question or open a ticket in #support-ticket.

## Requirements
warning
The below hardware requirements have been tested on bare metal servers. Cloud solutions are currently not supported.

CPU
16 cores or more
4.5GHz base clock speed or faster (single core performance)
CPU Architecture
AMD Zen 4 or newer
Intel Raptor Lake or newer
Ex. AMD Ryzen 7950x / Ryzen 9950x / EPYC 4584PX
RAM
At least 32GB
Disk
2TB Dedicated disk for TrieDB (Execution)
2TB Disk for MonadBFT / OS
PCIe Gen4x4 NVME SSD or better
warning
Hard drive performance can vary dramatically by manufacturer. Below are results from internal testing:

Ranked performance

Samsung 980 / 990 Pro - PCIe 4.0, top class performance
Samsung PM9A1 - PCIe 4.0, pretty good performance and stable performance under load
Micron 7450 - PCIe 4.0, pretty good performance BUT has weird random slowdowns under a lot of load
Known unreliable

Nextorage SSDs - can become unresponsive under load due to overheating, requiring a system reboot.
Components
monad-bft - Consensus client
monad-execution - Execution client
monad-rpc - RPC client
monad-mpt - One-time execution for initializing the disk for TrieDB making the disk available for it. Not required when upgrading unless explicitly mentioned
monad-execution-genesis - One-time execution for intializing the first block produced by the chain. (Block 0). Not required if the node is not part of genesis validators.
Prerequisites
Ubuntu 24.04+ OS and Kernel >= 6.8 (see warning below)
Disable HyperThreading or SMT via bios settings
warning
There is a known bug affecting Linux kernel versions v6.8.0.56-generic - v6.8.0.59-generic (inclusive) that causes Monad clients to hang in an uninterruptible sleep state, severely impacting node stability. We recommend v6.8.0.60-generic or higher.

## Prepare the Node
info
Instructions below assume you are running from root user

### Provision TrieDB Disk
Set the drive to be used for TrieDB and create a new partition table and a partition that spans the entire drive (the drive should be on a disk that has no filesystem mounted and no RAID)

TRIEDB_DRIVE=/dev/nvme1n1 # CHANGE THIS TO YOUR NVME DRIVE

parted $TRIEDB_DRIVE mklabel gpt
parted $TRIEDB_DRIVE mkpart triedb 0% 100%

Create a udev rule to set permissions and create a symlink for the partition

PARTUUID=$(lsblk -o PARTUUID $TRIEDB_DRIVE | tail -n 1)
echo "ENV{ID_PART_ENTRY_UUID}==\"$PARTUUID\", MODE=\"0666\", SYMLINK+=\"triedb\"" | tee /etc/udev/rules.d/99-triedb.rules


Trigger and reload udev rules

udevadm trigger
udevadm control --reload

Give permissions to the /dev/triedb drive

chmod a+rwx /dev/triedb

Check LBA Configuration
Define TRIEDB_DRIVE, enable 512 byte LBA if not enabled, create a new partition table and a partition that spans the entire drive. The TrieDB drive should be on a disk that has no filesystem mounted and no RAID configured.

TRIEDB_DRIVE=/dev/nvme1n1 # CHANGE THIS TO YOUR NVME DRIVE

Install nvme-cli which is used to check for 512 byte LBA.

apt-get install nvme-cli

Check if 512 byte LBA is enabled on TRIEDB_DRIVE.

nvme id-ns -H $TRIEDB_DRIVE | grep 'LBA Format' | grep 'in use'

This command should return the following expected output.

LBA Format  0 : Metadata Size: 0   bytes - Data Size: 512 bytes - Relative Performance: 0 Best (in use)


info
Data Size should be set to 512 bytes and marked as (in use). If that is not the case, then you will need to set the TRIEDB_DRIVE to use 512 byte LBA with the following command.

nvme format --lbaf=0 $TRIEDB_DRIVE

Verify that the configuration has been corrected.

nvme id-ns -H $TRIEDB_DRIVE | grep 'LBA Format' | grep 'in use'

## Create Dedicated User
Create a non-privileged user named monad with a home directory and Bash shell.

useradd -m -s /bin/bash monad

Create config directories in monad-bft in monad's home directory.

sudo -u monad bash -c "
mkdir -p ~/monad-bft/config \
        ~/monad-bft/blockdb \
        ~/monad-bft/ledger \
        ~/monad-bft/config/forkpoint
"

info
~/monad-bft/config/node.toml - contains configurable consensus parameters, most notably the name of the node, typically " <INSERT_VALIDATOR_NAME>" and a list of peers (denoted by secp pubkey and DNS) whitelisted for statesync from your node
~/monad-bft/config/forkpoint/ - contains consensus quorum checkpoints (written every block) used for bootstrapping state
~/monad-bft/config/validators.toml - contains the active and future validator sets and needs to be updated when changes to the validator set are staged
~/monad-bft/blockdb - contains "consensus blocks" (quorum certificates, transaction data, etc.)
~/monad-bft/ledger - contains "eth blocks" encoded in the standard Ethereum block format, used as input to execution, which updates triedb with modified state and MPT
Add APT Repo
cat <<EOF > /etc/apt/sources.list.d/category-labs.sources
Types: deb
URIs: https://pkg.category.xyz/
Suites: noble
Components: main
Signed-By: /etc/apt/keyrings/category-labs.gpg
EOF

## Add APT Auth
cat <<EOF > /etc/apt/auth.conf.d/category-labs.conf
machine https://pkg.category.xyz/
login pkgs
password af078cc4157b45f29e279879840fd260
EOF

## Add GPG Key
curl -fsSL https://pkg.category.xyz/keys/public-key.asc | gpg --dearmor -o /etc/apt/keyrings/category-labs.gpg


## Install Monad
apt update
apt install monad=0.1.0-testnet-2-genesis

## Configure UFW
Configure UFW to allow SSH inbound connections (remote access) and traffic to port 8000. The Monad consensus client uses both TCP and UDP. Also, allow access to port 8889 from Monad Foundation server (84.32.32.227) to scrape chain metrics.

```
# allow ssh connections
sudo ufw allow ssh
# allow p2p port
sudo ufw allow 8000
# allow MF Monitoring server to scrape chain metrics
sudo ufw allow from 84.32.32.227 proto tcp to any port 8889
# enable ufw
# default behavior: block all incoming traffic and enable all outgoing
sudo ufw enable
```

Hardware firewalls
If using hardware firewalls, you may need to perform additional steps to open up port 8000 to UDP and TCP traffic and port 8889 to allow Monad Foundation to scrape chain metrics.

## Add Configuration Files
 - Create backup folder for BLS and SECP keys
mkdir -p /opt/monad/backup

 - Make monad owner of the backup directory
chown -R monad:monad /opt/monad/backup

info
Instructions below assume you are running from monad user

Download and update ~/.env
CL_BUCKET=https://pub-b0d0d7272c994851b4c8af22a766f571.r2.dev
curl -o ~/.env $CL_BUCKET/config/testnet-2/latest/.env.example

In .env, type a strong password in KEYSTORE_PASSWORD. This password should be wrapped in single quotes, e.g. KEYSTORE_PASSWORD='str0ngp@ssw0rd' and is used for key encryption / decryption.

## Key Management
Validator nodes need two keystores to be generated in order for the node to run.

BLS Key
SECP Key
Generate encrypted BLS and SECP keys using the keystore binary:

// source the .env file which will load the KEYSTORE_PASSWORD
source /home/monad/.env

// Create the SECP key
monad-keystore create \
--key-type secp \
--keystore-path ~/monad-bft/config/id-secp \
--password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/secp-backup

// Create the BLS key
monad-keystore create \
--key-type bls \
--keystore-path ~/monad-bft/config/id-bls \
--password "${KEYSTORE_PASSWORD}" > /opt/monad/backup/bls-backup

Keypairs will be generated and backups stored locally under secp-backup and bls-backup. Please make sure keys are properly backed up.

Save a single file backup of both keys:

// Create a backup of both keys in /home/monad
grep "public key" /opt/monad/backup/secp-backup /opt/monad/backup/bls-backup > /home/monad/pubkey-secp-bls


## Checkpoint
Please provide public keys and DNS to the Monad Foundation team via this form and wait for confirmation to proceed.

Download testnet-2 configuration files
Note: these files will only be finalized after all validators have submitted the above Checkpoint information to Monad Foundation.

MF_BUCKET=https://bucket.monadinfra.com
curl -o ~/monad-bft/config/node.toml $MF_BUCKET/config/testnet-2/latest/node.toml
curl -o ~/monad-bft/config/validators.toml $MF_BUCKET/config/testnet-2/latest/validators.toml


## Launch the Monad Stack
Set your node_name
Update the node_name field in node.toml to include validator name and remove '#' from the variable

node_name = # "<INSERT_VALIDATOR_NAME>"

## Get latest forkpoint
Fetch the latest forkpoint (a new forkpoint is served every minute) and statesync the node to latest height.

MF_BUCKET=https://bucket.monadinfra.com
curl -sSL $MF_BUCKET/scripts/testnet-2/download-forkpoint.sh | bash

The final directory structure should look similar to this:

/home/monad
    └── .env
    └── monad-bft/
        └──config
            ├── id-bls
            ├── id-secp
            └── node.toml
            └── validators.toml
            └──forkpoint
                └── forkpoint.toml

## Initialize the database
systemctl start monad-mpt

Check journalctl to ensure that this worked correctly.

Below is an example of a successful outcome:

$ journalctl -u monad-mpt

# NOTE: output is "trimmed" for easier reading here
MPT database on storages:
          Capacity           Used      %  Path
           1.82 Tb        1.85 Gb  0.10%  "/dev/nvme1n1p1"
MPT database internal lists:
     Fast: 7 chunks with capacity 1.75 Gb used 1.60 Gb
     Slow: 1 chunks with capacity 256.00 Mb used 0.00 bytes
     Free: 7441 chunks with capacity 1.82 Tb used 0.00 bytes
MPT database has 281868 history, earliest is 0 latest is 281867.
     It has been configured to retain no more than 33554432.
     Latest voted is (281866, 281868).
     Latest finalized is 281865, latest verified is 281862, auto expire version >

Run Genesis OR Hard Reset
Perform the Genesis instructions below only if your node is part of the chain's genesis event. Otherwise, follow the Hard Reset instructions.

## Genesis
Start the stack
Run systemctl commands to start the node:

systemctl start monad-bft monad-execution monad-rpc

Verify the systemd services are running:

systemctl status monad-bft monad-execution monad-rpc

# Check logs for each service
journalctl -fu monad-bft
journalctl -fu monad-execution
journalctl -fu monad-rpc

View available CLI arguments
Execute the CLI --help command via the desired binary. Please note that these should not be changed arbitrarily as some configurations may result in unexpected behavior or crashes.

monad-rpc --help

Validator set updates
Please note that a validator will successfully join consensus after the validator keypairs are added to a future epoch as defined in validators.toml by a stake-weighted supermajority of the active validator set. Once the defined epoch starts, the validator will participate in consensus.

OTEL Collector usage
Install & Start OTEL Collector
curl -fsSL -o /tmp/otelcol_0.125.0_linux_amd64.deb https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.125.0/otelcol_0.125.0_linux_amd64.deb
dpkg -i /tmp/otelcol_0.125.0_linux_amd64.deb
cp /opt/monad/scripts/otel-config.yaml /etc/otelcol/config.yaml

systemctl start otelcol


In the latest systemd file, support was added for the OTEL collector. Through this, you will be able to see all the relevant Monad-specific metrics (available at 0.0.0.0:8889/metrics), while also pushing metrics to a Category Labs OTLP endpoint so that the CL team can help with debugging if any issues arise.
