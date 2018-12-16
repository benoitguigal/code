---
title:  "Elasticsearch: mapping and analyzers"
date:   2018-08-08 00:00:00
---

*Elasticsearch is a highly scalable open-source full-text search and analytics engine. It allows you to store, search, and analyze big volumes of data quickly and in near real time. It is generally used as the underlying engine/technology that powers applications that have complex search features and requirements.*

![logo]({{site.url}}/assets/images/elasticsearch/logo.png)

Elasticsearch is reputed to be *schema-less*, which means you can quickly insert documents to an index without specifying a schema beforehand. This differs from traditional relational databases where you have to explicitly specify tables, fields and fields types.

Under the hood though, Elasticsearch enforces a schema called *mapping* which describes the fields in the documents along with their data types and how they should be indexed by Lucene.

If you do not specify an explicit mapping, Elasticsearch will generate one for you based on the first document you insert. The mapping will also be automatically updated as new fields are added to the documents. Let’s see how it works.

### Dynamic mapping

Let’s index some user profiles in Elasticsearch. Each user profile has a `name`, a `date_of_birth` and a `description`. The `date_of_birth` is formatted as a timestamp in milliseconds.

<code data-gist-id="06eb0cac50ecaab29e1ac72e24f51539"></code>


Let’s verify that our document has effectively been created:

<code data-gist-id="01d1a4bad3b7a96ea8fd55074e7e2127"></code>

Elasticsearch has automatically inferred the type of each field based on a set of [simple rules](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html).

### Explicit mapping

Dynamic mapping is great but it can leads to unexpected search results.

Let’s suppose we want to search for all users born in the 80's. Our user `Bobby Kennedy` is born at epoch `553941680000` which corresponds to `1987–07–22`. It should pop up in the results. We can issue a [date range query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) on the `date_of_birth` field:

<code data-gist-id="991db4cf19b394ddb14cfa9b81a1bd88"></code>

No hits ??

If you had thoroughly looked at the mapping, you shouldn’t be surprised. Our `date_of_birth` field has a `long` type because we did not tell Elasticsearch it should store it as a `date`. This is where explicit mapping come into play. Let’s delete our index, set a mapping with explicit `date` type for `date_of_birth` , and reindex our document.

> You cannot change the mapping of an index when documents are already present.

<code data-gist-id="8a74c7e3bf0169d4bdc803720484020e"></code>

Note that you can set the mapping only partially. Mappings for other fields will be dynamic.

Let’s reissue our range query:

<code data-gist-id="f1710d8277d0d5f8dc8a1ae178e48f01"></code>

Hourah it worked !

> You know more about your data than Elasticsearch does

![standard-analyzer]({{site.url}}/assets/images/elasticsearch/standard-analyzer.png)

We can inspect how the analyser works thanks to the `_analyse` API.

<code data-gist-id="4a08c89b7747a7270e1a2502b304e3a8"></code>

Let’s verify a search for the word “*passionate*” matches :


<code data-gist-id="12a16d22b96bbbe81a916e56f342fbf2"></code>

This is great but we would like Elasticsearch to be tolerant to the different inflected forms of a word. In our example I would like my document to match if I search for the word passion. This can be achieved with a [language analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html) that will stem the tokens. This process is called [lemmatization](https://en.wikipedia.org/wiki/Lemmatisation) and will group together all inflected forms of a word. Ex: passionate → passion, foxes → fox, jumped → jump, etc.

![english-analyzer]({{site.url}}/assets/images/elasticsearch/english-analyzer.png)

Let’s remove the index, apply the *english analyzer* to our description field, and reindex our document.

<code data-gist-id="14b0cd50bc59b8b2a88cb5610d08ed0c"></code>

Let’s verify a search for the word *passion* matches:

<code data-gist-id="fa36caef9fb54d1723b64b36d9b0c573"></code>

The analysis is very flexible, you can even define your [own analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-custom-analyzer.html). There are also plenty of [analysis plugins](https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis.html) both official or supported by the community. A very interesting one is the phonetic analyser that analyses tokens into their phonetic equivalent using Soundex, Metaphone and other codecs.
