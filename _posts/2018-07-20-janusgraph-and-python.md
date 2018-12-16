---
title:  "JanusGraph & Python"
date:   2018-07-20 00:00:00
---

JanusGraph is a scalable [graph database](https://en.wikipedia.org/wiki/Graph_database) optimized for storing and querying graphs containing hundreds of billions of vertices and edges distributed across a multi-machine cluster.

The easiest way to connect to JanusGraph is through the [Gremlin console](http://tinkerpop.apache.org/docs/current/tutorials/the-gremlin-console/). It is a great tool to experiment but what if you want to develop an application in Python that will execute queries against JanusGraph in support of the application front-end ?

![stack]({{site.url}}/assets/images/janusgraph-and-python/stack.png)

Python is not a JVM-based language, hence we are not able to embed JanusGraph calls directly from a Python program. Instead, JanusGraph packages a long running server process that, when started, allows a remote client or logic running in a separate program to make JanusGraph calls. This long running server process is called [JanusGraph Server](https://docs.janusgraph.org/0.2.0/server.html). Moreover TinkerPop3 provides [gremlin-python](https://github.com/apache/tinkerpop/tree/master/gremlin-python), a Gremlin language variant that allows developer to express Gremlin graph traversal natively and send requests to the server.

> JanusGraph is just a set of jar files with no thread of executions. The primary pattern for using JanusGraph is by embedding JanusGraph calls from within a client program providing its own thread of execution. This is what is happening when you start a Gremlin console. This could also be done from any JVM-based language client programs like Java, Scala (see gremlin-scala) or Clojure (see ogre). This is a little bit confusing at first because most of the databases we know are packaged as a server (PostgreSQL, MySQL, MongoDB, Elasticsearch, etc).

## JanusGraph server

JanusGraph uses [Gremlin Server](http://tinkerpop.apache.org/docs/3.2.6/reference/#gremlin-server) of the [Apache TinkerPop](https://medium.com/@BGuigal/janusgraph-python-9e8d6988c36c) stack to service client requests. You can start a JanusGraph server using janusgraph.sh script:

```
Usage: bin/janusgraph.sh [options] {start|stop|status|clean}
```

> The `janusgraph.sh` script will fork Cassandra and Elasticsearch for you before calling `bin/gremlin-server.sh conf/gremlin-server.yaml`. If you need a different storage and index backend you can adapt this script to fit your needs.

The server is configured by a YAML file `conf/gremlin-server/gremlin-server.yaml`. The file tells the Gremlin server many things such as

* the host and port to serve on
* the script engines enabled
* the serializers to make available
* the different `Graph` instances to expose

<br/>
<code data-gist-id="48645113c8dafc24a78fe8b953b909f7"></code>


> A JanusGraph instance maintains a set of vertices and edges, as well as access to the underlying pluggable storage. In the configuration file above, the line graph: `conf/gremlin/server/janusgraph-cassandra-es-server-properties` tells the Gremlin server which storage backend to use.

## Connecting via the gremlin console

Once the server is running, you can try it out in the Gremlin console.

```
./bin/gremlin.sh

gremlin > :remote connect tinkerpop.server conf/remote.yaml session     ==> Configured localhost/127.0.01:8181

gremlin > :remote console
==> All scripts will now be sent to Gremlin Server

gremlin > graph
==> standardjanusgraph[cassandrathrift:[127.0.0.1]]
```

The graph instance defined in the configuration file is injected as global variables in the console ! Note that we could have defined several graphs with different configurations and they would all have been made available.

The script `scripts/empty-sample.groovy` (also defined in the configuration file) defines a default traversal source as `g`.

```
gremlin > g
==>graphtraversalsource[standardjanusgraph[cassandrathrift:[127.0.0.1]], standard]
```

Let’s load the graph of the gods data into our graph instance and perform some traversals.

```
gremlin > GraphOfTheGodsFactory.load(graph)
==>null

gremlin> g.V().count()
==>12

gremlin> saturn = g.V().has('name', 'saturn').next()
==>v[4240]

gremlin> g.V(saturn).in('father').in('father').values('name')
==>hercules
```

## Connecting with Python

![gremlin-python]({{site.url}}/assets/images/janusgraph-and-python/gremlin-python.png)

In order to connect to the server from Python we need to configure the gremlin-server for Python and to install the `gremlin-python` package.

```
# This will download a set of jar files in ./ext/gremlin-python
./bin/gremlin-server.sh -i org.apache.tinkerpop gremlin-python 3.2.6

# Install gremlin-python from PyPi
pip install gremlinpython==3.2.6
```

The version of `gremlin-python` should match the TinkerPop version which is compatible with your version of JanusGraph. See the [releases page](https://github.com/JanusGraph/janusgraph/releases) to find about your version of JanusGraph.

Let’s connect to the Gremlin server from Python using the [Gremlin Python driver](http://tinkerpop.apache.org/docs/3.2.6/reference/#connecting-via-python):


<code data-gist-id="d3860f32d5d1a5537d09e880f741ebbb"></code>

Using the Python driver is okay but we can do better: it is possible to express Gremlin traversal in plain Python thanks to [Gremlin Python language variant](http://tinkerpop.apache.org/docs/3.2.6/reference/#gremlin-variants).

> Gremlin is a graph traversal language that makes use of two fundamental programming constructs: [function composition](https://en.wikipedia.org/wiki/Function_composition) and [function nesting](https://en.wikipedia.org/wiki/Nested_function). Given this generality, it is possible to embed Gremlin in any modern programming language. It elevates Gremlin to a top-level citizen in the language of choice.

<code data-gist-id="6899ba58210bbd3e7b63e9d3ebb305c9"></code>

There are some slight language variations compared to using Gremlin in the Gremlin console.

* Traversal should be explicitly terminated by calling an action method among `next()`, `nextTraverser()` , `toList()`, `toSet()`, `iterate()`.
* `as`, `in`, `and`, `or`, `is`, `not`, `from`, `global` are reserved keywords in Python and you should a postfix notation. For instance: `g.V().as('a').in_().as_('b').select('a','b')`.
* You will need to explicitly import static enums into the scope to use anonymous traversal like `out()`.

<br/>
How does this magic happen ? Under the hood, a traversal in native Python is translated to Gremlin `Bytecode`, sent over the network and ultimately compiled to a `Traversal` by JanusGraph

![language-variant]({{site.url}}/assets/images/janusgraph-and-python/language-variant.png)

For details about Gremlin language variants, please follow the [tutorial](http://tinkerpop.apache.org/docs/current/tutorials/gremlin-language-variants/) in the TinkerPop documentation.

## JanusGraph specific features

JanusGraph implements a lot of useful features that are not part of the TinkerPop specs. For example you can use an index like ElasticSearch and use text predicate to make a full text fuzzy search against a property of graph. For example

```
john = g.V().has('name', textContainsFuzzy('John Doe'))
```

Unfortunately, you won’t be able perform this traversal with the Python Gremlin variant because it is Janus specific. You will need to construct a string representation of the query and to submit it to the Python Gremlin driver.

## Useful links

* [http://tinkerpop.apache.org/](http://tinkerpop.apache.org/)
* [https://www.slideshare.net/JasonPlurad/start-flying-with-python-apache-tinkerpop](https://www.slideshare.net/JasonPlurad/start-flying-with-python-apache-tinkerpop)
* [https://www.datastax.com/dev/blog/the-benefits-of-the-gremlin-graph-traversal-machine](https://www.datastax.com/dev/blog/the-benefits-of-the-gremlin-graph-traversal-machine)
* [https://stackoverflow.com/questions/46070265/gremlin-python-returning-empty-graph](https://www.datastax.com/dev/blog/the-benefits-of-the-gremlin-graph-traversal-machine)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-the-titan-graph-database-with-cassandra-and-elasticsearch-on-ubuntu-16-04](https://www.datastax.com/dev/blog/the-benefits-of-the-gremlin-graph-traversal-machine)