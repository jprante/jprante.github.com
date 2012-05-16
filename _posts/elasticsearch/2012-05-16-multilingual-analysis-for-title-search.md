---
layout: post
category: lessons
title: "Elasticsearch multilingual index analysis setup for library catalog search"
tagline: "by Jörg Prante"
tags: [intro, elasticsearch, tutorial, howto]
comments: true
---

![TowerOfBabel](/assets/images/Brueghel-tower-of-babel.jpg)

When we recently explored ISBN search, we got some appetite for books, and now we want more, advanced book search. Let's turn to a library catalog. How do libraries, book stores, publishers and vendors offer a convenient, friendly search for book titles? ISBN is obviously not enough. Favorites for search are title keywords.

In this lesson, we focus on the most important search category, the *known item search*. Known item search is a very old concept \[1\], it's as old as information retrieval as an academic discipline. It is evident that most people are in need for successful known item search in a catalog. But how can we implement such a title search with Elasticsearch?

What the users are longing for
-----------------------------

At first, we should remind the characteristics and current expectations of a known-item search from a user's point of view.

- very fast response

- it shall be easy-as-pie, require as few title keywords as possible to obtain relevant results

- it shall give back all the titles that match, but only titles that are most preferred on top

- it shall allow precise title search, where title keyword search is separated from other kinds of searches like author name search, for instance if a person name appears in a title

- it shall not be restricted to a localization environment, for example a certain language

We will focus on the latter point in this lesson.

Nature of title word forms
--------------------------

The nature of title word forms in a library catalog is one of the most challenging that can be imagined. Not only the language of domestic books, which are in majority, but languages from all over the world may appear in the title field, not mentioning artificial and ancient word forms. Moreover, in an online environment, different users from all over the world needs to be served appropriately to find the books in their own language. Ths challenge is often called multilingual search.

