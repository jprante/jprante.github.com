---
layout: post
category: lessons
title: "How to write a simple plugin for Elasticsearch"
tagline: "by Jörg Prante"
tags: [intro, elasticsearch, tutorial, howto]
---

Elasticsearch is a distributed, scalable, RESTful open-source search engine implementation based on Lucene. One of the most attractive features of Elasticsearch is the extensiblity by writing plugins. 

Sometimes it is reported as a disadvantage that Elasticsearch is developed only by a single person, Shay Banon. But it should be emphasized that he equipped Elasticsearch with a convenient plugin mechanism for developers to encourage participation. The plugin mechanism allows other developers the extension of Elasticsearch with a lot of interesting features, such as adding analyzers, adding rivers, or adding connection mechanism to external libraries and systems, without the need to step into each and every detail of the Elasticsearch core.

In this writeup, I want to show how easy it is to make use of some of the most interesting features of Elasticsearch. The example explains

- how to extend Elasticsearch with a RESTful command called `_termlist`
- how to distribute and collect basic data structures within the shard architecture
- how to expose Lucene's IndexReader `terms()` method via the Elasticsearch API

The first step before we can get to the gory details is to step back and get an overview about the Elasticsearch philosophy.

- We need to understand the API that Elasticsearch uses to receive requests and to send responses.
- We need an action which distributes the job from a node to the shards.
- And finally, we need the implementation of the real low-level work on the shard level.

The plan of attack:

- Write a Plugin class, for attaching the plugin to the Elasticsearch core
- Write a REST action class for API interaction
- Write Request and Reponse classes for the node that receives the REST action
- Write a Transport action that is dedicated to the process that works on shard level

Source code
-----------
The source code of the example plugin can be found at 

[https://github.com/jprante/elasticsearch-index-termlist](https://github.com/jprante/elasticsearch-index-termlist)

Writing a Plugin class
----------------------

For the example plugin, we choose the name `termlist`. The class extends `AbstractPlugin`. Each plugin may come with modules. Elasticsearch is divided into logical parts which are called modules. For our simple plugin, we need to hook a `RestModule` and an `ActionModule` into the Elasticsearch core. By using injection, the Elasticsearch core invokes the `onModule()` method of our plugin and sets up the necessary structures in the background for us. All we have to do is

	public void onModule(RestModule module) {
        module.addRestAction(RestTermlistAction.class);
    }

    public void onModule(ActionModule module) {
        module.registerAction(TermlistAction.INSTANCE, TransportTermlistAction.class);        
    }

So we have declared three Action classes `RestTermlistAction`, `TermlistAction`, and `TransportTermlistAction` that we need to implement by further steps.

Writing a REST Action class
---------------------------

In the action class `RestTermlistAction`, we assign the REST URI info (path patterns) to executions, and the main request/response method `handleRequest` that is evaluating the query parameters and writing the JSON response.
 
Here, we just need to fetch the values out of the URI info, transfer it to a global request `TermlistRequest`, and invoke a client that will produce a `TermListResponse`. Elasticsearch offers a `RestXContentBuilder` API for easy JSON wrapping of our response data. The term list we want to generate is an array of strings, which is just built by writing the line

    builder.array("terms", response.getTermlist().toArray());

When failures occured, we use Elasticsearch `XContentThrowableRestResponse` to create a failure response. 

Writing a Request and Reponse class
-----------------------------------

The `termlist` plugin should work on the whole cluster or on Elasticsearch indexes. This is done by formulating the request `TermlistRequest` as an extension of a `BroadcastOperationRequest`. This kind of request will be distributed automagically to all the shards that participate in storing the index (or indexes) requested. If no indexes are given in the request, the whole cluster is addressed.

We add a custom parameter `field` beside the index parameter, so that we can expose the IndexReader of Lucene filtered by a given field name. Confining term lists to the terms of a single field is a good idea, for instance if there are a lot of unique terms in the index.

Elasticsearch requires us make the request and the response streamable. Streams are used by the network transport layer to serialize the request to the nodes. All we need to do is writing a few code lines for the `field` parameter in `readFrom(StreamInput in)` and `writeTo(StreamOutput out)`. This mechanism is responsible for both submitting and receving the parameters, from node to node.

The `TermlistResponse` is going the opposite direction after the work has done, from the participating shards to the node that will produce the API response. At instantiation of the class, we know the numbers of total shards, successful shards, and failed shards, and a set of terms, the termlist for the `RestXContentBuilder`. The response must be streamable just like the request class.

Writing the Action class
------------------------
The action class `TermlistAction` is a glue code that clamps together `TermlistRequest`, `TermlistResponse`, and a `TermlistRequestBuilder` into a logical unit. All we need to do is taking care of the generics of the class and filling out the instantiation helpers `newRequestBuilder()` and `newResponse()`. From now on, the Elasticsearch internal client knows how to construct the request and response structures for our action. But up to this point, no shard logic is involved.

Writing the Transport Action class
----------------------------------
The shard level logic is done by `TransportItemlistAction`, a class extending `TransportBroadcastOperationAction`. This action class holds the real execution code of our  `_termlist` command. Here, we can inject an `IndicesServer`, which is the magic Elasticsearch doorway to the Lucene IndexReader internals. In the `shardOperation` method, we can extract an `Engine.Searcher` object for our current shard ID, from which we obtain the Lucene `IndexReader` terms in a loop. 

After collecting the terms into a string set, we send them back in a `ShardTermlistResponse`. `ShardTermlistRequest` and `ShardTermlistResponse` are sibling classes to `TermlistRequest` and `TermlistResponse` on the shard level, extending `BroadcastShardOperationRequest` and `BroadcastShardOperationResponse` respectively.

Besides that, we can control threading and some other shard topics on this level. So it is possible to address only a subset of shards, the primary shards for example, to save resource usage. 

Summary
-------

We have learned how to write a simple Elasticsearch plugin by understanding that such an extension is separated into three actions, one for REST, one for the node level to orchestrate the shards, and one for the shard level to perform the low-level work. Each level is connected by a pair of Request/Response classes. While the Request/Response classes can stream data, the REST action builds the response in memory with a RestXContentBuilder.

By extending the Request classes, we learned how we can also pass additional parameters we need for our operation. By extending the Response classes, we learned how to pass our own result structure back to the REST API surface.

We are now able to

- write Elasticsearch basic plugin code 
- extend the REST API of Elasticsearch by a custom command
- expose Lucene IndexReader to the Elasticsearch API

Of course, there is a lot more that Elasticsearch can offer by using the plugin architecture. In this text, we could only scratch the surface. Have fun exploring Elasticsearch for more exciting opportunities!

