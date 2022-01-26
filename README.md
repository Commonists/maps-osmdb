# OSM database replica on Cloud VPS

This repo will contain all necessary tools, scripts, and instructions to maintain teh maps project Cloud VPS OSM database replica server.

## Prerequisites

```
sudo apt install -y libosmium2-dev libprotozero-dev libboost-filesystem-dev libboost-program-options-dev libbz2-dev zlib1g-dev libexpat1-dev cmake libyaml-cpp-dev libpqxx-dev pandoc gettext-base postgresql-common postgresql-server-dev-all
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
