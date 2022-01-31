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
sudo apt install postgresql postgresql-contrib postgis
```

### Change data directory

Stop postgres, move the data directory onto the cinder

```
sudo systemctl stop postgresql.service
sudo mv /var/lib/postgresql /volume
sudo usermod -d /volume/postgresql postgres
sudo vim /etc/postgresql/13/main/postgresql.conf
```

and set the following options

```
data_directory = '/volume/postgresql/13/main'
shared_buffers = 1GB
work_mem = 50MB
maintenance_work_mem = 10GB
autovacuum_work_mem = 2GB
wal_level = minimal
checkpoint_timeout = 60min
max_wal_size = 10GB
checkpoint_completuion_target = 0.9
max_wal_senders = 0
random_page_cost = 1.0
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

### Install `osmosis` and `osm2pgsql`

```
sudo apt install -y osmosis osm2pgsql osmctools
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


## References

* https://ircama.github.io/osm-carto-tutorials/updating-data/
