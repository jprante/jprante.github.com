---
layout: post
category: lessons
title: "Elasticsearch for bibliographic authorities"
tagline: "by Jörg Prante"
tags: [intro, elasticsearch, tutorial, howto]
comments: true
---

![Leibnix](/assets/images/Gottfried_Wilhelm_von_Leibniz.jpg)

*Leibniz, as a librarian, used subject headings in Bibliotheca boineburgica to establish 'locus communis'*

Libraries are challenged with many search tasks since decades. When producing new catalog entries, it has been observed very soon since the mid of the 20th century that it is useful to save work by automatically obtaining already existing entries from the catalog. But how do library professionals find existing catalog entries efficiently?

There are several inventions of library professionals in this field:

- Metadata. Yes, librarians invented 'intellectual' metadata. Metadata allows for excerpting important parts from all the information of a media resource like a book. Librarians invented not only subject headings but even compact title and author entities a few centuries before, revolutionizing the publishing industry.

- Controlled metadata. Catalog rules were invented to enforce librarians using always the same verbal forms for entity names in catalogs, the *authorities*. The form selected for main entries in the catalog is also called *heading*. One of the most successful controlled metadata is history are authority files or authority lists \[1\], such as the Library of Congress Subject Headings (LCSH). In Germany, the first automated authority file was the Körperschaftsdatei of the Zeitschriftendatenbank, invented in the early 1970s.

- The exchange of controlled metadata. Librarians also invented methods to fusion metadata in large databases for harmonization of authority files. A recent impressive one is VIAF, the Virtual International Authority File. Many nations worldwide can align their entity vocabulary in authority files with the help of VIAF.

- The integration of controlled metadata into the Semantic Web. By using simple *Linked Data* techniques as postulated by Tim Berners-Lee, librarians joined the efforts of the W3C to establish the web as a huge global online information database of facts and statements under inclusion of library catalogs.

In Germany, the Deutsche Nationalbibliothek (DNB) just harmonized defragmented german authority files into a single authory file called *Gemeinsame Normdatei* (GND). DNB adjusted the catalog rules and assigned URIs to authority entities. This has been recognized as a remarkable step in authority metadata management in german libraries, which has a long history \[2\], \[3\].

Efficient authority search
--------------------------

But librarians did not care too much for efficient authority search as they go along with their inventions. Up to now, many german libraries offer only rudimentary OPACs with insufficent search capabilities for authority files. Such search interfaces are always affected of impedance mismatches due to traditional data formats for cataloging like MARC or MAB.

Lateley, Deutsche Nationalbibliothek prepared GND in an RDF Turtle dump \[4\], which fits nicely into W3C semantic web and is perfect for indexing into modern search engines. The RDF turtle dump is licensed into the public domain (CC0), so everyone can use the data and enrich the data to the fullest extent without having to ask for permission.

Updates are offered by DNB via an Open Archives Initiative (OAI) interface on a daily basis. This is a "pull" mechanism with limited scalability.

It wasn't too hard to implement an indexer that indexes RDF turtle data in Elasticsearch. The rough components are

- a turtle parser

- a namespace environment for parsing

- a JSON serializer for RDF graphs

- a bulk indexer for Elasticsearch

- an OAI river for Elasticsearch for receivng updates

Here is an example command line how to index the full data set into Elasticsearch. The tool is called 
``org.xbib.tools.indexer.ElasticsearchGNDIndexer``, the source GND file is ``GND.ttl.gz``, and the target is an Elasticsearch cluster reachable by the network interface where ``hostname`` is bound to, with the index ``gnd`` and type ``gnd``

Let's first create an index ``gnd`` and switch off automatic date field detection.

	curl -XPUT 'localhost:9200/gnd' -d '{ "mappings" : { "gnd" : { "date_detection" : false } } }'

