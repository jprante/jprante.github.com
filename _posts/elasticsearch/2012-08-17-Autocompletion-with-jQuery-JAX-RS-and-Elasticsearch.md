---
layout: post
category: applications
title: "Autocompletion with jQuery, JAX-RS and Elasticsearch"
tagline: "by Jörg Prante"
tags: [autocompletion, assistive technology, jQuery, jax-rs, elasticsearch]
comments: true
---

![Typing](/assets/images/autocomplete.jpeg)

*<http://www.flickr.com/photos/atomicshark/144630706/>*

## What is Autocompletion?

Autocompletion is the provision of words that are frequently used in response to a user's keystrokes. It was first invented to assist people with physical disabilities to improve their typing speed. \[1\]

Autocomplete or word completion works so that when the writer writes the first letter or letters of a word, a program predicts one or more possible words as choices.

In this text, we will show how to implement a solution for autocompletion, a search frontend for Elasticsearch. It consists of a jQuery page, presenting a search form to the user, submitting query terms to a gateway service, and receiving JSON results for the autocompleted word list.

For our use case, we build a library catalog title autocompletion, where 80% of user queries are searches for title keywords. While typing, users should immediately got aware of titles matching their requests, and Elasticsearch should do the hard work to filter the relevant documents in near realtime.

## The n-gram method

How will the autocompletion works? One approach is changing the normal indexing of title words. A well-known method is using n-grams.

In the fields of computational linguistics, an n-gram is a contiguous sequence of n items from a text sequence. N-gram matching implementation is simple and provides good performance. The algorithm is based on the principle: if a word A matches a word B containing some errors, they will most likely have at least one common substring of length 'n'.

At indexing step, the word is partitioned into n-grams, the word is added to lists that correspond each of these n-grams. At search time, the query is also partitioned into n-grams, and for each of them corresponding lists are scanned using the metric.

One observation is that users of a library catalog enter characters from the front in the order the characters are written in the title word, so an effective improvement of n-grams is introducing *edged* n-grams, where the word n-grams are always starting from one of the word edges, here, the front.

## The Elasticsearch setting

Note, we use a multilingual library catalog setup (multilingual setup is not the focus now so details are not discussed now). We add edge n-gram analysis by a filter we call **edgengramfilter**. It builds n-grams from the word front, with a minimum length of 2 characters and a maximum of 16 characters.

	"settings" : {
      "index" : {
         "analysis" : {
            "filter" : {
               "germansnow" : {
                  "type" : "snowball",
                  "language" : "German2"
               },
               "edgengramfilter" : {
                   "type" : "edgeNgram",
                   "side" : "front",
                   "min_gram" : 2,
                   "max_gram" : 16
               } 
            },
            "analyzer" : {
               "german" : {
                  "type" : "custom",
                  "tokenizer" : "lowercase",
                  "filter" : "germansnow"                  
               },
               "icu" : {
                  "type" : "custom",
                  "tokenizer" : "icu_tokenizer",
                  "filter" :  "icu_folding"

               },
               "default" : {
                  "sub_analyzers" : [
                     "german",
                     "icu",
                     "standard"
                  ],                  
                  "type" : "combo"
               },
               "icu_autocomplete" : {
                  "type" : "custom",
                  "tokenizer" : "icu_tokenizer",
                  "filter" : [ "icu_folding", "edgengramfilter" ]
               },
               "autocomplete" : {
                  "sub_analyzers" : [
                     "german",
                     "icu_autocomplete",
                     "standard"
                  ],                  
                  "type" : "combo"
               }
            }
         }
      }
   }

## Checking the edge n-gram creation

With the analyzer API, we can observe the edge n-gram filter if it works like we want to. Note the maximum length of the grams should be relatively small. The number of extra words created is significant for the expected index size growth.

	curl -XGET 'localhost:9200/hbztest//_analyze?pretty&analyzer=icu_autocomplete&text=wissen'
	{
	  "tokens" : [ {
	    "token" : "wi",
	    "start_offset" : 0,
	    "end_offset" : 2,
	    "type" : "word",
	    "position" : 1
	  }, {
	    "token" : "wis",
	    "start_offset" : 0,
	    "end_offset" : 3,
	    "type" : "word",
	    "position" : 2
	  }, {
	    "token" : "wiss",
	    "start_offset" : 0,
	    "end_offset" : 4,
	    "type" : "word",
	    "position" : 3
	  }, {
	    "token" : "wisse",
	    "start_offset" : 0,
	    "end_offset" : 5,
	    "type" : "word",
	    "position" : 4
	  }, {
	    "token" : "wissen",
	    "start_offset" : 0,
	    "end_offset" : 6,
	    "type" : "word",
	    "position" : 5
	  } ]
	}

