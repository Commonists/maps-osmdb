# OSM database replica on Cloud VPS

This repo will contain all necessary tools, scripts, and instructions to maintain the maps project Cloud VPS OSM database replica server.

## Attach and prepare cinder volume

```
sudo mkdir /volume
```

Attach the volume to the VM in Horizon. Take not of the device and `cfdisk /dev/sd?` to partition the entire volume into a single partition.
Format the new partition as `ext4`

```
mkfs.ext4 /dev/sdxy
```

Take note of the UUID of the new partition and add it to `/etc/fstab` mounted at `/volume`

## Install Postgres

```
sudo apt install postgresql postgresql-contrib
```

### Change data directory

Stop postgres, move the data directory onto the cinder

```
sudo systemctl stop postgresql.service
sudo mv /var/lib/postgresql /volume
sudo usermod -d /volume/postgresql postgres
sudo vim /etc/postgresql/13/main/postgresql.conf
```

and change the following lines

```
data_directory = '/var/lib/postgresql/13/main'         # use data in another directory
#wal_level = replica                    # minimal, replica, or logical
#max_replication_slots = 10      # max number of replication slots
```

to

```
data_directory = '/volume/postgresql/13/main'           # use data in another directory
wal_level = logical                     # minimal, replica, or logical
max_replication_slots = 10      # max number of replication slots
```

then restart Postgres. Note that the paths above contain the Postgres version (in this case `13`)!

```
sudo systemctl start postgresql.service
```

### Build empty OSM database

```
sudo su - postgres -c 'createdb gis'
curl https://raw.githubusercontent.com/openstreetmap/openstreetmap-website/master/db/structure.sql | sudo su - postgres - -c 'psql gis'
```

## `osmdbt`

Build and set up the OSM replication tools

### Prerequisites

```
sudo apt install -y libosmium2-dev libprotozero-dev libboost-filesystem-dev libboost-program-options-dev libbz2-dev zlib1g-dev libexpat1-dev cmake libyaml-cpp-dev libpqxx-dev pandoc gettext-base postgresql-common postgresql-server-dev-all devscripts
```

## `osmdbt`

Build the OSM replication tools

```
git submodule update --init
cd osmdbt
```

compile according to the instructions in `osmdbt/README.md`

```
mkdir build
cd build
cmake ..
cmake --build .

```

## Build `osm-locgical` plugin

```
cd osm-logical
make
sudo make install
```

## Setup replicate user

Create user

```
echo "CREATE USER replicate WITH REPLICATION LOGIN;" | sudo su - postgres - -c psql
```

```
echo "GRANT SELECT ON ALL TABLES IN SCHEMA public TO replicate; | sudo su - postgres - -c 'psql gis'
```