Then let's start indexing.

	java -cp target/xbib-tools-1.0-SNAPSHOT-elasticsearchgndindexer.jar org.xbib.tools.indexer.ElasticsearchGNDIndexer --gndfile file:///Users/joerg/Downloads/GND.ttl.gz --elasticsearch "es://hostname:9300" --index gnd --type gnd

	Jun 01, 2012 11:04:08 PM org.xbib.elasticsearch.ElasticsearchConnection findClusterName
	Information: cluster name found in URI es://hostname:9300?es.cluster.name=oaicluster
	Jun 01, 2012 11:04:08 PM org.xbib.elasticsearch.ElasticsearchSession createClient
	Information: starting discovery for clustername oaicluster
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.ElasticsearchSession findNodes
	Information: adding hostname address for transport client = inet[Jorg-Prantes-MacBook-Pro.local/192.168.1.113:9300]
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.ElasticsearchSession findNodes
	Information: hostname address added
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.ElasticsearchSession findNodes
	Information: addresses = [inet[Jorg-Prantes-MacBook-Pro.local/192.168.1.113:9300]]
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.ElasticsearchSession findNodes
	Information: connected nodes = [[#transport#-1][inet[Jorg-Prantes-MacBook-Pro.local/192.168.1.113:9300]]]
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.ElasticsearchSession findNodes
	Information: new connection to #transport#-1 
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 1 requests currently active)
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 2 requests currently active)
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 3 requests currently active)
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 4 requests currently active)
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 5 requests currently active)
	Jun 01, 2012 11:04:09 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 6 requests currently active)
	Jun 01, 2012 11:04:10 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 7 requests currently active)
	Jun 01, 2012 11:04:10 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 8 requests currently active)
	Jun 01, 2012 11:04:10 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 9 requests currently active)
	Jun 01, 2012 11:04:10 PM org.xbib.elasticsearch.BulkWrite processBulk
	Information: submitting new bulk index request (100 docs, 10 requests currently active)
	Jun 01, 2012 11:04:10 PM org.xbib.elasticsearch.BulkWrite processBulk
	[..]

The process took roughly 80 minutes on a MacBook Pro for ~9,5 millions of different RDF subjects. The result is a 7.5 GB index in 5 shards (five distributed Lucene indices).

![elasticsearch-gnd-sample-index](/assets/images/elasticsearch-gnd-sample-index.png)

Here is an example of a quick search. The result contains two entries of the GND, one from the former Schlagwortdatei and the Gemeinsame Körperschaftsdatei.

	curl -XGET 'localhost:9200/gnd/gnd/_search?q=hbz%20nordrhein'

	[...]
		         {
	             "_source" : {
	               "gnd:preferredNameForTheCorporateBody" : "Hochschulbibliothekszentrum des Landes Nordrhein-Westfalen <Köln>",
	               "gnd:geographicAreaCode" : "http://d-nb.info/standards/vocab/gnd/geographic-area-code#XA-DE",
	               "rdf:type" : "http://d-nb.info/standards/elementset/gnd#CorporateBody",
	               "gnd:oldAuthorityNumber" : "(DE-588b)2047974-8",
	               "gnd:gndIdentifier" : "2047974-8",
	               "gnd:placeOfBusiness" : "http://d-nb.info/gnd/4031483-2",
	               "gnd:variantNameForTheCorporateBody" : [
	                  "Hochschulbibliothekszentrum NRW <Köln>",
	                  "hbz",
	                  "Hochschulbibliothekszentrum <Köln>"
	               ]
	            },
	            "_score" : 1.4516485,
	            "_index" : "gnd",
	            "_id" : "http://d-nb.info/gnd/2047974-8",
	            "_type" : "gnd"
	         },
	         {
	            "_source" : {
	               "gnd:preferredNameForTheCorporateBody" : "Hochschulbibliothekszentrum des Landes 	Nordrhein-Westfalen",
	               "gnd:gndSubjectCategory" : "http://d-nb.info/vocab/gnd-sc#6.7",
	               "rdf:type" : "http://d-nb.info/standards/elementset/gnd#CorporateBody",
	               "gnd:oldAuthorityNumber" : "(DE-588c)4194078-7",
	               "gnd:gndIdentifier" : "4194078-7",
	               "gnd:geographicAreaCode" : "http://d-nb.info/standards/vocab/gnd/geographic-area-code#XA-DE-NW",
	               "gnd:spatialAreaOfActivity" : "http://d-nb.info/gnd/4042570-8",
	               "gnd:topic" : "http://d-nb.info/gnd/4132773-1",
	               "gnd:broaderTermInstantial" : "http://d-nb.info/gnd/4630294-3",
	               "gnd:placeOfBusiness" : "http://d-nb.info/gnd/4031483-2",
	               "gnd:variantNameForTheCorporateBody" : "HBZ"
	            },
	            "_score" : 1.4498183,
	            "_index" : "gnd",
	            "_id" : "http://d-nb.info/gnd/4194078-7",
	            "_type" : "gnd"
	         },
	[...]