Now let's imagine we need to index a library catalog, where the title information is additionally directed to an autocompletion field **xbib:titleAutocomplete**, for example

	"xbib:title" : {
       "type" : "string",
       "type" : "multi_field",
       "fields" : {
             "xbib:title": {
                 "type" : "string",
                 "index_name" : "dc:title"
             },
             "xbib:titleAutocomplete": {
                 "type" : "string",
                 "index_analyzer" : "icu_autocomplete",
                 "include_in_all" : false
             }
        }
    }

You don't need to touch your indexing code, just adding the field into the mapping is all Elasticsearch requires.

## An Autocompleter JAX-RS gateway

A JAX-RS gateway is used to separate the jQuery and web design folks from direct access to the Elasticsearch cluster. Hence, you set up a middleware that is proxying valid autocomplete requests to the Elasticsearch backend that may be securely locked away in the darkest corners of your hosting facility.

In this working JAX-RS code, you can see how JSON data from an Elasticsearch query result can be generated directly into an JAX-RS streaming outputstream. 
The hard part of the work is encapsulated in service classes ElasticsearchSession and QueryResultAction which are not the focus of this article.

The autocompletion requests are formed by three input parameters: **field** is the field to be queried in the Elasticsearch index, **term** is the typed input of the user, and **resultfields** are the fields that Elasticsearch should return. 

	package jaxrs;

	import java.io.IOException;
	import java.io.OutputStream;
	import java.util.List;
	import javax.servlet.http.HttpServletResponse;
	import javax.ws.rs.FormParam;
	import javax.ws.rs.POST;
	import javax.ws.rs.Path;
	import javax.ws.rs.Produces;
	import javax.ws.rs.WebApplicationException;
	import javax.ws.rs.core.Context;
	import javax.ws.rs.core.MediaType;
	import javax.ws.rs.core.MultivaluedMap;
	import javax.ws.rs.core.Response;
	import javax.ws.rs.core.StreamingOutput;
	import javax.ws.rs.core.UriInfo;
	import javax.ws.rs.ext.ContextResolver;
	import org.xbib.elasticsearch.ElasticsearchSession;
	import org.xbib.elasticsearch.QueryResultAction;
	import org.xbib.logging.Logger;
	import org.xbib.logging.LoggerFactory;

	/**
	 * Elasticsearch autocomplete query bridge
	 */
	@Path("/autocomplete")
	public class Autocompleter {

	    @Context
	    ContextResolver<ElasticsearchSession> resolver;
	    private static final Logger logger = LoggerFactory.getLogger("es.query.autocomplete");
	    private final QueryResultAction action = new QueryResultAction();

	    @POST
	    @Path("/{index}")
	    @Produces(MediaType.APPLICATION_JSON)
	    public StreamingOutput createIndexPage(
	            @Context HttpServletResponse response,
	            @Context UriInfo uriInfo,
	            @FormParam("field") final String field,
	            @FormParam("term") final String term,
	            @FormParam("resultfields[]") final List<String> resultfields,            
	            @FormParam("filter") final String filter,
	            @FormParam("from") final int from,
	            @FormParam("size") final int size)
	            throws Exception {
	        return query(response, uriInfo, field, term, resultfields, filter, from, size);
	    }

	    @POST
	    @Path("/{index}/{type}")
	    @Produces(MediaType.APPLICATION_JSON)
	    public StreamingOutput createIndexTypePage(
	            @Context HttpServletResponse response,
	            @Context UriInfo uriInfo,
	            @FormParam("field") final String field,
	            @FormParam("term") final String term,
	            @FormParam("resultfields[]") final List<String> resultfields,            
	            @FormParam("filter") final String filter,
	            @FormParam("from") final int from,
	            @FormParam("size") final int size)
	            throws Exception {
	        return query(response, uriInfo, field, term, resultfields, filter, from, size);
	    }

	    private StreamingOutput query(
	            final HttpServletResponse response,
	            final UriInfo uriInfo,
	            final String field,
	            final String term,
	            final List<String> resultfields,
	            final String filter,
	            final int from,
	            final int size) throws IOException {
	        MultivaluedMap<String, String> pathParams = uriInfo.getPathParameters();
	        final String index = pathParams.getFirst("index");
	        final String type = pathParams.getFirst("type");
	        action.setQueryLogger(logger);
	        action.setSession(resolver.getContext(ElasticsearchSession.class));
	        action.setIndex(index);
	        if (type != null) {
	            action.setType(type);
	        }
	        action.setFrom(from);
	        action.setSize(size);
	        return new StreamingOutput() {
	            @Override
	            public void write(OutputStream output) throws IOException, WebApplicationException {
	                if (field == null) {
	                    throw new WebApplicationException(Response.status(400).entity("field must not be null").build());
	                }
	                if (term == null) {
	                    throw new WebApplicationException(Response.status(400).entity("term must not be null").build());
	                }
	                if (resultfields == null) {
	                    throw new WebApplicationException(Response.status(400).entity("resultfields must not be null").build());
	                }
	                action.setTarget(output);
	                StringBuilder sb = new StringBuilder();
	                if (filter != null) {
	                    sb.append("{\"fields\":");
	                    append(sb, resultfields);
	                    sb.append(",\"query\":{\"filtered\":{\"query\":{\"text\":{\"")
	                            .append(field).append("\":\"").append(term)
	                            .append("\"}},\"filter\":{\"term\":{").append(filter).append("}}}}}");
	                } else {
	                    sb.append("{\"fields\":");
	                    append(sb, resultfields);
	                    sb.append(",\"query\":{\"text\":{\"")
	                            .append(field).append("\":\"").append(term)
	                            .append("\"}}}");
	                }
	                action.search(sb.toString());
	            }
	        };
	    }

	    private void append(StringBuilder sb, List<String> list) {
	        boolean first = true;
	        sb.append("[");
	        for (String s : list) {
	            if (!first) {
	                sb.append(',');
	            }
	            sb.append("\"").append(s).append("\"");
	            first = false;
	        }
	        sb.append("]");
	    }
	}
	
	
