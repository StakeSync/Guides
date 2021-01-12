# Installation Guide - Cardano Node + DB Sync + Smash

Last Update: 2021-01-12  --  Cardano Node 1.24.2, DB Sync 7.1, Smash 1.3

This is a simple tutorial to set up the latest Cardano Node with DB Sync and Smash. It is based on the [Guild Operators guide](https://cardano-community.github.io/guild-operators) using Cabal. Check it out to get more background information or if you want to set up a stake pool, this guide mainly focuses on DB Sync and Smash.

Live preview of the resulting databases here: [https://stakesync.io/db](https://stakesync.io/db) 


## Prerequisites

The guide is written for Debian and Ubuntu servers, although the steps are similar for other operating systems. Minimum hardware requirement is currently about 4GB ram. The initial build and sync process is quite lengthy and resource intensive, decent cpu and ssd hard-disk recommended.

Create a non-root user, e.g:
```
adduser user1
```

Grant it sudo rights.
```
sudo usermod -aG sudo user1
```

Login with the new user and set the timezone.
```
sudo dpkg-reconfigure tzdata
```

Update your system.
```
sudo apt -y update && sudo apt -y upgrade
```

Install necessary tools.
```
sudo apt -y install curl nano htop gpg git gnupg2
```



## Webserver + PHP + Adminer

Some optional basics: PHP Support and a database GUI.


### Install Apache
```
sudo apt install apache2
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo service apache2 restart
```

You should be getting the default Apache website by opening `https://YOURSERVERIP` in a browser.


### Install PHP

#### PHP on Debian
```
sudo apt -y install lsb-release apt-transport-https ca-certificates
sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/php.list
sudo apt -y update && sudo apt -y upgrade
sudo apt -y install php7.4 php7.4-curl php7.4-pgsql php7.4-mysqli
sudo service apache2 restart
```

#### PHP on Ubuntu
```
sudo apt -y install software-properties-common language-pack-en-base
sudo LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php
sudo apt -y update && sudo apt -y upgrade
sudo apt -y install php7.4 php7.4-curl php7.4-pgsql
sudo service apache2 restart
```

### Install Adminer

[Adminer](https://adminer.org) is a simple PHP file to manage databases in a GUI (similar to PHPMyAdmin). Download it directly into the webdir like this
```
sudo curl -L https://www.adminer.org/latest.php -o /var/www/html/adminer.php
```

and manage your databases at `https://YOURSERVERIP/adminer.php`



## Install PostgreSQL

The following 2 commands are only required on Debian 10 (buster)
```
echo "deb http://ftp.us.debian.org/debian/ buster main contrib non-free" | sudo tee /etc/apt/sources.list.d/nonfree.list
sudo apt -y update && sudo apt -y install libpq-dev/buster
```

Proceed setting up PostgreSQL
```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" | sudo tee  /etc/apt/sources.list.d/pgdg.list
sudo apt -y update && sudo apt -y upgrade
sudo apt-get -y install postgresql-13 postgresql-client-13 postgresql-server-dev-13 postgresql-contrib libghc-hdbc-postgresql-dev libpq-dev
sudo systemctl restart postgresql
sudo systemctl enable postgresql
```

Now create two superusers in PostgreSQL. The first one with the same name as your current username ( check with `whoami` ) to allow peer connections from DB-Sync and Smash. The second one (stakesync) is for password-authenticated access, required if you are using Adminer, StakeSync or any other application accessing the databases with password authentication.
```
sudo su postgres
psql
CREATE ROLE user1 SUPERUSER LOGIN;
CREATE ROLE stakesync SUPERUSER LOGIN;
ALTER USER stakesync PASSWORD 'MyPassword123';
\q
exit
```

Open the PostgreSQL configuration file.
```
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

Find "local   all             postgres                                peer" and add this text in the line below:
```
local all stakesync md5
```

Save changes and exit (ctrl+x -> y). Then Restart PostgreSQL.
```
sudo service postgresql restart
```

If you have installed Adminer in the previous step, you can now log in with system "PostgreSQL", username "stakesync" and the password you have just set. After setting up DB Sync, there will be a database called "cexplorer" here where you can watch it sync.



## Install Cardano Node

We keep this guide up to date, so it's recommended to use the same release versions to avoid incompatibilities and deviance. However if the last update of this guide seems outdated, you should replace the version numbers below with more recent release tags.

Documentation: [https://docs.cardano.org/projects/cardano-node/en/latest](https://docs.cardano.org/projects/cardano-node/en/latest)


First run the [Guild Operators](https://cardano-community.github.io/guild-operators/#/basics) cnode helper script, which will take care of any prerequisites and create the cnode folder structure in /opt/cardano/cnode ($CNODE_HOME).
```
mkdir "$HOME/tmp"
cd "$HOME/tmp"
curl -sS -o prereqs.sh https://raw.githubusercontent.com/cardano-community/guild-operators/master/scripts/cnode-helper-scripts/prereqs.sh
chmod 755 prereqs.sh
./prereqs.sh
```

Once completed, run the following commands to compile the node
```
. "${HOME}/.bashrc"
cd ~/git
git clone https://github.com/input-output-hk/cardano-node
cd cardano-node
git fetch --tags --all
git pull
git checkout 1.24.2
nohup $CNODE_HOME/scripts/cabal-build-all.sh -o &
```

Nohup makes the build process run in the background, you can follow the events with `tail nohup.out -f`. It takes about an hour to compile, copying all binaries to `~/.cabal/bin`. Once completed, exit tail with ctrl+c and run `cardano-cli version` and `cardano-node version` to make sure it worked.

Head over to the cnode scripts folder.
```
cd $CNODE_HOME/scripts
```

You can manually start the node with `./cnode.sh`, but it's recommended to install it as a service with `./deploy-as-systemd.sh`. Press `n` when asked to deploy the topology updater as service (only needed for stake pool relay nodes). Finally run `sudo systemctl restart cnode.service` to start the node. You can monitor the status and sync process (3-9 hours) with `./gLiveView.sh`.



## Install DB Sync

After your node is synced up, proceed installing DB Sync.

Documentation: [https://docs.cardano.org/projects/cardano-db-sync/en/latest](https://docs.cardano.org/projects/cardano-db-sync/en/latest)


Back up the binaries that get overwritten with older versions during the build.
```
cp ~/.cabal/bin/cardano-node ~/.cabal/bin/cardano-node-1.24.2
cp ~/.cabal/bin/db-analyser ~/.cabal/bin/db-analyser-1.24.2
```

Clone and build the DB Sync repository.
```
cd ~/git
git clone https://github.com/input-output-hk/cardano-db-sync
cd cardano-db-sync
git fetch --tags --all
git pull
echo -e "package cardano-crypto-praos\n flags: -external-libsodium-vrf" > cabal.project.local
git checkout 7.1.0
nohup $CNODE_HOME/scripts/cabal-build-all.sh &
```

Run `tail nohup.out -f` to follow the process. 

Once completed, restore the overwritten binaries.
```
mv ~/.cabal/bin/cardano-node-1.24.2 ~/.cabal/bin/cardano-node
mv ~/.cabal/bin/db-analyser-1.24.2 ~/.cabal/bin/db-analyser
```

Create an empty folder for the state-dir.
```
mkdir $CNODE_HOME/ledger-state-dbsync
```

Set the pgpassfile and create the database (should return 'All good!').
```
export PGPASSFILE=~/git/cardano-db-sync/config/pgpass-mainnet
chmod 0600 $PGPASSFILE
scripts/postgresql-setup.sh --createdb
```

Create a start script with `nano start.sh` and insert the following command.
```
PGPASSFILE=config/pgpass-mainnet cardano-db-sync-extended --config config/mainnet-config.yaml --socket-path $CNODE_HOME/sockets/node0.socket --state-dir $CNODE_HOME/ledger-state-dbsync/ --schema-dir schema/
```

Save & exit nano and make the file executable.
```
chmod +x start.sh
```

Run DB Sync. (12-24 hours to sync)
```
nohup ./start.sh &
tail nohup.out -f
```



## Install Smash

Proceed installing the Smash Server.

Documentation: [https://docs.cardano.org/projects/smash/en/latest](https://docs.cardano.org/projects/smash/en/latest)


Downgrade GHC to prevent build errors.
```
ghc --version
ghcup install 8.6.5
ghcup set ghc 8.6.5
ghc --version
```

Clone and build the Smash repository.
```
cd ~/git
git clone https://github.com/input-output-hk/smash.git
cd smash
git fetch --tags --all
git pull
git checkout 1.3.0
nohup cabal build smash &
tail nohup.out -f
```

After the build is completed, install Smash.
```
nohup cabal install smash &
tail nohup.out -f
```

Once completed, create the database (should return 'All good!').
```
export SMASHPGPASSFILE=~/git/smash/config/pgpass
chmod 0600 $SMASHPGPASSFILE
SMASHPGPASSFILE=config/pgpass ./scripts/postgresql-setup.sh --createdb
```

Run migrations.
```
SMASHPGPASSFILE=config/pgpass cabal run smash-exe -- run-migrations --config config/mainnet-config.yaml --mdir ./schema
```

Create an empty folder for the state-dir.
```
mkdir $CNODE_HOME/ledger-state-smash
```

Create a start script with `nano start.sh` and insert the following command.
```
SMASHPGPASSFILE=config/pgpass cabal run smash-exe -- run-app-with-db-sync --config config/mainnet-config.yaml --socket-path $CNODE_HOME/sockets/node0.socket --state-dir $CNODE_HOME/ledger-state-smash/ --schema-dir schema/
```

Save & exit nano and make the file executable.
```
chmod +x start.sh
```

Run Smash. (3-6 hours to sync)
```
nohup ./start.sh &
tail nohup.out -f
```

One last sync and you're good to go :)


To install the DB Sync / Smash Add-on [StakeSync](https://stakesync.io), continue with the instructions in its README file.
