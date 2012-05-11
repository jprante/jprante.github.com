---
layout: post
category: lessons
title: "ISBN search with Elasticsearch"
tagline: "by Jörg Prante"
tags: [intro, elasticsearch, tutorial, howto]
comments: true
---

The challenge: ISBN search
==========================

As most of you know, International Standard Book Numbers (ISBN) come in two representations. 
First, there is a legacy form, ISBNs in short form, ony ten characters in length.
Those ISBNs appeared exclusively before 2007 and are still very common to most of us.
The fact is that ISBNs were expanded to thirteen characters, since the possible numbers
given by the ISBN specification tended to run out of space. As a matter of fact, each old-style ISBN is now convertible to the new form, and all book merchands, the printing industries, and libraries were advised to only use the new ISBN form. The only ones who can not be advised are the public, that is you and me, who like to search for whatever ISBN comes into the mind.

Anyway, there is a challenge in searching for ISBNs just because of that. There are at least two forms of the same ISBN (some of you who discover printable ISBN forms on the book covers would say there are even more) that need to be searched in parallel. Now, if you carefully store each ISBN form in separate fields, you need to reformulate your search engine queries, guiding the search to all of the various ISBN fields you crafted tediously for your unique best-of-breed book search engine as your users come along.

The thing is, reformulating queries just because of different representations of the same thing is not very clever. Such queries can soon become complex and take search time. Boolean search operations are not very welcome in the eyes of performance knights - you know, they come at a price. And, you need to teach your clients about what fields to use when. This all will take time and energy.

But how to avoid boolean query management, like or'ing all of the fields you want the user to look into? One of the obvious ideas is to combine the ISBN forms into a single field. 

You might reply, well, good idea, but combining the forms into a single field is not feasible, because this hinders from searching for each form separately. You won't be able to deduce from a user's input what kind of ISBN form was entered. And it's not hard to imagine a requirement where the user alone is up to decide whether to search for a legacy or new ISBN form. Or, you have an EAN barcode device connected to your search engine, that needs exact evidence about the fact whether an EAN (which is for books equivalent to ISBN-13) exists in the inventory or not.

Now you could resign and go back into your dark and gloomy office, hide from the sun, lock in for months and write nasty and slow code scattered with boolean operations for searching ISBNs. Thinking about waiting for your clients coming by with one legitimate change request after another because suddenly they all turned into ISBN/ISSN/ISMN experts won't set you into a state of much relief. Or, you could start over and try Elasticsearch and set up a smart configuration that is able to exactly do what all the people will want from you.

Smart Elasticsearch configuration for ISBN objects
--------------------------------------------------

Elasticsearch JSON offers hierarchical organized properties for your fields in the index, so they can be seen as a structure in an object-like manner. Under such a structure, you can sum up child fields that shall belong to a common parent field. 

Your first decision is to put all the ISBN forms under a parent called ``identifier``. Now, you can index many identifiers in a row, just by using a JSON array. 

Your second decision is adding children to the ``identifier`` field, named ``isbn``, ``ean``, and ``isbnprintable``. There can be even more children in the future. Later, you can read the Elasticsearch JSON source that is sent back as a response to your search and extract the information about which forms of ISBN are connected to each other. Yes, you will need to do that analysis sooner or later, because of a very simple reason: some books might have more than one ISBN.

Let us restrict ourselves to just one ISBN for simplicity. We assume an ISBN with three different forms. For example, let's take ISBN 1-932394-28-1. This form is printed on the book you just have in your hands. From this ISBN form, you derive the form 1932394281 (some call it a normalized form, but ISBNs are not really normalized like this) and, because you are a professional, the form 9781932394283, by recalculating the checksum for ISBN-13. The book was published back in 2005, so the publisher did not know about EAN (or ISBN-13), but you remember you were obliged to use this additional form in current applications, too.

So you have designed your ISBN JSON structure for the Elasticsearch document. Let's see:

	{
	    "identifier" : {
	        "isbn" : "1932394281",
	        "ean" : "9781932394283",
	        "isbnprintable" : "1-932394-28-1"
	    }
	}

Redirection with ``index_name``
-------------------------------

Elasticsearch has two more smart features in the mappings for an index. One is called *index_name* and the other one is *multi_field*. By combining these two features we are able to accomplish our mission.

With *index_name*, ``ean`` values can be redirected to the same index as ``isbn``. But, by setting ``multi_field``, the EAN is still kept in a separate index that we call ``eanonly``. Finally, we add a mapping for ``isbnprintable`` for the third form of representation.

Just have a look at the ISBN demo mapping:

	{
	   "mappings" : {
	      "_default_" : {
	         "_source" : {
	            "enabled" : true
	         },
	         "_all" : {
	            "analyzer" : "default",
	            "enabled" : true
	         },
	         "properties" : {
	            "identifier" : {
	               "dynamic" : false,
	               "properties" : {
	                  "isbn" : {
	                     "type" : "string",
	                     "include_in_all" : true
	                  },
	                  "ean" : {
	                     "type" : "multi_field",
	                     "fields" : {
	                         "ean" : {
	                             "index_name" : "isbn",
	                             "type" : "string"
	                         },
	                         "eanonly" : {
	                             "type" : "string"
	                         }
	                     }
	                  },
	                  "isbnprintable" : {
	                     "index_name" : "isbn",
	                     "index" : "not_analyzed",
	                     "type" : "string"
	                  }
	               }
	            }
	         }
	      }
	   }
	}


Proving the ISBN search
-----------------------

Our first test case are users who submit queries and do not like to search in a special ISBN field. They just use a single search field, pasting ISBNs in there, and send it off.

	{
	    "query" : {
	         "text" : {
	             "_all" : "1932394281"
	         }
	    }
	}

	{
	    "query" : {
	         "text" : {
	             "_all" : "9781932394283"
	         }
	    }
	}

Because of the nature of the ``include_in_all`` of the ``isbn`` field, both ISBN and EAN are included into the ``_all`` field. We did the first case.

Now, let's satisfy our next users who prefer entering ISBNs into a special ISBN field, regardless of what form. We send their searches into the field ``identifier.isbn``.

	{
	    "query" : {
	         "text" : {
	             "identifier.isbn" : "1932394281"
	         }
	    }
	}

	{
	    "query" : {
	         "text" : {
	             "identifier.isbn" : "9781932394283"
	         }
	    }
	}

Because the ``ean`` field is redirected to the ``isbn`` field, this works well.

And finally, let's ensure that our barcode scan devices get only hits on EANs in our book inventory.

	{
	    "query" : {
	         "text" : {
	             "identifier.ean.eanonly" : "9781932394283"
	         }
	    }
	}

This will not return a hit, as expected:

	{
	    "query" : {
	         "text" : {
	             "identifier.ean.eanonly" : "1932394281"
	         }
	    }
	}

We left an example with ``isbnprintable`` search as an exercise.

Summary
=======

With Elasticsearch, challenges like ISBN search, where multiple representations of the same thing need to get indexed nearby, can be tackled in an elegant and powerful way. You learned 

- that you can design an object-like structure, where you can later easily deduce what forms are connected to a common parent

Then you found out that 

- with a suitable Elasticsearch mapping, you can redirect field values to other indexes so they get appended to each other

And finally

- with the ``multi_field`` property, you realized that you can even keep such values and put them into a separate index to avoid queries returning false hits.

You can find a shell-script curl-based executable of the above example [here](https://gist.github.com/2662060).

Now, happy book searching with Elasticsearch!


