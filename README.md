# Using common GIS tools

## Visualizing and operating on geospatial data

I've worked for a large ridesharing company for three years and toward the beginning of my job in late 2014 was responsible for 
much of the public-facing data visualization[1], and many of favorites were static maps. 
I’ve found these to be an incredibly useful way of drawing insight from data. Here's a tutorial on how to create simple works like those.

## Why use PostGIS for GIS

The reasons to use PostGIS are numerous:

**Efficiency by avoiding cross-joins.** Postgres/PostGIS can index data for hyper-efficient spatial joins. For example,
if I have a bunch of polygons for ZIP code boundaries in New York and California, and many points in both California and New York, 
my query engine should be intelligent enough to determine that it shouldn’t check if a point is contained in an 
obviously distant shapes–don’t check if points in NY belong in CA. 

**Scale**. You can very easily get about 120 million points indexed on a standard MacBook Pro running PostGIS--
I’m sure you can comfortably get many more on a 2xlarge EC2. Running these analyses locally
means that you don’t have to ask a DBA for permission to make table changes.

**Postgres ecosystem**. Postgres is a quite good analytics database–date-time conversions, string aggregation, UDAFs, UDFs.

**Living documentation**.  Since the GIS community has hardly migrated away from PostGIS, you have the benefit of 10+ years of StackOverflow Q+A’s.

## Why use QGIS

QGIS is my vote for the best free geospatial data visualizer and editor. Candidly, I wish I didn’t have to use QGIS. 
It’s quite buggy--long-time colleagues know I describe it as a piece of malware that happens to double as mapping software.
However, it’s still the best free GUI-based tool for the job.

## Setup

1) Install Postgres. I'd recommend either [Docker PostGIS](https://hub.docker.com/r/mdillon/postgis/) (if you're familiar with Docker)
or the [Postgres.app](https://postgresapp.com) installation if you're not.
2) Connect to the prompt of your Postgres installation and type `CREATE EXTENSION postgis;` If it's already installed, you'll get
the message `ERROR:  extension "postgis" already exists`. If instead you see the phrase `CREATE EXTENSION`,
you'll have installed the PostGIS extension for Postgres, which will let us manipulate geometry data in SQL.
3) Install [QGIS](https://www.qgis.org/en/site/forusers/alldownloads.html), which will help visualize our work. (I recommend the KyngChaos installation if you're using OS X.) It's not uncommon in my experience for QGIS to be buggy, and the installation process is occasionally finicky.
4) Optional, but highly recommended–[Install a basemap plugin](https://gis.stackexchange.com/questions/20191/adding-basemaps-from-google-or-bing-in-qgis). This will put a map behind the geometry objects that we’ll visualize on a map.
5) You may also need to install [shp2pgsql](https://gis.stackexchange.com/questions/148524/i-have-not-found-shp2pgsql-in-postgis-installation) for converting shapefiles to Postgres data.

## Data for these examples

1) From the [Uber TLC FOIL data set](https://github.com/fivethirtyeight/uber-tlc-foil-response), download the July 2014 data by right-clicking [here](https://raw.githubusercontent.com/fivethirtyeight/uber-tlc-foil-response/master/uber-trip-data/uber-raw-data-jul14.csv) and selecting *Save as...*.
2) Download the New York state urban area neighborhood boundaries from Zillow.com by clicking [here](https://www.zillowstatic.com/static/shp/ZillowNeighborhoods-NY.zip). Unpack the zip file and double-click on the .shp, and open it up in QGIS. We should be able to navigate to the burroughs of New York City.

![Viewing NYC .shp](https://raw.githubusercontent.com/ajduberstein/gis_tutorial/master/1_using_qgis.gif)

## Visualizing point-in-polygon relationships

### Importing polygons into Postgres

Here's a directory with both files:
```
duberstein@MacBook-Pro:~/Desktop/geo|
⇒  ls
ZillowNeighborhoods-NY  uber-raw-data-jul14.csv
```

I can copy the neighborhood boundaries into Postgres using the `shp2pgsql` utility:

```
duberstein@MacBook-Pro:~/Desktop/geo|
⇒  shp2pgsql -s 4326 ZillowNeighborhoods-NY/ZillowNeighborhoods-NY.shp ny_nbhds postgres > ny.sql
```

This produces a SQL file with all the shape data--run `more ny.sql` to see the contents. You can execute that SQL file from `psql` with the `-f` flag.

```
duberstein@MacBook-Pro:~/Desktop/geo|
⇒  psql -f ny.sql
```

You've now created a table called `ny_nbhds`. Check it out using `psql`:

```
duberstein@MacBook-Pro:~/Desktop/geo|
⇒  psql
psql (9.5.7)
Type "help" for help.

duberstein=# SELECT COUNT(*) FROM ny_nbhds;
 count
-------
   266
(1 row)
```

The table itself contains shapes and their metadata:

```
duberstein=# \x auto
Expanded display is used automatically.
duberstein=# SELECT * FROM ny_nbhds LIMIT 1;
-[ RECORD 1 ]-
gid      | 1
state    | NY
county   | Monroe
city     | Rochester
name     | Ellwanger-Barry
regionid | 343894
geom     | 0106000020E6100...
```

The shape is listed in a format called [well-known binary (WKB)](http://edndoc.esri.com/arcsde/9.1/general_topics/wkb_representation.htm), which we'll see a few more times. It's a common representation for geometry data.

We can also see what values are indexed on the table:

```
duberstein=# \d ny_nbhds
                                     Table "public.ny_nbhds"
  Column  |            Type             |                       Modifiers
----------+-----------------------------+--------------------------------------------------------
 gid      | integer                     | not null default nextval('ny_nbhds_gid_seq'::regclass)
 state    | character varying(2)        |
 county   | character varying(43)       |
 city     | character varying(64)       |
 name     | character varying(64)       |
 regionid | numeric                     |
 geom     | geometry(MultiPolygon,4326) |
Indexes:
    "ny_nbhds_pkey" PRIMARY KEY, btree (gid)
```

Since we plan to join on geometry data, let's add an index to the geometry like so:

```
duberstein=# CREATE INDEX ON ny_nbhds USING GIST(geom);
CREATE INDEX
```

We can use `\d` again in the `psql` prompt to verify that we have an additional index:

```
duberstein=# \d ny_nbhds
                                     Table "public.ny_nbhds"
  Column  |            Type             |                       Modifiers
----------+-----------------------------+--------------------------------------------------------
...
Indexes:
    "ny_nbhds_pkey" PRIMARY KEY, btree (gid)
    "ny_nbhds_geom_idx" gist (geom)
```

##

## Visualizing points-near-POI relationships

I don’t believe there’s a real name for this pattern of work, but it’s definitely frequent request. For example, say we want 
to study the complementarity between a for-hire vehicles and public transit.
To do that, we could find all the drop-offs that were within 10 meters of a public transit stop.

Placeholder text

## What to do when you don’t have PostGIS

*Relying on geohashes.* Geohashes are often a great way of solving the problem and a decent hack for creating indexes. 
Although it’s not as efficient as a [GiST index](https://en.wikipedia.org/wiki/GiST),
you can use them in MapReduce-based frameworks to significantly speed up
spatial joins.

[1] For example, [here](https://www.americaninno.com/boston/uber-boston-uberx-more-responsive-than-boston-taxis/),
[here](https://pbs.twimg.com/media/BxXPUwbIQAEp0TY.jpg), and [here](https://uberblogapi.10upcdn.com/wp-content/uploads/sites/78/2014/11/Las_Vegas_Eyeballs.jpg).
