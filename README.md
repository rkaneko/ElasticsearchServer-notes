Elasticsearch Server #6
===

6. Beyond Full-text Searching
+ Aggregations
  +General query structure
  + Available aggregations
    + Metric aggregations
      + Min, max, sum, and avg aggregations
        + Using scripts
      + The value_count aggregation
      + The stats and extended_stats aggregations
    + Bucketing
      + The terms aggregation
      + The range aggregation
      + The date_range aggregation
      + IPv4 range aggregation
      + The missing aggregation
      + Nested aggregation
      + The histogram aggregation
      + The date_histogram aggregation
        + Time zones
      + The geo_distance aggregation
      + The geohash_grid aggregation
  + Nesting aggregations
  + Bucket ordering and nested aggregations
  + Global and subsets
    + Inclusions and exclusions
+ Faceting
  + The document structure
  + Returned results
  + Using queries for faceting calculations
  + Using filters for faceting calculations
  + Terms faceting
  + Ranges based faceting
    + Choosing different fields for an aggregated data calculation
  + Numerical and date histogram faceting
    + The date_histogram facet
  + Computing numerical field statistical data
  + Computing statistical data for terms
  + Geographical faceting
  + Filtering faceting results
  + Memory considerations
+ Using suggesters
  + Available suggester types
  + Including suggestions
    + The suggester response
  + The term suggester
    +The term suggester configuration options
    + Additional term suggester options
  + The phrase suggester
    + Configuration
  + The completion suggester
    + Indexing data
    + Querying the indexed completion suggester data
    + Custom weights
+ Percolator
  + The index
  + Percolator preparation
  + Getting deeper
    + Getting the number of matching queries
    + Indexed documents percolation
+ Handling files
  + Adding additional information about the file
+ Geo
  + Mappings preparation for spatial search
  + Example data
  + Sample queries
    + Distance-based sorting
    + Bounding box filtering
    + Limiting the distance
  + Arbitrary geo shapes
    + Point
    + Envelope
    + Polygon
    + Multipolygon
    + An example usage
    + Storing shapes in the index
+ The scroll API
  + Problem definition
  + Scrolling to the rescue
+ The terms filter
  + Terms lookup
    + The terms lookup query structure
    + Terms lookup cache settings
+ Summary


---


# Aggregations

+ [Aggregation changes @ v2.0](https://www.elastic.co/guide/en/elasticsearch/reference/2.0/breaking_20_aggregation_changes.html)

+ Prerequisites

```bash
# create index
$ curl -XPUT 'http://localhost:9200/example/'
{"acknowledged":true}

# define mappings
# v2.0からroot mappingは不要に?
$ curl -XPUT 'http://localhost:9200/example/book/_mapping' -d @json/mapping-book.json \
--header 'Content-Type: application/json'
{"acknowledged":true}

# see example/book mappings
$ curl -XGET 'http://localhost:9200/example/_mapping/book?pretty'
```

## General query structure

aggregationsのqueryは例えば、次のようになる。

```js
{
  "aggs": {
    "years": {
      "stats": {
        "field": "year"
      }
    },
    "words": {
      "terms": {
        "field": "copies"
      }
    }
  }
}
```

`years`と`words`っていう二つのaggsが定義されている。

### Note

見つかったドキュメント自体が不要なら、`search_type=count`パラメータを使う。こうすることで、必要ない作業が省略されて効率的。

