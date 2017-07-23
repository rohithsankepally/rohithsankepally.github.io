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
Let us say we have a very big street represented as a linestring. Let us suppose that I am interested in a particular part of the street and not the whole street. In such cases we may have to segment the bigger street into small streets.

A figure explaining the above:
<p align="center">
<img src="{{ site.baseurl }}/images/404.jpg">
</p>