# The jQuery front page

With the autocompleter written in Java JAX-RS, it is more easy to understand the jQuery part of the autocompletion architecture. This HTML page is a very basic jQuery example. It is enriched with HTML autocompletion because we find it useful to add icons to the library title catalog information to signal the media type to the user.

A somewhat complex part is the construction of the title label for linewise display in the drop down area. You can safely ignore that part. It is enough to know the structure of the metadata of a library catalog is hierarchically organized, where title information forms a sequence of objects.

Note how the image icon is excluded from the autocompletion value, since we should pass only textual information to the finl query construction, when the user selects an entry from the autocompletion offer.

	<html>
	    <head>
	        <meta charset="UTF-8"/>
	        <link type="text/css" href="css/smoothness/jquery-ui-1.8.22.custom.css" rel="Stylesheet" />
	        <script src="js/jquery-1.8.0.min.js" type="text/javascript"></script>
	        <script src="js/jquery-ui-1.8.22.custom.min.js" type="text/javascript"></script>
	        <script src="js/jquery.ui.autocomplete.html.js" type="text/javascript"></script>
	    </head>

	    <body>

	        <div>
	            <h1>Autocompletion Demo</h1>
	            <div>
	                <label for="search">Search </label>
	                <input id="search"/>
	            </div>
	        </div>

	        <script>
	            $(function() {
	                $("#search").autocomplete({
	                    html: true,
	                    source: function(request, response) {
	                        $.ajax({
	                            url: "http://index.hbz-nrw.de/query/services/autocomplete/hbz/title",
	                            type: "post",
	                            data: {
	                                field: "xbib:titleAutocomplete",
	                                term: request.term,
	                                resultfields: [ "dc:title","dc:contributor","dc:format" ],
	                                from: 0,
	                                size: 10
	                            },
	                            dataType: "json",
	                            success: function(data) {
	                                response($.map(data.hits.hits, function(item) {
	                                    var v = item.fields['dc:title'];
	                                    var w;
	                                    if( Object.prototype.toString.call(v) === '[object Array]' ) {
	                                        v.map(function(element, index, array) {
	                                            for (var i in element) {
	                                                if (element.hasOwnProperty(i)) {
	                                                    w = w ? w + " / " + element[i] :element[i];
	                                                }
	                                            }
	                                        })                                     
	                                    }
	                                    var label = 
	                                            (w ? w : item.fields['dc:title']['xbib:title']) + 
	                                            (item.fields['dc:title']['xbib:titleSub'] ? " / " + item.fields['dc:title']['xbib:titleSub'] : "") +
	                                            (item.fields['dc:contributor'] && item.fields['dc:contributor']['bib:namePersonal'] ? " / " + item.fields['dc:contributor']['bib:namePersonal'] : "") +
	                                            (item.fields['dc:contributor'] && item.fields['dc:contributor']['bib:nameCorporate'] ? " / " + item.fields['dc:contributor']['bib:nameCorporate'] : "") +
	                                            (item.fields['dc:contributor'] && item.fields['dc:contributor']['bib:nameConference'] ? " / " + item.fields['dc:contributor']['bib:nameConference'] : "");
	                                    return {
	                                        label: '&nbsp;<img alt="" src="images/format/' + item.fields['dc:format']['dcterms:medium'] + '.png"></img> ' + label,
	                                        value: label
	                                    }
	                                }));
	                            }
	                        });
	                    },
	                    minLength: 2
	                })
	            });
	        </script>

	    </body>
	</html>

## Examples

![Typing](/assets/images/autocomplete-example-1.png)

![Typing](/assets/images/autocomplete-example-2.png)

## References

\[1\] Swiffin, A. L., Arnott, J. L., Pickering, J. A., & Newell, A. F. (1987). Adaptive and predictive techniques in a communication prosthesis. Augmentative and Alternative Communication, 3, 181–191

