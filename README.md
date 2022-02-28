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
sudo apt install postgresql postgresql-contrib postgis postgresql-13-postgis-3
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
echo 'CREATE EXTENSION postgis;' | sudo su - postgres - -c 'psql gis'
echo 'CREATE EXTENSION hstore;' | sudo su - postgres - -c 'psql gis'
```

### Build `osm2pgsql`

Install prerequisites

```
sudo apt-get install make cmake g++ libboost-dev libboost-system-dev \
  libboost-filesystem-dev libexpat1-dev zlib1g-dev \
  libbz2-dev libpq-dev libproj-dev lua5.3 liblua5.3-dev pandoc python3-pyosmium python3-psycopg2
```

configure and build

```
cd osm2pgsql
mkdir build && cd build
cmake ..
make
sudo make install
```

### Build `osmupdate`

```
wget -O - http://m.m.i24.cc/osmupdate.c | cc -x c - -o osmupdate
```

## Initial import

### Download weekly Planet .osm.pbf file

The following command will download about 65GB of data (as of early 2022)

```
wget https://ftp.osuosl.org/pub/openstreetmap/pbf/planet-latest.osm.pbf
```

an old planet pbf can be made current with `osmupdate` (this step can take a few hours)

```
./osmupdate --day --hour planet-latest.osm.pbf planet-latest2.osm.pbf
mv planet-latest2.osm.pbf planet-latest.osm.pbf
```

### Import pbf into postgres

```
osm2pgsql -c -s -k planet-latest.osm.pbf -d gis
```

### Initialize replication

```
osm2pgsql-replication init -d gis
```

### Replication

```
osm2pgsql-replication update -d gis
```

## References

* https://ircama.github.io/osm-carto-tutorials/updating-data/
