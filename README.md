Geo Spatial Demo
====================

NOTE - this example requires DSE > 4.5

## Scenario

This is short demo which shows how to find all the uk postcode within a certain area. This area can be a postcode
or a longitude/latitude combination.

## Schema Setup
Note : This will drop the keyspace "datastax_postcode_demo" and create a new one. All existing data will be lost. 

To specify contact points use the contactPoints command line parameter e.g. '-DcontactPoints=192.168.25.100,192.168.25.101'
The contact points can take mulitple points in the IP,IP,IP (no spaces).

To create the a single node cluster with replication factor of 1 for standard localhost setup, run the following

    mvn clean compile exec:java -Dexec.mainClass="com.datastax.demo.SchemaSetup"

Now we can load the postcode data from the postcodes_new.csv file in the home directory.

You can use the cqlsh copy to do this. 

Open a cql shell and run  

	copy datastax_postcode_demo.postcodes (post_code, lon_lat) from '{demo_dir}/postcodes_new.csv';

When the data is loaded, the Solr code needs to be updated with config and schema which will allow us to search.

Then go to the 'src/main/resources/solr'

Use the 'Commands.txt' file which shows the commands run to add the config and schema to create the Solr core. The commands assume you are running this on your localhost, if you are not, you will need to change the host name. 

Once these commands have been run, we can start to query the Solr Admin. 

First we will using the 'q' field to query for post_code = SW209AQ 

![Image of Postcode lookup]	
(https://raw.githubusercontent.com/PatrickCallaghan/images/master/PostCodeLookup.png)

To do this in CQL run the following
 
	select * from postcodes where solr_query = '{"q": "post_code:SW209AQ"}';

Next we do a spatial query to filter out anything that is outside of 1km from lon/lat co-ordinates that we found from looking up SW209AQ. Now we use check the spatial checkbox and fill in the details - also we use the 'fq' field to declare we are geo filtering. The 'q' field is now set back to '*:*'

![Image of Postcode lookup within Distance]	
(https://raw.githubusercontent.com/PatrickCallaghan/images/master/PostCodeDist1.png)

To do this in CQL run the following

	select * from postcodes where solr_query = '{"q": "*:*", "fq": "{!geofilt sfield=lon_lat pt=51.404970234124800,-.206445841245690 d=1}"}';
	
Next we want to add the distance from the query point to the results and also sort them by the nearest postcodes. For this we add the 'geodist() asc' to the sort field and in the field 'fl' we will return 'post_code, lon_lat and geodist()'

![Image of Postcode lookup within Distance sorted]	
(https://raw.githubusercontent.com/PatrickCallaghan/images/master/PostCodeDist1Sorted.png)


Finally we are going to change the search area to be a bounded box area which has a radius of 1km. This is sometimes better for maps that are relatively square. For this we change the filtering query from '{!geofilt}' to '{!bbox}'

![Image of Postcode lookup for a bounded box filter]	
(https://raw.githubusercontent.com/PatrickCallaghan/images/master/PostCodeBBox.png)


To remove the tables and the schema, run the following.

    mvn clean compile exec:java -Dexec.mainClass="com.datastax.demo.SchemaTeardown"
    
To use with Spark you can use the following

Using Cassandra Table

	sc.cassandraTable("datastax_postcode_demo", "postcodes").select("post_code").where("solr_query='{\"q\": \"*:*\", \"fq\": \"{!geofilt sfield=lon_lat pt=51.404970234124800,-.206445841245690 d=1}\"}'").collect.foreach(println)
	
	sc.cassandraTable("datastax_postcode_demo", "postcodes").select("post_code").where("solr_query='{\"q\": \"*:*\", \"fq\": \"lon_lat:[51.2,-.2064458 TO 51.3,-.2015418]\"}'").collect.foreach(println)

Filtering with radius and box bounds

	import com.datastax.spark.connector.cql.CassandraConnector
	import scala.collection.JavaConversions._

	CassandraConnector(sc.getConf).withSessionDo { session => session.execute("select * from datastax_postcode_demo.postcodes where solr_query='{\"q\": \"*:*\", \"fq\": \"{!geofilt sfield=lon_lat pt=51.404970234124800,-.206445841245690 d=1}\"}'")
	}.all.foreach(println)

	val rdd = CassandraConnector(sc.getConf).withSessionDo { session =>
	  session.execute("select post_code, lon_lat from datastax_postcode_demo.postcodes where solr_query='{\"q\": \"*:*\", \"fq\": \"{!bbox sfield=lon_lat pt=51.404970234124800,-.206445841245690 d=1}\"}'")
	}.all.foreach(println)

Spark SQL

	import org.apache.spark.sql.cassandra.CassandraSQLContext
	
	val rdd = csc.sql("select post_code, lon_lat from datastax_postcode_demo.postcodes where solr_query='{\"q\": \"*:*\", \"fq\": \"{!geofilt sfield=lon_lat pt=51.404970234124800,-.206445841245690 d=1}\"}'")
	rdd.collect.foreach(println)
	
	val rdd = csc.sql("select post_code, lon_lat from datastax_postcode_demo.postcodes where solr_query='{\"q\": \"*:*\", \"fq\": \"lon_lat:[51.2,-.2064458 TO 51.3,-.2015418]\"}'")
	rdd.collect.foreach(println)    
    
    

##Front End Setup
Once the setup of the backend has been completed, you can use the front end as well.

The front-end application is composed of a d3.js powered map of the UK (and Ireland). It leverages the solr HTTP API from front end ajax (this is not recommended in prod environments or IRL due to the security risks associated with bypassing cors and exposing the Solr API, at the very least there should be a solr security proxy in place).

Install node.js (needs to be 0.10.x or greater):

    apt-add-repository ppa:chris-lea/node.js
    curl -sL https://deb.nodesource.com/setup | sudo bash -
    sudo apt-get install -y nodejs

    node -v

    sudo apt-get install npm
    sudo npm install -g cors-anywhere

Dependencies:

    sudo npm install -g cors-anywhere
    sudo npm install cors-anywhere

Kick off the proxy:

    screen node server.js 

Then Navigate to GeoMap and kick off the web server that will host your web application (depends on python 2.x):

    python -m SimpleHTTPServer
    
You can now test the app on port 8000

For more on Spatial Search in Solr please visit https://wiki.apache.org/solr/SpatialSearch
    
    
