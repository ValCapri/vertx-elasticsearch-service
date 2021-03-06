# Vert.x ElasticSearch Service

Vert.x 3 elasticsearch service with event bus proxying.

# TODO: Update the documents for vert.x 3 service proxying.

## Configuration

The configuration options are as follows:

```json
{
    "address": <address>,
    "transportAddresses": [ { "hostname": <hostname>, "port": <port> } ],
    "cluster_name": <cluster_name>,
    "client_transport_sniff": <client_transport_sniff>
}
```

* `address` - The event bus address to listen on.  The default is `"et.vertx.elasticsearch"`.
* `transportAddresses` - An array of transport address objects containing `hostname` and `port`.  If no transport address are provided the default is `"localhost"` and `9300`
    * `hostname` - the ip or hostname of the node to connect to.
    * `port` - the port of the node to connect to.  The default is `9300`.
* `cluster_name` - the elastic search cluster name.  The default is `"elasticsearch"`.
* `client_transport_sniff` - the client will sniff the rest of the cluster and add those into its list of machines to use.  The default is `true`.

An example configuration would be:

```json
{
    "address": "eb.elasticsearch",
    "transportAddresses": [ { "hostname": "host1", "port": 9300 }, { "hostname": "host2", "port": 9301 } ],
    "cluster_name": "my_cluster",
    "client_transport_sniff": true
}
```

NOTE: No configuration is needed if running elastic search locally with the default cluster name.


#### Dependency Injection and the HK2VerticleFactory

The `ElasticSearchServiceVerticle` verticle requires a `TransportClientFactory` to be injected.  The default binding provided is for HK2, but you can create your own bindings for your container of choice.

