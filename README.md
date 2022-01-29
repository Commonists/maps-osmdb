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

### Install osmosis

```
sudo apt install -y osmosis
```

### Build `osmupdate`

```
wget -O - http://m.m.i24.cc/osmupdate.c | cc -x c - -o osmupdate
```

## Initial import

### Download weekly Planet .osm.pbf file

```
wget https://ftp.osuosl.org/pub/openstreetmap/pbf/planet-latest.osm.pbf
```

```
osmosis --read-pbf planet-latest.osm.pbf --write-apidb host="localhost" database="gis" user="$psql_username" password="$psql_password" validateSchemaVersion="no"
```
