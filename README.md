# osm-server

This provides an open street map tile server inside Docker containers.

## Setup
You'll need [Docker](https://www.docker.com/) 17.12.0 or higher, and [Docker compose](https://docs.docker.com/compose/) installed.

### Initialise the tile server
You can initialise the tile server by running the script

```
./initialise_tile_server    # Linux
initialise_tile_server.bat  # Linux type shell on Windows
```

This will start two containers - a Postgres database and a renderer/web server. It takes about 30s to run because it waits for the containers to be up and then restarts them. A network and a volume for the database will also be created.

### Get some map data
Download a PBF file of OSM data from somewhere like [Geofabrik](http://download.geofabrik.de/).
If you'd like a set of these merged, you can do this using the [OSM PBF merger](https://github.com/bwindsor/osm-pbf-merger) container.

### Populate the database
#### 1. Set Environment variables
These two variables need setting
```
export IMPORT_FROM=/c/Users/myusername/osm-server
export IMPORT_FILE=rhode-island-latest.osm.pbf
```
* `IMPORT_FROM` is the directory name on your computer. If you're using `docker-machine` on Windows, this is the directory location mounted inside the machine.
* `IMPORT_FILE` is just the name of the `osm.pbf` file which you want to import.

* On Windows, replace the word `export` with `SET`.

#### 2. Import data
Once those environment variables are set, this will import your data:

`docker-compose -f docker-compose-importer.yml run --rm data-importer`

The import may also fail if there is insufficient RAM for the import. See the [Customising Data Import](#customising-data-import) and [Hardware Requirements](#hardware-requirements) sections.

**This step can take a very long time! Typically 8 hours for the UK, and few days for the USA.**

### Restart the tile server
Once data import is complete, you should restart the tile server with

```
docker-compose restart
```
You'll then be able to find a map page at `http://localhost`. If you've got your own web client, you'll find the tiles at `http://localhost/osm-tiles/{z}/{x}/{y}.png` where `z` is the zoom coordinate and `x` and `y` are the tile coordinates for that zoom level. For a basic test visit `http://localhost/osm-tiles/0/0/0.png` and see if you can see the world. Replace `localhost` with the IP of your docker machine (usually `192.168.99.100`) if you're on Windows.

### Updating the tile server
1. Make sure the server is running.
2. Set environment variables and run the import command as in the [Populate the database](#populate-the-database) section. Note this will clear the database before repopulating it. If you want to keep the old database then you'll need to backup the `osmserver_osm_postgres_database` volume. [Here's an example of how to do that](https://loomchild.net/2017/03/26/backup-restore-docker-named-volumes/).
3. Clear pre-rendered tiles as in the [clear pre-rendered or cached tiles](#clear-pre-rendered-or-cached-tiles) section.
4. Restart the tile server as in the [Restart the tile server](#restart-the-tile-server) section.

### Destroy the tile server
To clean everything up, remove the containers and the volume containing the postgres database with
```
docker-compose down  # Removes the containers and network
docker volume rm osmserver_osm_postgres_database  # Removes the data
```
The volume name to remove may be different - you can find the volume name with `docker volume ls -q | grep osm_postgres_database | head -1`.

### Customising data import
You may wish to customise the parameters which are passed to `osm2pgsql` which is the program which imports the data into the database. Changing these can vary the speed of the import quite significantly. In particular, giving it more RAM helps. You can edit the `docker-compose-importer.yml` file to change the `osm2pgsql` command. The `--cache` argument is how many MB of RAM can be used for the import. The `--cache-strategy` is about how memory is allocated (in one block or sparsely). More information on the import can be found [here](https://wiki.openstreetmap.org/wiki/Osm2pgsql#Optimization) and [here](http://www.volkerschatz.com/net/osm/osm2pgsql-usage.html).

### Tile pre-rendering
You can force the rendering of all or a subset of map tiles. This means that when they are requested it will be faster, because the png will only need to be fetched from disk rather than having to be generated from the database as well. The `render_list_geo` perl script does this. See the README for [this repository](https://github.com/alx77/render_list_geo.pl) for more information on how to use the command..

In the following command, you will need to replace `osmserver_tile-server_1` with the name of the running tile server docker container. You should be able to get the name of it with `docker ps | grep tile-server`.

This example renders zoom levels 0 to 15 for the UK. (z is zoom, x is longitude, y is latitude). See also [the original readme](https://github.com/alx77/render_list_geo.pl).

`docker exec -it osmserver_tile-server_1 render_list_geo -z 0 -Z 15 -x -8 -X 1.8 -y 49.8 -Y 60.9`

### Clear pre-rendered or cached tiles
In the following command, you will need to replace `osmserver_tile-server_1` with the name of the running tile server docker container. You should be able to get the name of it with `docker ps | grep tile-server`. This will clear all cached tiles:

`docker exec -it osmserver_tile-server_1 rm -r /var/lib/mod_tile/default`

### Hardware requirements
**Memory:** The more memory you can give the process, the better. Typically `--cache 4000` is enough for it not to be too slow, but `--cache 64000` on a beefy computer would be faster. The `--cache-strategy` flag can also make a difference.

**Disk:** To import the USA you'll need about 250GB of disk space.

**CPU:** For rendering tiles quickly, more cores and faster speeds will lead to tiles loading faster for the user.

### User Interface
This system provides a tile service - to view a map in a browser you'll need a front end web page which requests the tiles and displays them correctly. You can use [Leaflet](http://leafletjs.com/) to do this. Inside the folder `example-front-end` you'll see an example of such a page. You'll need to edit `index.js` to make `TILE_SERVER_URL` point to the tile server which you've just set up.

## How it works
There are three components to this system
### 1. Postgres database
This stores the OSM map data in the form of vertices, lines, etc. with associated metadata. The docker image used for this is a basic Postgres database with some extensions specific to OSM added.
### 2. Web tile server/renderer
When a tile is requested at a certain zoom level and location, this container queries the database for the relevant data and then renders it into a png image. This image is then cached inside the container, so expect this container to grow in size over time.
### 3. Data importer
This isn't part of the running system, but this container contains `osm2pgsql`, a program to connect to the postgres database and populate it with data from a `.pbf` file.

### Storage
* The database is stored in a docker volume which is mounted into the postgres container. This volume is created during the `docker-compose up` command. When `docker-compose down` destroys the containers, this volume is not destroyed but should not be reused - you can manually destroy it.
* The rendered png tiles are cached inside the web server container. As such the tile server container will grow in size over time.

### Networking
* A network is connected to by all three containers. This allows them to communicate with each other.