See the [englishtown/vertx-hk2](https://github.com/englishtown/vertx-hk2) project for more details.


## Action Commands

### Index

http://www.elasticsearch.org/guide/reference/api/index_/

Send a json message to the event bus with the following structure:

```json
{
    "action": "index",
    "_index": <index>,
    "_type": <type>,
    "_id": <id>,
    "_source": <source>
}
```

* `index` - the index name.
* `type` - the type name.
* `id` - the string id of the source to insert/update.  This is optional, if missing a new id will be generated by elastic search and returned.
* `source` - the source json document to index

An example message would be:

```json
{
    "action": "index",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_source": {
        "user": "englishtown",
        "message": "love elastic search!"
    }
}
```

The event bus replies with a json message with the following structure:

```json
{
    "status": <status>,
    "_index": <index>,
    "_type": <type>,
    "_id": <id>,
    "_version" <version>
}
```

* `status` - either `ok` or `error`
* `index` - the index where the source document is stored
* `type` - the type of source document
* `id` - the string id of the indexed source document
* `version` - the numeric version of the source document starting at 1.

An example reply message would be:

```json
{
    "status": "ok",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 1
}
```
NOTE: A missing document will always be created (`upsert` mode) because the `op_type` parameter is not implemented yet (http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/docs-index_.html).


### Get

http://www.elasticsearch.org/guide/reference/api/get/

Send a json message to the event bus with the following structure:

```json
{
    "action": "get",
    "_index": <index>,
    "_type": <type>,
    "_id": <id>
}
```

* `index` - the index name.
* `type` - the type name.
* `id` - the string id of the source to get.

An example message would be:

```json
{
    "action": "get",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1"
}
```

The event bus replies with a json message with the following structure:

```json
{
    "status": <status>,
    "_index": <index>,
    "_type": <type>,
    "_id": <id>,
    "_version": <version>
    "_source": <source>
}
```

* `status` - either `ok` or `error`
* `index` - the index name.
* `type` - the type name.
* `id` - the string id of the source to insert/update.  This is optional, if missing a new id will be generated by elastic search and returned.
* `version` - the numeric version of the source document starting at 1.
* `source` - the source json document to index

An example message would be:

```json
{
    "status": "ok",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 1
    "_source": {
        "user": "englishtown",
        "message": "love elastic search!"
    }
}
```


### Search

http://www.elasticsearch.org/guide/reference/api/search/

http://www.elasticsearch.org/guide/reference/query-dsl/

Send a json message to the event bus with the following structure:

```json
{
    "action": "search",
    "_index": <index>,
    "_indices": <indices>,
    "_type": <type>,
    "_types": <types>,
    "query": <query>,
    "postFilter": <postFilter>,
    "facets": <facets>,
    "search_type": <search_type>,
    "scroll": <scroll>,
    "size": <size>,
    "from": <from>,
    "fields": <fields>,
    "timeout": <timeout>
}
```

* `index` - a string index to be searched.  This is optional.
* `indices` - an array of string indices to be searched.  This is optional.
* `type` - a string type to be searched.  This is optional.
* `types` - an array of string types to be searched.  This is optional.
* `query` - a json object, see the elastic search documentation for details (http://www.elasticsearch.org/guide/reference/query-dsl/).  This is optional.
* `postFilter` - a json object, see the elastic search documentation for details (http://www.elasticsearch.org/guide/reference/api/search/postFilter/).  This is optional.
* `facets` - a json object, see the elastic search documentation for details (http://www.elasticsearch.org/guide/reference/api/search/facets/).  This is optional.
* `search_type` - a string to specify the type of search to be performed (http://www.elasticsearch.org/guide/reference/api/search/search-type/).  This is optional.  Possible values include:
    * query_and_fetch - execute the query on all relevant shards
    * query_then_fetch - query is executed against all shards, but documents are not returned.  The results are then sorted and ranked and then only the relevant shards are asked for the documents.
    * dfs_query_and_fetch - same as query_and_fetch, except for an initial scatter phase for more accurate scoring.
    * dfs_query_then_fetch - same as query_then_fetch, except for an initial scatter phase for more accurate scoring.
    * count - returns the count without any documents
    * scan - use this to scroll a large result set
* `scroll` - a string time value parameter (ex. `"5m"` or `"30s"`).  This is only required when search_type is `scan`.
* `size` - a number representing the max number of results to be returned. The default if not specified is 10. (http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-from-size.html). This is optional.
* `from` - a number representing the offset from the first result to be returned. The default if not specified is 0. (http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-from-size.html). This is optional.
* `fields` - an array of strings representing the fields to return. Else the entire document is returned. (http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-fields.html). This is optional.
* `timeout` - a number representing the timeout in milliseconds that should be given to elasticsearch to perform a command. This is optional.

An example message would be:

```json
{
    "action": "search",
    "_index": "twitter",
    "_type": "tweet",
    "query": {
        "match": {
            "user": "englishtown"
        }
    }
}
```

The event bus replies with a json message with a status `"ok"` or `"error"` along with the standard elastic search json search response.  See the documentation for details.

An example reply message for the query above would be:

```json
{
    "status": "ok",
    "took" : 3,
    "timed_out" : false,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    },
    "hits" : {
        "total" : 2,
        "max_score" : 0.19178301,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "1",
                "_score" : 0.19178301,
                "_source" : {
                    "user": "englishtown",
                    "message" : "love elastic search!"
                }
            },
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "2",
                "_score" : 0.19178301,
                "_source" : {
                    "user": "englishtown",
                    "message" : "still searching away"
                }
            }
        ]
    }
}
```


### Scroll

http://www.elasticsearch.org/guide/reference/api/search/scroll/

First send a search message with `search_type` = `"scan"` and `scroll` = `"5m"` (some time string).  The search result will include a `_scroll_id` that will be valid for the scroll time specified.


Send a json message to the event bus with the following structure:

```json
{
    "action": "scroll",
    "_scroll_id": <_scroll_id>,
    "scroll": <scroll>
}
```

* `_scroll_id` - the string scroll id returned from the scan search.
* `scroll` - a string time value parameter (ex. `"5m"` or `"30s"`).


An example message would be:

```json
{
    "action": "scroll",
    "_scroll_id": "c2Nhbjs1OzIxMTpyUkpzWnBIYVMzbVB0VGlaNHdjcWpnOzIxNTpyUkpzWnBI",
    "scroll": "5m"
}
```

The event bus replies with a json message with a status `"ok"` or `"error"` along with the standard elastic search json scroll response.  See the documentation for details.

An example reply message for the scroll above would be:

```json
{
    "status": "ok",
    "_scroll_id": "c2Nhbjs1OzIxMTpyUkpzWnBIYVMzbVB0VGlaNHdjcWpnOzIxNTpyUkpzWnBI",
    "took": 2,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits" : {
        "total" : 2,
        "max_score" : 0.0,
        "hits" : [
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "1",
                "_score" : 0.0,
                "_source" : {
                    "user": "englishtown",
                    "message" : "love elastic search!"
                }
            },
            {
                "_index" : "twitter",
                "_type" : "tweet",
                "_id" : "2",
                "_score" : 0.0,
                "_source" : {
                    "user": "englishtown",
                    "message" : "still searching away"
                }
            }
        ]
    }
}
```

### Delete

http://www.elasticsearch.org/guide/reference/api/delete/

Send a json message to the event bus with the following structure:

```json
{
    "action": "delete",
    "_index": <index>,
    "_type": <type>,
    "_id": <id>
}
```

* `index` - the index name.
* `type` - the type name.
* `id` - the string id of the document to delete.

An example message would be:

```json
{
    "action": "delete",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1"
}
```

The event bus replies with a json message with the following structure:

```json
{
    "found": <status>,
    "_index": <index>,
    "_type": <type>,
    "_id": <id>,
    "_version": <version>
}
```

* `found` - either `true` or `false`
* `index` - the index name.
* `type` - the type name.
* `id` - the string id of the source to delete.
* `version` - the numeric version of the deleted document starting at 1.

An example message would be:

```json
{
    "found": "true",
    "_index": "twitter",
    "_type": "tweet",
    "_id": "1",
    "_version": 1
}
```
