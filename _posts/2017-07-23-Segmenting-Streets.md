---
layout: post
title: Segmenting Open Street Map Streets Using PostGIS.
published: true
project: false
---
This blog post gives an overview on how to segment streets in OSM data.

Overview
---------
A street in OSM data is a linestring. A Linestring represents a sequence of points and line segments connecting them. Segmenting is a process of dividing a segment into multiple segments.

Need for Segmenting
--------------------
Let us suppose there is a big street represented by a linestring. Let us suppose that we interested in a particular part of the street and not the whole street. In such cases we may have to segment the bigger street into small streets.

A figure explaining the above:
<p align="center">
<img src="{{ site.baseurl }}/images/osm_segmentation.png">
</p>

Some PostGIS Functions
-----------------------
Before we start with segmentation, let us first try to understand some basic PostGIS functions which are used in segmentation process.

[ST_Split](https://postgis.net/docs/ST_Split.html) function splits geometries and returns a collection of geometries. The code below illustrates an example

{% highlight psql %}
SELECT ST_AsText(ST_Split(ST_GeomFromText('LINESTRING(0 0, 2 2)'), ST_GeomFromText('POINT(1 1)')));

                          st_astext                          
-------------------------------------------------------------
 GEOMETRYCOLLECTION(LINESTRING(0 0,1 1),LINESTRING(1 1,2 2))

{% endhighlight %} 