As you noticed by this quick search, GND may contain duplicates from the predecessing authority files.

It is not a big challenge to transform Elasticsearch JSON search results back to RDF, to N-triples, Turtle, or RDF/XML format, for exporting documents.

As Elasticsearch is only used as a coarse-grained triple store with documents containing sub-graphs of RDF triples with common subject URIs, it is rather tough to implement a SPARQL interface covering all the documents. So, for a full semantic web approach, you have to use a SPARQLified triple store like 4store. The disadvantage of tripe store is the slow searching and the response time latency.

A river for OAI
---------------

With rivers, Elasticsearch can be extended to fetch data from external sources. I have implemented an Open Archives Initiative river \[5\].

For example an OAI river for GND updates will look like:

	curl -XPUT 'localhost:9200/_river/gnd/_meta' -d '{"type":"oai", "oai":{"url":"http://services.dnb.de/oai/repository","set":"authorities","metadataPrefix":"RDFxml"}, "index":{"index":"gnd","type":"gnd"}}'

This will pull GND data on a regular basis from DNB, for example, each hour.

An OAI river in action will give messages like this in the Elasticsearch logfile.

	[2012-06-02 12:21:47,252][INFO ][river.oai                ] [Solarr] [oai][gnd] OAI harvest: URL [http://services.dnb.de/oai/repository] set [authorities] metadataPrefix [RDFxml] from [2012-06-02T11:00:00Z] until [2012-06-02T12:00:00Z] resumptionToken [null]
	Jun 02, 2012 12:21:47 PM org.xbib.io.http.netty.HttpOperation prepareExecution
	Information: method=[GET] uri=[http://services.dnb.de/oai/repository] parameter=["metadataPrefix=RDFxml"; "set=authorities"; "verb=ListRecords"]
	[2012-06-02 12:21:47,887][INFO ][river.oai                ] [Solarr] [oai][gnd] submitting new bulk index request (13 docs, 1 requests currently active)
	[2012-06-02 12:21:47,892][INFO ][river.oai                ] [Solarr] [oai][gnd] waiting for 1 active bulk requests
	[2012-06-02 12:21:47,910][INFO ][river.oai                ] [Solarr] [oai][gnd] bulk index success (23 millis, 13 docs, total of 26 docs)
	[2012-06-02 12:21:47,913][INFO ][river.oai                ] [Solarr] [oai][gnd] next harvest, waiting 1h, URL [http://services.dnb.de/oai/repository] set [authorities] metadataPrefix [RDFxml]

By receiving updates an a regular basis, the GND search solution is complete.

Summary
-------

With powerful search capabilities, Elasticsearch can be turned into an important back-bone for the future libary catalog search infrastructure for the following reasons:

- 9,5 million GND entries are indexed in minutes

- search results return in milliseconds

- growing search and index requirements are not a big challenge

- a wealth of additional search features such as facet search (drill-down) is availiable which is known to be useful for authority-controlled library catalogs

- by keeping the RDF data structure intact, Elasticsearch offers scalable RDF literal search

- using the schema-free and multi-tenancy property of Elasticsearch, it is a platform for maintaining several catalogs in multiple indexes and for aggregating related metadata

Aggregating is not limited to authority files. Elasticsearch could also aggregate holdings from many libraries. It is even possible to attach full SRU and OAI capabilities to Elasticsearch, turning Elasticsearch into a complete front-end for traditional library systems. This will be shown in one of the subsequent postings here.

This posting is just scraping the surface, but search engines like Elasticsearch can help pushing libraries to a new level of global data harmonization, for example, union catalogs on steroids, or an inter-library loan index. And the improved results of such a globalization will be useful all of you, as users of public and academic libraries, all around the world.

References
----------

\[1\] Allen Kent, Harold Lancour: Encyclopedia of Library and Information Science. New York,
Dekker, 1969, Vol. 2, p. 132−138.

\[2\] <http://bibliothekarisch.de/blog/2012/05/02/gnd-loest-pnd-gkd-und-swd-ab/>

\[3\] <http://repositorium.uni-osnabrueck.de/bitstream/urn:nbn:de:gbv:700-201001304634/1/ELibD27_normdateien.pdf>

\[4\] <http://www.dnb.de/DE/Service/DigitaleDienste/LinkedData/linkeddata_node.html>

\[5\] <https://github.com/jprante/elasticsearch-river-oai/>
