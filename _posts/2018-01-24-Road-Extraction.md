---
layout: post
title: Prediction of most congested roads in Melbourne Road Network.
published: true
project: false
---
This blog post gives an overview on how accurately we can predict the congested roads of a road network using graph theory without any prior information like traffic.

Overview
---------
Roads play a vital role in transportation. Road transport can be defined as the transport of passengers or goods along the roads. Road transport is a popular and most preferred mode of transport. The increased usage of the road network lead to congestion/traffic which is a growing problem in many countries. Traffic has many negative effects like delay, fuel wastage, accidents etc. There is a need to monitor the traffic by the road
network authorities to manage the traffic and further planning. Usually the traffic is monitored by deploying CCTV, using probe vehicles etc. However we would like to evaluate a graph theory based solution to predict the most congested roads.

Datasets
--------
The current method is tested on the road network of Melbourne, Australia. The road network data used in this study is extracted from Open Street Map Data using osm2pgrouting tool. The analysis was carried out using pgrouting tool. The results were tested and verified against the Average Annual Daily Traffic(AADT) data of Melbourne.

Preprocessing
--------------

{% highlight psql %}
SELECT ST_AsText(ST_Split(ST_GeomFromText('LINESTRING(0 0, 2 2)'), ST_GeomFromText('POINT(1 1)')));

                          st_astext                          
-------------------------------------------------------------
 GEOMETRYCOLLECTION(LINESTRING(0 0,1 1),LINESTRING(1 1,2 2))

{% endhighlight %} 