Yale university library, which is undoubtedly one of the most experienced library using original languages in cataloging, has examined this topic [in depth](https://collaborate.library.yale.edu/yufind/public/FinalReportPublic.pdf) and concluded

> *The current state of development in Unicode, Lucene/Solr and Solrmarc has advanced to a level that makes providing the high-priority original script support features entirely feasible  for a relatively modest cost.*

Now, let's tell our plan with Elasticsearch:

- because most of the users of a german library catalog are german speaking, we implement "expanded umlaut" search: german umlauts (ä, ö, ü), when absent on keyboards, are often entered as ae, oe, or ue, or simply abbreviated as a, o, u. Note this tri-fold search is a very unique requirement, only existent in german language. In Elasticsearch, we select the snowball stemmer with German2 language setting to cope with this \[2\]

- because of the wealth of international languages, we add Unicode-based character folding with the help of the International Components for Unicode (ICU) Elasticsearch plugin

- we fix certain deficiencies of the Snowball stemmer by additional indexing of the unchanged word form

Here is the Index Analysis setting we choose:

	"index" : {
	   "analysis" : {
	      "filter" : {
	         "germansnow" : {
	            "language" : "German2",
	            "type" : "snowball"
	         }
	      },
	      "analyzer" : {
	         "german" : {
	            "filter" : [
	               "germansnow",
	               "icu_folding"
	            ],
	            "type" : "custom",
	            "tokenizer" : "icu_tokenizer"
	         },
	         "default" : {
	            "sub_analyzers" : [
	               "standard",
	               "german"
	            ],
	            "type" : "combo"
	         }
	      }
	   }
	}

Let's describe what we do here.

1. We define a ``germansnow`` analysis filter. The filter is of type ``snowball`` and is configured to use the language ``German2``, which allows for the infamous umlaut expansion.

2. We define two analyzers, one for german snowball with unicode folding filter appended, the other for combining the standard analyzer and the german analyzer.

3. By setting the latter analyzer name to ``default``, it is enabled by default. 

4. Note that the order of applying analyzers and filters is significant.

The Köln Test
------------
Let's test this index analysis setting. Elasticsearch has a wonderful diagnostic tool, the Analyze API, for testing how words get transformed before they got indexed.

We check the tri-folding of Köln by using the word forms ``köln``, ``koeln``, and ``koln``. What we require is equivalence of the indexed forms. And we can find that each form is indexed with the reduced (stemmed) form ``koln``.

	curl 'localhost:9200/test/_analyze?text=k%c3%b6ln&pretty=1'
	{
	  "tokens" : [ {
	    "token" : "köln",
	    "start_offset" : 0,
	    "end_offset" : 4,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "koln",
	    "start_offset" : 0,
	    "end_offset" : 4,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}


	curl 'localhost:9200/test/_analyze?text=koeln&pretty=1'
	{
	  "tokens" : [ {
	    "token" : "koeln",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "koln",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

	curl 'localhost:9200/test/_analyze?text=koln&pretty=1'
	{
	  "tokens" : [ {
	    "token" : "koln",
	    "start_offset" : 0,
	    "end_offset" : 4,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "koln",
	    "start_offset" : 0,
	    "end_offset" : 4,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

Note the original word form is always indexed, together with the stemmed form. So, by showing the analyzed forms of the three word forms of Köln, the requirement of indexing and searching german titles in all various forms is fulfilled.

Fixing overstemming
-------------------

Let's check for something we call the overstemming phenomenon. It occurs when a language stemmer is producing stemmed forms that are too short for a reasonable search. As a consequence, searches will miss some hits. Quoting Snowball German2 stemmer: 

> *Of the native German words, about half seem to be improved by the variant stemming, and the other half made worse. In any case the differences are little more than one word per thousand among the native German words.*

The french word ``baudelairienne`` is an example. If being applied to the Snowball German2 stem algorithm, the last character ``e`` is dropped.

	curl 'localhost:9200/test/_analyze?text=baudelairienne&pretty=1'
	{
	  "tokens" : [ {
	    "token" : "baudelairienne",
	    "start_offset" : 0,
	    "end_offset" : 14,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "baudelairienn",
	    "start_offset" : 0,
	    "end_offset" : 14,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

It does not surprise us because german stemming is not meant for french words. Interestingly, if the word is given in uppercase, the stem algorithm gives another result - the character is no longer dropped. This is just an example for a possible stem algorithm fault.

	curl 'localhost:9200/test/_analyze?text=BAUDELAIRIENNE&pretty'
	{
	  "tokens" : [ {
	    "token" : "baudelairienne",
	    "start_offset" : 0,
	    "end_offset" : 14,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "baudelairienne",
	    "start_offset" : 0,
	    "end_offset" : 14,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

Now, with the help of the magic combo analyzer, we have combined the standard analysis with the overstemming german Snowball analyzer. As a result, both forms appear in the index, and search hits on both are guaranteed.

Japanese word forms
-------------------

Let's try japanese word forms. Searches for the name of the japanese capital can be observed in three forms, US-ASCII, transliterated, and japanese: Tokyo, Tōkyō, and 東京.

Unicode folding leads to the effect that transliterated forms get reduced to their base US-ASCII word forms. So, Tōkyō becomes Tokyo in the index.

	curl 'localhost:9200/test/_analyze?text=tokyo&pretty'
	{
	  "tokens" : [ {
	    "token" : "tokyo",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "tokyo",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

	curl 'localhost:9200/test/_analyze?text=t%c5%8dky%c5%8d&pretty'
	{
	  "tokens" : [ {
	    "token" : "tōkyō",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "tokyo",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

	curl 'localhost:9200/test/_analyze?text=%e6%9d%b1%e4%ba%ac&pretty'
	{
	  "tokens" : [ {
	    "token" : "東",
	    "start_offset" : 0,
	    "end_offset" : 1,
	    "type" : "<IDEOGRAPHIC>",
	    "position" : 1
	  }, {
	    "token" : "東",
	    "start_offset" : 0,
	    "end_offset" : 1,
	    "type" : "<IDEOGRAPHIC>",
	    "position" : 1
	  }, {
	    "token" : "京",
	    "start_offset" : 1,
	    "end_offset" : 2,
	    "type" : "<IDEOGRAPHIC>",
	    "position" : 2
	  }, {
	    "token" : "京",
	    "start_offset" : 1,
	    "end_offset" : 2,
	    "type" : "<IDEOGRAPHIC>",
	    "position" : 2
	  } ]
	}

Note that japanese language symbols are stored as ideographic\[3\] tokens, where delimiting space is treated different from the romanized word forms in the Lucene index.

Hebrew
------

Let's try some hebrew. This is how the hebrew characters of the city name *Tel-Aviv* אביב-יפ is indexed. Note the hebrew alphabet is related to the latin alphabet.

	curl 'localhost:9200/test/_analyze?text=%d7%9c%2d%d7%90%d7%91%d7%99%d7%91&pretty=1'
	{
	  "tokens" : [ {
	    "token" : "ל",
	    "start_offset" : 0,
	    "end_offset" : 1,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "ל",
	    "start_offset" : 0,
	    "end_offset" : 1,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "אביב",
	    "start_offset" : 2,
	    "end_offset" : 6,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  }, {
	    "token" : "אביב",
	    "start_offset" : 2,
	    "end_offset" : 6,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  } ]
	}

Arabic
------

Finally, let's try analysis of the original script and romanized form of the name of the egyptian capital, al-Qāhira ‏القاهرة‎

	 curl 'localhost:9200/test/_analyze?text=%e2%80%8f%d8%a7%d9%84%d9%82%d8%a7%d9%87%d8%b1%d8%a9%e2%80%8e&pretty'
	{
	  "tokens" : [ {
	    "token" : "القاهرة‎",
	    "start_offset" : 1,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "القاهره",
	    "start_offset" : 1,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  } ]
	}

	curl 'localhost:9200/test/_analyze?text=al-Q%c4%81hira&pretty'
	{
	  "tokens" : [ {
	    "token" : "al",
	    "start_offset" : 0,
	    "end_offset" : 2,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "al",
	    "start_offset" : 0,
	    "end_offset" : 2,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "qāhira",
	    "start_offset" : 3,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  }, {
	    "token" : "qahira",
	    "start_offset" : 3,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  } ]
	}

	curl 'localhost:9200/test/_analyze?text=al-Qahira&pretty'
	{
	  "tokens" : [ {
	    "token" : "al",
	    "start_offset" : 0,
	    "end_offset" : 2,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "al",
	    "start_offset" : 0,
	    "end_offset" : 2,
	    "type" : "<ALPHANUM>",
	    "position" : 1
	  }, {
	    "token" : "qahira",
	    "start_offset" : 3,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  }, {
	    "token" : "qahira",
	    "start_offset" : 3,
	    "end_offset" : 9,
	    "type" : "<ALPHANUM>",
	    "position" : 2
	  } ]
	}

We reached the end of this lesson. If you are curious about the effectiveness of our index setting, you can try more languages. 

Summary
=======

We explored indexing words for title searches in a library catalog and learned about the plentitude of languages we have to deal with when we want to offer appropriate search experience to users from all over the world.

By setting up Elasticsearch with a customized index analysis setting, we learned

- that we can succesfully manage difficult localized requirements such as german word stemming with additional umlaut expansion

- that it is possible to filter tokens not only by word stemming but also by unicode folding provided by the Elasticsearch ICU plugin \[4\]

- that words can get overstemmed and need fixing by combining analyzers with the help of the Elasticsearch Combo Analysis plugin \[5\]

- that we can check the indexed word forms with the help of the Elasticsearch analysis API.

Elasticsearch turned out to be a very flexible and powerful tool for library catalog searches. 

Author name search is also an interesting topic. In one of the next blog entries we will discuss how to use library authority files such as VIAF or Gemeinsame Normdatei (GND) to enhance Elasticsearch library catalog searches for author names. See you then!

\[1\]  Tagliacozzo, R., Kochen, M. (1970/12)."Information-seeking behavior of catalog users." Information Storage and Retrieval 6(5): 363-381.

\[2\] <http://snowball.tartarus.org/algorithms/german2/stemmer.html>

\[3\] <http://en.wikipedia.org/wiki/CJK_Unified_Ideographs>

\[4\] <https://github.com/elasticsearch/elasticsearch-analysis-icu>

\[5\] <https://github.com/jprante/elasticsearch-analysis-combo>
