Map-Reduce Ingest of GDELT
=================

This tutorial shows how to:

1. Ingest big [GDELT](http://www.gdeltproject.org) data files into a
   GeoMesa Accumulo table via a Hadoop Map/Reduce job.
2. Leverage GeoServer to query and visualize the data.

Prerequisites
-------------

> :warning: You will need access to a Hadoop 2.2.2 or better installation as well as an Accumulo 1.5-1.6 database.

You will also need:

-  access to an Accumulo user that has both create-table and write
   permissions,
-  an instance of GeoServer 2.8.1 with the GeoMesa plugin installed,
-  Java JDK 7,
-  [Apache Maven](http://maven.apache.org) 3.2.2, and
-  a [git](http://git-scm.com) client.

Obtaining GDELT data
--------------------

The [GDELT Event database](http://www.gdeltproject.org) provides a
comprehensive time- and location-indexed archive of events reported in
broadcast, print, and web news media worldwide from 1979 to today. You
can download raw GDELT data files at
http://data.gdeltproject.org/events/index.html. For example, the
following Bash commands use ``wget`` to download all event files for
2013 and 2014 (5 GB compressed, 30 GB uncompressed) and ``md5sum`` to
verify each by their MD5 hash:

```bash
$ mkdir gdelt && cd gdelt
$ wget http://data.gdeltproject.org/events/md5sums
$ for file in `cat md5sums | cut -d' ' -f3 | grep '^201[34]'` ; do wget http://data.gdeltproject.org/events/$file ; done
$ md5sum -c md5sums 2>&1 | grep '^201[34]'
```

Note: the whole GDELT data set is very large, over 100 GB compressed and
over 1 TB uncompressed. We download all data for 2013-2014 as we look at
events in the Ukraine in this time period in the **ANALYZE** section
below. However, you may load a smaller dataset for demonstration
purposes; data for a typical month is about ~2 GB uncompressed, but you
will see fewer data in the analysis below. In any case, omit the
aggregated, reduced GDELT data in
``GDELT.MASTERREDUCEDV2.1979-2013.zip``, which has fewer columns than
the fully-described event data in the other files, and is not supported
by the ingest code described below.

In the directory with the ``*.zip`` data files, use the following
command to unpack and aggregate them into a file called ``gdelt.tsv`` on
the Hadoop file system:

```bash
$ (ls -1 *.zip | xargs -n 1 zcat) | hadoop fs -put - /gdelt/uncompressed/gdelt.tsv
```

Note: the individual files, when uncompressed, have ``*.csv``
extensions, but the data contained within them are tab-separated values,
hence the ``*.tsv`` extension. See the [GDELT raw data file
documentation](http://www.gdeltproject.org/data.html#rawdatafiles)
for more information on the format of these files.

Building the tutorial code
~~~~~~~~~~~~~~~~~~~~~~~~~~

Clone the geomesa project and build it, if you haven't already:

```bash
$ git clone https://github.com/locationtech/geomesa.git
$ cd geomesa
$ mvn clean install
```

This is needed to install the GeoMesa JAR files in your local Maven
repository. For more information see the [GeoMesa Accumulo Quick
Start](../geomesa-quickstart-accumulo) tutorial.

Clone the geomesa-tutorials project and build it:

```bash
$ git clone https://github.com/geomesa/geomesa-tutorials.git
```

The example code is in ``geomesa-examples-gdelt``:

```bash
$ cd geomesa-tutorials/geomesa-examples-gdelt
$ mvn clean install
```

After building, the built JAR file bundled with all dependencies will be
in the ``target`` subdirectory.

Running the ingest
~~~~~~~~~~~~~~~~~~

Use ``hadoop jar`` to launch the Map/Reduce ingest job:

```bash
$ hadoop jar ./target/geomesa-examples-$VERSION.jar \
   com.example.geomesa.gdelt.GDELTIngest           \
   -instanceId <accumulo-instance-id>              \
   -zookeepers <zookeeper-hosts-string>            \
   -user <username> -password <password>           \
   -auths <comma-separated-authorization-string>   \
   -tableName gdelt -featureName event             \
   -ingestFile hdfs:///gdelt/uncompressed/gdelt.tsv
```

Note that authorizations are optional. Unless you know that your table
already exists with explicit authorizations, or that it will be created
with default authorizations, you probably want to omit this parameter.

DataStore Initialization
------------------------

[GeoTools](http://www.geotools.org) uses a ``SimpleFeatureType`` to
represent the schema for individual ``SimpleFeatures`` created from
the GDELT data. We can easily create a schema for the GDELT
``SimpleFeatureType`` using the [GeoTools DataUtilities
class](http://docs.geotools.org/latest/userguide/library/main/feature.html).
The schema string is a comma separated list of attribute descriptors of
the form "<name>:<class>", e.g. "Year:Integer". Some attributes may have
a third term with an appended "hint", e.g. "geom:Point:srid=4236", and
the default geometry attribute is often prepended with an asterisk. For
example, a complete schema string for a ``SimpleFeatureType`` describing
a city with a latitude/longitude point, a name, and a population might
be "\*geom:Point:srid=4326,cityname:String,population:Integer".

This is how ``GDELTIngest.java`` creates the ``SimpleFeatureType`` for
the GDELT event data:

```java
static List<String> attributes = Lists.newArrayList(
    "GLOBALEVENTID:Integer",
    "SQLDATE:Date",
    "MonthYear:Integer",
    "Year:Integer",
    "FractionDate:Float",
    //...
    "*geom:Point:srid=4326"
);
String spec = Joiner.on(",").join(attributes);
SimpleFeatureType featureType = DataUtilities.createType(name, spec);
```

After we create the GDELT feature type, we have to tell GeoMesa which
field to use for the date index. We specify this field using the
``SimpleFeatureType``'s user data.

```java
//This tells GeoMesa to use this Attribute as the Start Time index
featureType.getUserData().put(Constants.SF_PROPERTY_START_TIME, "SQLDATE");
```

Finally, we create the new feature type in GeoMesa as follows.

```java
ds.createSchema(featureType);
```

Mapper
------

In the ``setup`` method of the Mapper class, we grab the connection
params from the ``JobContext`` and get a handle on a ``FeatureWriter``.

```java
DataStore ds = DataStoreFinder.getDataStore(connectionParams);
featureType = ds.getSchema(featureName);
featureBuilder = new SimpleFeatureBuilder(featureType);
featureWriter = ds.getFeatureWriter(featureName, Transaction.AUTO_COMMIT);
```

The input to the map method is a single line of the GDELT TSV file. We
split the line on tabs and extract the attributes of the data. We parse
the latitude and longitude field to set the default geometry of our
``SimpleFeature``.

GeoTools provides common conversions for most data types and some date
formats. However, any attribute strings that will not convert
automatically into the specified class need to be explicitly set on the
``SimpleFeature``. See "SQLDATE" below.

```java 
featureBuilder.reset();
featureBuilder.addAll(attributes);

Double lat = Double.parseDouble(attributes[LATITUDE_COL_IDX]);
Double lon = Double.parseDouble(attributes[LONGITUDE_COL_IDX]);
Geometry geom = geometryFactory.createPoint(new Coordinate(lon, lat));
SimpleFeature simpleFeature = featureBuilder.buildFeature(attributes[ID_COL_IDX]);
simpleFeature.setAttribute("SQLDATE", formatter.parse(attributes[DATE_COL_IDX]));
simpleFeature.setDefaultGeometry(geom);

try { SimpleFeature next = featureWriter.next();
    next.setAttributes(simpleFeature.getAttributes());
    ((FeatureIdImpl)next.getIdentifier()).setID(simpleFeature.getID());
    featureWriter.write();
}
```

Analyze
-------

### GeoServer Setup

First, make sure that GeoServer is installed and configured to use GeoMesa as described in the :doc:`../user/installation_and_configuration` section of the GeoMesa User Manual.

### Register the GeoMesa DataStore with GeoServer

Click "Stores" and "Add new Store". If you do not see the Accumulo
Feature Data Store listed under Vector Data Sources, ensure the plugin
is in the right directory and restart GeoServer.

![Registering new Data Store](../assets/geomesa-examples-gdelt/Accumulo_Feature_Data_Store.png)

Register the newly created Accumulo table using the same parameters
specified in the command line above. (If you use a workspace:layer name
other than "geomesa:gdelt", you will need to change the WMS requests
that follow.)

![Registering new Accumulo Feature Data Store](../assets/geomesa-examples-gdelt/Geoserver_Accumulo_Store_Registration.png)

### Publish layer

After registering the DataStore, click to publish the layer. You will be
taken to the Edit Layer screen. In the Data pane, enter values for the
bounding boxes. For the whole world, use [-180,-90,180,90]. In the
Dimensions pane, enable Time and select the SQLDATE field. You will also
need to specify a presentation for time - use List as a default.

![Enable Time for the Layer](../assets/geomesa-examples-gdelt/Edit_Layer_Enable_Time.png)

### Query

Let's look at events in the Ukraine from 2013 until April 30, 2014. The
red squares are the default styling that GeoServer applies to point
data.

```bash
http://localhost:8080/geoserver/wms?service=WMS&version=1.1.0&request=GetMap&layers=geomesa:gdelt&styles=&bbox=31.6,44,37.4,47.75&width=1200&height=600&srs=EPSG:4326&format=application/openlayers&TIME=2013-01-01T00:00:00.000Z/2014-04-30T23:00:00.000Z
```

![Showing all GDELT events from Jan 1, 2013 to April 30, 2014](../assets/geomesa-examples-gdelt/Ukraine_Unfiltered.png)

The above map is using the [Stamen Toner](http://maps.stamen.com/toner) layer as a base layer. For more
information about adding multiple layers into one group see the
[GeoServer documentation](http://docs.geoserver.org/stable/en/user/webadmin/data/layergroups.html).

### Filter

Let's narrow our results. GDELT labels events with [CAMEO (Conflict and 
Mediation Event Observations)](http://www.gdeltproject.org/data.html#documentation)
codes. The CAMEO code for events of type 'THREATEN' starts with '13'. We
can filter down to these events using the drop down in GeoServer's
OpenLayers preview.

![Open GeoServer Toggle Options Toolbar](../assets/geomesa-examples-gdelt/Geoserver_Toggle_Options_Toolbar.png)

![Enter CQL Filter into Toolbar](../assets/geomesa-examples-gdelt/Geoserver_Layer_Preview_Drop_Down.png)

Let's use a custom icon to display THREATEN events, by adding an [SLD
style](http://docs.geoserver.org/latest/en/user/styling/index.html)
to the layer. Add the SLD file [threat.sld](../assets/geomesa-examples-gdelt/2014-04-17-geomesa-gdelt-analysis/threat.sld)
to GeoServer (See the GeoServer
documentation for [more information about adding SLD
files](http://docs.geoserver.org/latest/en/user/styling/sld-working.html).
For the ExternalGraphic in the SLD to work, move the image file to the
specified location in your GeoServer installation.

```bash
http://localhost:8080/geoserver/wms?service=WMS&version=1.1.0&request=GetMap&layers=geomesa:gdelt&CQL_FILTER=EventRootCode=13&styles=threat&bbox=31.6,44,37.4,47.75&width=1200&height=600&srs=EPSG:4326&format=application/openlayers&TIME=2013-01-01T00:00:00.000Z/2014-04-30T23:00:00.000Z
```

![Showing GDELT events with CAMEO root code THREATEN from Jan 1, 2013 to April 30, 2014]
(../assets/geomesa-examples-gdelt/Ukraine_Event_RootCode_Threaten.png)

### Heatmaps

Use a heatmap to more clearly visualize multiple events in the same
location or high volume of data in general. Add the SLD file [heatmap.sld](../assets/geomesa-examples-gdelt/heatmap.sld)
to GeoServer.

In the request below, the heatmap is before the points layer so that the
points will be overlayed and not hidden. Notice the
"&env=radiusPixels:30" in the URL; this is SLD variable substitution,
and will replace the default value assigned in the SLD.

```bash
http://localhost:8080/geoserver/wms?service=WMS&version=1.1.0&request=GetMap&layers=geomesa:gdelt,geomesa:gdelt&CQL_FILTER=include;EventRootCode=13&styles=heatmap,threat&bbox=31.6,44,37.4,47.75&width=1200&height=600&srs=EPSG:4326&format=application/openlayers&TIME=2013-01-01T00:00:00.000Z/2014-04-30T23:00:00.000Z&env=radiusPixels:30
```

![Showing heatmap with event overlay of GDELT events with CAMEO root code THREATEN from Jan 1, 2013 to April 30, 2014]
(../assets/geomesa-examples-gdelt/Heatmap_Ukraine_EventRootCode_Threaten.png)