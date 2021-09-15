![Elasticsearch](/assets/esc.png)

A beginner-friendly guide to Elasticsearch 6.x covering its architecture, basic operations and query DSL. This is not an ELK stack tutorial but focuses more on the Elasticsearch database itself.

---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Elasticsearch](#elasticsearch)
- [Architecture](#architecture)
- [Getting started](#getting-started)
  - [Creating a new index](#creating-a-new-index)
  - [Adding a document to the index](#adding-a-document-to-the-index)
  - [Fetching a document from the index](#fetching-a-document-from-the-index)
  - [Deleting a document from the index](#deleting-a-document-from-the-index)
- [Mapping](#mapping)
- [Search](#search)
  - [Kibana Discover](#kibana-discover)
  - [Term-Level queries](#term-level-queries)
  - [Full-Text queries](#full-text-queries)
- [Conclusion](#conclusion)
- [Resources](#resources)
- [Acknowledgement](#acknowledgement)
- [Contributing](#contributing)
- [Contact](#contact)
- [License](#license)

<!-- /code_chunk_output -->

---

## Elasticsearch

Elasticsearch is an open-source, RESTful, distributed search and analytics engine built on [Apache Lucene](http://lucene.apache.org/). Since its release in 2010, Elasticsearch has quickly become the most popular search engine, and is commonly used for log analytics, full-text search, security intelligence, business analytics, and operational intelligence use cases.

---

## Architecture

Elasticsearch is built for horizontal scaling, storing data on multiple nodes (servers) at once. An Elasticsearch cluster consists of multiple Elasticsearch nodes, each node storing multiple shards.

![](/assets/cluster.png)

Before we go into shards, it's important to know what an Elasticsearch index is. Coming from SQL world, an index can thought of as a database table with a single schema. The indices in turn, are stored on shards.

For instance, consider a 1.5TB index that is going to be stored on three nodes. Each of the three nodes would store one-third of the index i.e. 500GB on one shard. Overall, we have three nodes (A, B and C) and three shards(1, 2 and 3).\*

![](/assets/index.png)

This leads to the issue that if one of the nodes fail, the entire cluster goes down as we'd be missing one-third of the data. To overcome this, Elasticsearch provides shard-replication. Along with storing one shard on each node, we would also store the replica of the other two shards on the same node. Now, if a single node goes down, we have its data replicated on the other nodes.

In the current example, we have node A storing its primary shard as usual but also storing the replica shards of nodes B and C, making each shard self-sufficient.

![](/assets/shard.png)

As you may observe, maintaining an Elasticsearch cluster can be a job in itself and for the same reason, both Elastic and AWS provided their own cloud-based solutions. If you would rather spin up your own instance, more information can be found [here](https://logz.io/blog/elasticsearch-cluster-tutorial/).

> \*The shard replication strategy can be customized as required. By default, running Elasticsearch on two nodes would provide 5 primary shards and 5 replica shards, 10 shards in total.

## Getting started

A common use case of Elasticsearch is log analysis alongside Logstash and Kibana. However, this guide focuses on the core Elasticsearch database itself and its query language. I am using an AWS managed Elasticsearch instance that ships with Kibana. All the queries here are directly run on Kibana's Dev Tools console. Feel free to use cURL or any other tool of your choice. Detailed installation notes on Elasticsearch and Kibana can be found [here](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elastic-stack-on-ubuntu-18-04).

Once you have set up Elasticsearch and Kibana, below are some queries to get the metadata for your instance.

```
GET _cat/health?v
GET _cat/nodes?v
GET _cat/indices?v
GET _cat/allocation?v
GET _cat/shards?v
```

![](/assets/nodes.png?classes=shadow)

### Creating a new index

Like any other database, it all starts with creating a Table, or an `index` in Elasticsearch.

```
PUT apple
```

The above PUT command creates the index `apple` in the Elasticsearch cluster.

### Adding a document to the index

Once we have set up the index, we can add a document (equivalent to a Row from SQL world) to it.

```
POST apple/default/1
{
  "brand": "Apple",
  "item": "iPhone X",
  "capacity": "64 GB",
  "price": "999.99",
  "description": "iPhone X is a smartphone designed, developed, and marketed by Apple Inc. It was the eleventh generation of the iPhone. It was announced on September 12, 2017, alongside the iPhone 8 and iPhone 8 Plus, at the Steve Jobs Theater in the Apple Park campus. The phone was released on November 3, 2017, marking the iPhone series' tenth anniversary."
}
```

Since Elasticsearch is schema-less, we need not define a prior structure for the document. Here's another example of adding a document to the index.

```
POST apple/default/2
{
  "name": "iPhone XS Max",
  "description": "Apple iPhone XS Max with 128 GB of memory, priced at $1199.99."
}
```

Note that we are manually providing an id to every document, `1` and `2`, in the above examples. However, if we do not provide an id, Elasticsearch will add one on its own.

```
POST apple/default/
{
  "brand": "Apple",
  "item": "Airpods",
  "capacity": null,
  "price": "159.99",
  "description": "Now with more talk time, voice-activated Siri access — and a new wireless charging case — AirPods deliver an unparalleled wireless headphone experience. Simply take them out and they're ready to use with all your devices. Put them in your ears and they connect immediately, immersing you in rich, high-quality sound."
}
```

Adding documents manually one-after-another is far from a real-life situation, we would rather be feeding data into Elasticsearch through an API or a file dump.

Below is an example of dumping a JSON file `apple.json` into the `apple` index.

```bash
$ curl -k -H "Content-Type: application/json" -XPOST --user <username>:<password> "https://<your_elastic_instance_url>:<PORT>/apple/default/_bulk?pretty" --data-binary "@apple.json"
```

We can also use the plugin [Conveyor](https://samtecspg.github.io/conveyor/) for ingesting various types of data and streams.

### Fetching a document from the index

If the `id` of the document is known, then it can be fetched by,

```
GET apple/default/1
```

![](/assets/apple1.png)

While this might not be the case most of the times, we would more likely use `search` to fetch documents.

```
GET apple/default/_search
{
  "query": {
    "match": {
      "item": "airpods"
    }
  }
}
```

![](/assets/airpods.png)

The above query will return all documents that contain `airpods` in the `item` field. More on search later.

### Deleting a document from the index

```
DELETE apple/default/2
```

---

## Mapping

In the previous section, we dumped a JSON file directly into Elasticsearch without providing a context for the fields present in the document. Looking closely, the file `apple.json` consisted of 1,000 objects (or documents), each of them having the following fields.

- brand
- item
- capacity
- price

Below is a sample document from the same file.

```
{
  "brand": "Apple"
  "item": "Airpods",
  "capacity": null,
  "price": "159.99",
  "description": "Now with more talk time, voice-activated Siri access — and a new wireless charging case — AirPods deliver an unparalleled wireless headphone experience. Simply take them out and they're ready to use with all your devices. Put them in your ears and they connect immediately, immersing you in rich, high-quality sound."
}
```

Elasticsearch is smart enough to set the context for each field. In other words, Elasticsearch managed to set the correct datatype for each field e.g. `string` for `item`, `number` for `price` and so on. This is called [Dynamic Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html). We can see the mappings of the `apple` index by,

```
GET apple/default/_mapping
```

The default mapping behavior of Elasticsearch can be overwritten by manually providing the mappings for the fields.

```
PUT apple
{
  "mappings": {
    "default": {
      "dynamic": false,
      "properties": {
        "price": {
          "type": "integer"
        }
      }
    }
  }
}
```

In the above example, we are overwriting the default datatype for the field `price` to `integer`.

---

## Search

Elasticsearch offers two broad kinds of searches, **Term-Level** and **Full-Text**. Before going into either, let's dive into how Elasticsearch _searches_.

The basic concept here is how often a term `t` appears in a single document (term frequency or _tf_) against how many documents actually contain the term `t` (inverse document frequency or _idf_). For instance, consider a document with 100 words and the terms `apple` and `samsung` appear in it 5 and 3 times respectively. Now, assume we have 1,000 such documents, then the term `apple` will have a higher relevance than `samsung`. This is the foundation of the [tf-idf](https://en.wikipedia.org/wiki/Tf%E2%80%93idf) theory. The newer versions of Elasticsearch use a more sophisticated algorithm called [Okapi BM25](https://en.wikipedia.org/wiki/Okapi_BM25).

### Kibana Discover

Kibana has a handy tool called Discover that can be used to query data without writing long and nested JSON objects. All the queries below can be run either on Dev Tools console or inside Discover.

Before we get started, it's required to create an _index pattern_ for Discover to query for an index. The index pattern is mostly same as the original index name and can be created by going to `Kibana` &rarr; `Management` &rarr; `Index Patterns` &rarr; `Create Index Pattern`.

![](/assets/apple_index.png)

Once we have the index pattern in place, we are ready to write some queries.

### Term-Level queries

Term-level queries match for the exact term in the Inverted Index. They should be used for matching an exact expression e.g. date, range, acronym etc. For general purpose searching, use Full-Text search described in the next section.

Below are some examples of term-level queries using the `apple` index defined earlier.

```
GET apple/default/_search
{
  "query": {
    "term": {
      "item": "airpods"
    }
  }
}
```

The same search can be done directly on `Kibana` &rarr; `Discover`.

The query above produces 5 hits (results). In other words, there are five documents in the `apple` index where the `name` field is equal to `airpods`. If we capitalize the query and search for `Airpods` instead, there won't be any hits as term-level queries are case-sensitive.

Searching for multiple terms at once.

```
GET apple/default/_search
{
  "query": {
    "terms": {
      "item": [
        "iphone",
        "airpods"
      ]
    }
  }
}
```

![](/assets/iphone_airpods.png)

Here's an example of a range based query.

```
GET apple/default/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 1000
      }
    }
  }
}
```

A similar range query `price < 200` run on Kibana Discover.

![](/assets/discover_range.png)

Searching with Wildcard character.

```
GET apple/default/_search
{
  "query": {
    "wildcard": {
      "item": "air*"
    }
  }
}
```

![](/assets/wildcard.png)

### Full-Text queries

Full-Text queries are more intuitive and what we'd be using most of the times. Below is the same example where we are searching for `Airpods`, note that we have not capitalized the term but Elasticsearch still produces the results.

```
GET apple/default/_search
{
  "query": {
    "match": {
      "item": "airpods"
    }
  }
}
```

Below is the same query as run from Kibana Discover.

![](/assets/discover_airpods.png)

This is due to the fact that Full-Text queries are _analyzed_ while Term-Level queries are not. It simply means that whenever we conduct a Full-Text search, the query is first analyzed e.g. converted to lowercase, trimmed of white spaces etc. Elasticsearch by default uses the `standard` analyzer. More info on analyzers can be found [here](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html).

We can also search for multiple terms across multiple fields. The query below searches for documents that contain either `Apple` or `Airpods` in either `brand` or `name` fields. A document containing both `Apple` and `Airpods` in the `brand` field will be scored above the document that just contains `Apple`.

```
GET recipes/default/_search
{
  "query": {
    "multi_match": {
      "query": "Apple Airpods",
      "fields": ["brand", "item"]
    }
  }
}
```

![](/assets/multi_match.png)

And that covers the basics of Elasticsearch, there are plenty of other searching parameters/quirks to go through. There is also the whole world of Kibana where you can build one-click visualizations and dashboards. Hope this was a good primer and happy hacking!

## Resources

- [Complete Guide to Elasticsearch](https://www.udemy.com/elasticsearch-complete-guide/) by Bo Anderson
- [Elasticsearch Tutorials](https://www.youtube.com/watch?v=WhXOdGhfE6o&list=PLBtyBPTlyC7sPQDrYOEcQC3d1txIwF299) by Frank Kane
- [Elasticsearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)

## Contributing

Contributions and translations are more than welcome. Feel free to send a PR.

## Contact

[Rahul Sharma](https://rahulxsharma.com)

## License

Distributed under the MIT license. See [LICENSE](/LICENSE) for more information.
