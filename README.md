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

+ [Breaking changes in 2.0 - Aggregation changes](https://www.elastic.co/guide/en/elasticsearch/reference/2.0/breaking_20_aggregation_changes.html)
+ [Facets have been replaced by aggregations in Elasticsearch 1.0, which are a superset of facets](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-facets.html)
+ [Migrating from facets to aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/2.0/search-facets-migrating-to-aggs.html)

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
$ curl -XPUT 'http://localhost:9200/example/book/_mapping' -d @json/mapping-book.json
{"acknowledged":true}

# see example/book mappings
$ curl -XGET 'http://localhost:9200/example/_mapping/book?pretty'

# index(upsert) sample data
$ curl -XPOST 'http://localhost:9200/_bulk' --data-binary @json/example-book-data.json

# see all example/book documents
$ curl -XGET 'http://localhost:9200/example/book/_search?q=*:*'

# (delete mapping API は 2.0 から廃止に https://www.elastic.co/guide/en/elasticsearch/reference/2.0/indices-delete-mapping.html )
# 代わりにindexごと消して再作成しないといけない?
$ curl -XDELETE 'http://localhost:9200/example/'
```

## General query structur
aglregationsのqueryは例えば、次のようになる。

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

上で定義したaggregationsを実行すると次のようになる。

```bash
$ curl -XGET 'http://localhost:9200/_search?search_type=count&pretty' -d @json/first-aggs.json

{
  "took" : 5,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "words" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 6,
      "buckets" : [ {
        "key" : "and",
        "doc_count" : 5
      }, {
        "key" : "harry",
        "doc_count" : 5
      }, {
        "key" : "potter",
        "doc_count" : 5
      }, {
        "key" : "the",
        "doc_count" : 5
      }, {
        "key" : "of",
        "doc_count" : 4
      }, {
        "key" : "fire",
        "doc_count" : 2
      }, {
        "key" : "goblet",
        "doc_count" : 2
      }, {
        "key" : "part",
        "doc_count" : 2
      }, {
        "key" : "1",
        "doc_count" : 1
      }, {
        "key" : "2",
        "doc_count" : 1
      } ]
    },
    "years" : {
      "count" : 5,
      "min" : 1997.0,
      "max" : 2000.0,
      "avg" : 1998.8,
      "sum" : 9994.0
    }
  }
}
```

`words`では、**buckets**と呼ばれるdocumentsから計算された集合が結果に返される。

aggrerationsでは複数のaggregation typesを利用できる。

### Note

見つかったドキュメント自体が不要なら、`search_type=count`パラメータを使う。こうすることで、必要ない作業が省略されて効率的。

# Available aggregations

+ metric aggregations
+ bucketing aggregations

## Metric aggregations
### Min, max, sum, and avg aggregations

numeric fieldに対して、min等の統計量を計算することができる。

```js
{
  "aggs": {
    "min_year": {
      "min": {
        "field": "year"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' -d @json/min-year-aggs.json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "min_year" : {
      "value" : 1997.0
    }
  }
}
```

#### Using scripts

inputにscriptを使うことができる。Dynamic scriptingはdefaultで無効化されている。

+ build-in ([default scripting language: groovy](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting.html#_default_scripting_language))
  + groovy, expression, mustache
+ plugin
  + javascript, python
+ [Native (Java) Scripts](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting.html#native-java-scripts)

+ config/elasticsearch.ymlで有効・無効設定

```yml
script.engine.groovy.inline.aggs: on
```

+ [Clustering Update Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-update-settings.html)で動的には設定変更はできないみたい。

groovy script使ってみる。

```js
{
  "aggs": {
    "min_year": {
      "min": {
        /* field 指定ではなく,　field値を演算する */
        "script": "doc['year'].value - 1000"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
-d @json/min-year-aggs-using-scripts.json
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "min_year" : {
      "value" : 997.0
    }
  }
}
```

次のように、別プロパティにscriptやparamsを分けることもできる。

```js
{
  "aggs": {
    "min_year": {
      "min": {
        "field": "year",
        "script": "_value - mod",
        "params": {
          "mod": 1000
        }
      }
    }
  }
}
```

### The value_count aggregation

value_count aggregationsでは集計したいfieldはnumericである必要はない。

```js
{
  "aggs": {
    "number_of_items": {
      "value_count": {
        "field": "characters"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
> -d @json/number-of-items.json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "number_of_items" : {
      "value" : 30
    }
  }
}
```

ここで、Elasticsearch aggregationsがどのようにカウントされる値を探しているかを考察する。
Elasticsearchはすべてのdocumentsをまたいで characters　field からすべてのtokenをカウントする。

サンプル・データでは、各documentに、

`Harry`,`Potter`,`Ron`,`Weasley`,`Hermione`,`Granger`

の６つのtokenが存在する。｀Harry Potter`は解析後、`harry`,`potter`のようになる。

このよう挙動を目的としない場合は、not-analyzed versionの characters field を使う。

### The stats and extended_stats aggregations

stats, extended_stats aggregationsは一つのaggregation objectで複数の計算された特徴量を扱える。

```js
{
  "aggs": {
    "stats_year": {
      "stats": {
        "field": "year"
      }
    }
  }
}
```

結果はfirst-aggsと同じなので略

stats aggregationsよりももっと統計的な情報が欲しいときは、extended_stats aggregationsを使う。

```js
{
  "aggs": {
    "stats_year": {
      "extended_stats": {
        "field": "year"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
-d @json/extended-stats-year.json
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "stats_year" : {
      "count" : 5,
      "min" : 1997.0,
      "max" : 2000.0,
      "avg" : 1998.8,
      "sum" : 9994.0,
      "sum_of_squares" : 1.9976014E7,
      "variance" : 1.3600000001490116,
      "std_deviation" : 1.1661903790329482,
      "std_deviation_bounds" : {
        "upper" : 2001.132380758066,
        "lower" : 1996.467619241934
      }
    }
  }
}
```

+ `sum_of_squares`: 平方和
+ `variance`: 分散
+ `std_deviation`: 標準偏差

## Bucketing

bucketing aggregationsは複数のsubsetを返し、とbucketと呼ばれる特定のsubsetの評価する。

### The terms aggregation

terms aggregationはfield内で有効なtermに対する一つのbucketを返す。
field値の出現頻度といった統計量を求めることができる。

次のQuestionsはterms aggregationで答えることができる。

0. それぞれの年ごとに何冊の本が出版されたか？
0. 借りることができる本は何冊？
0. 所有している本でもっとも多い本は何冊か？

3つ目のQuestionに答えるために次のaggregationを実行。

```js
{
  "aggs": {
    "availability": {
      "terms": {
        "field": "copies"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
-d @json/terms-copies.json
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "availability" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [ {
        "key" : 2,
        "doc_count" : 2
      }, {
        "key" : 3,
        "doc_count" : 1
      }, {
        "key" : 4,
        "doc_count" : 1
      }, {
        "key" : 5,
        "doc_count" : 1
      } ]
    }
  }
}
```

defaultではElasticsearchでは`doc_count`の降順にsortingされた結果が返される。

`order`プロパティを指定することで、`key`等でsort順を指定できる。

以下は、keyで降順ソートした例。

```js
{
  "aggs": {
    "availability": {
      "terms": {
        "field": "copies",
        /* size で最大でどれだけのbucketを返せばよいかを指定できる */
        "size": 40,
        /* _term を指定するとkey propertiesを使ってsorting指定できる */
        "order": { "_term": "desc" }
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \-d @json/terms-copies-with-order.json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 5,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "availability" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [ {
        "key" : 5,
        "doc_count" : 1
      }, {
        "key" : 4,
        "doc_count" : 1
      }, {
        "key" : 3,
        "doc_count" : 1
      }, {
        "key" : 2,
        "doc_count" : 2
      } ]
    }
  }
}
```

#### Note

fieldが解析されるとき、analyed termsからbucketsが得られる。これが望ましくない場合は、field に解析に使いたい not-analyzed versionを追加すること。

### The range aggregation

range aggregationでは、定義された範囲でbucketsが作成される。

ex)与えられた期間に何冊の本が出版されたか?

```js
{
  "aggs": {
    "years": {
      "range": {
        "field": "year",
        "ranges": [
          { "to": 2000 },
          { "from": 1998, "to": 1999 },
          { "from": 1998, "to": 2000 },
          { "from": 1997 }
        ]
      }
    }
  }
}
```

出版年度のデータは以下のようになっているとき,

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?q=*:*&pretty' | jq '.hits.hits[]._source' | jq '.title, .year'
"Harry Potter and the Goblet of Fire - Part 2"
2000
"Harry Potter and the Chamber of Secrets"
1998
"Harry Potter and the Goblet of Fire - Part 1"
2000
"Harry Potter and the Philosopher's Stone"
1997
"Harry Potter and the Prisoner of Azkaban"
1999
```

```bash
$ curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
-d @json/published-between-given-period.json | jq '.aggregations'

{
  "years": {
    "buckets": [
      {
        "doc_count": 3,
        "to_as_string": "2000.0",
        "to": 2000,
        "key": "*-2000.0"
      },
      {
        "doc_count": 5,
        "from_as_string": "1997.0",
        "from": 1997,
        "key": "1997.0-*"
      },
      {
        "doc_count": 1,
        "to_as_string": "1999.0",
        "to": 1999,
        "from_as_string": "1998.0",
        "from": 1998,
        "key": "1998.0-1999.0"
      },
      {
        "doc_count": 2,
        "to_as_string": "2000.0",
        "to": 2000,
        "from_as_string": "1998.0",
        "from": 1998,
        "key": "1998.0-2000.0"
      }
    ]
  }
} 
```

bucketに名前をつけることもできる。

```js
{
  "aggs": {
    "years": {
      "range": {
        "field": "year",
        "ranges": [
          /* key で bucket に名前を付ける */
          { "key": "Before 20th century", "to": 2000 },
          { "key": "from 1998 to 1999", "from": 1998, "to": 1999 },
          { "key": "from 1988 to 2000", "from": 1998, "to": 2000 },
          { "key": "After year 1997", "from": 1997 }
        ]
      }
    }
  }
}
```

```bash
curl -XGET 'http://localhost:9200/example/book/_search?search_type=count&pretty' \
-d @json/published-between-given-period-with-bucket-name.json | jq '.aggregations'

{
  "years": {
    "buckets": [
      {
        "doc_count": 3,
        "to_as_string": "2000.0",
        "to": 2000,
        "key": "Before 20th century"
      },
      {
        "doc_count": 5,
        "from_as_string": "1997.0",
        "from": 1997,
        "key": "After year 1997"
      },
      {
        "doc_count": 1,
        "to_as_string": "1999.0",
        "to": 1999,
        "from_as_string": "1998.0",
        "from": 1998,
        "key": "from 1998 to 1999"
      },
      {
        "doc_count": 2,
        "to_as_string": "2000.0",
        "to": 2000,
        "from_as_string": "1998.0",
        "from": 1998,
        "key": "from 1988 to 2000"
      }
    ]
  }
}
```

#### Note

rangesはdisjoint(互いに素)である必要がないところが便利。

### The date_range aggregation

date types field のためにデザインされたrange aggregation。

サンプルデータでは、`year`はnumberでdateではない。

次のようなnewspaperを想定したテストデータを用意する。

```bash
# create index
$ curl -XPUT 'http://localhost:9200/library2/'
# define mapping
$ curl -XPUT 'http://localhost:9200/library2/book/_mapping' -d @json/mapping-newspaper.json

# upsert documents
$ curl -XPOST 'http://localhost:9200/_bulk' --data-binary @json/library2-newspaper-data.json

# see all documents
$ curl -XGET 'http://localhost:9200/library2/book/_search?q=*:*&pretty' | jq '.hits.hits[]._source' | jq '.title, .published'

"Kitting magazine"
"2010-11-07T11:32:00"
"Hadoop World"
"2012-01-01T04:00:00"
"Fishing news"
"2010-12-03T10:00:00"
"The guardian"
```

date_range aggregationのフォーマットは次のようになる。

```js
{
  "aggs": {
    "years": {
      "date_range": {
        "field": "published",
        "ranges": [
          { "to": "2009-12-31" },
          { "from": "2010-01-01", "to": "2010-12-31" },
          { "from": "2011-01-01" }
        ]
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library2/book/_search?search_type=count&pretty' \
-d @json/date-range-published.json | jq '.aggregations'

{ 
  "years": {
    "buckets": [
      {
        "doc_count": 1,
        "to_as_string": "2009-12-31T00:00:00.000Z",
        "to": 1262217600000,
        "key": "*-2009-12-31T00:00:00.000Z"
      },
      {
        "doc_count": 2,
        "to_as_string": "2010-12-31T00:00:00.000Z",
        "to": 1293753600000,
        "from_as_string": "2010-01-01T00:00:00.000Z",
        "from": 1.262304e+12,
        "key": "2010-01-01T00:00:00.000Z-2010-12-31T00:00:00.000Z"
      },
      {
        "doc_count": 1,
        "from_as_string": "2011-01-01T00:00:00.000Z",
        "from": 1.29384e+12,
        "key": "2011-01-01T00:00:00.000Z-*"
      }
    ]
  }
}
```

+ `from`,`to` がmillisecondsの数値となっている。
+ `from_as_string`,`to_as_string` がhuman readable? な時刻表現になっている。 (mapping時に定義した[date format](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html)による)
+ range aggretationと同様に`key`プロパティの定義もできる。

+ date format指定もできる。(※ date typeのmappingに指定ししているならば、指定したformatと互換性がある必要がある)
  - `_timestamp` fieldが廃止され、date format　が厳格化

```js
{
  "aggs": {
    "years": {
      "date_range": {
        "field": "published",
        /* date format指定 */
        "format": "strict_date_optional_time",
        "ranges": [
          { "to": "2009-12-31" },
          { "from": "2010-01-01", "to": "2010-12-31" },
          { "from": "2011-01-01" }
        ]
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library2/book/_search?search_type=count&pretty' \
-d @json/date-range-published-with-format.json | jq '.aggregations'

{
  "years": {
    "buckets": [
      {
        "doc_count": 1,
        "to_as_string": "2009-12-31T00:00:00.000Z",
        "to": 1262217600000,
        "key": "*-2009-12-31T00:00:00.000Z"
      },
      {
        "doc_count": 2,
        "to_as_string": "2010-12-31T00:00:00.000Z",
        "to": 1293753600000,
        "from_as_string": "2010-01-01T00:00:00.000Z",
        "from": 1.262304e+12,
        "key": "2010-01-01T00:00:00.000Z-2010-12-31T00:00:00.000Z"
      },
      {
        "doc_count": 1,
        "from_as_string": "2011-01-01T00:00:00.000Z",
        "from": 1.29384e+12,
        "key": "2011-01-01T00:00:00.000Z-*"
      }
    ]
  }
}
```

### IPv4 range aggregation

Internet address に関するaggregation。Elasticsearchでは2.0時点では、IPv6 typeはサポートされていない。
https://www.elastic.co/guide/en/elasticsearch/reference/2.0/ip.html

ip typeでは、CIDR表記によるIP rangeを定義できる。

ip_range aggregationは次のようになる。

```js
{
  "aggs": {
    "access": {
      "ip_range": {
        "field": "ip",
        "ranges": [
          { "from": "192.168.0.1", "to": "192.168.0.254" },
          { "mask": "192.168.1.0/24" }
        ]
      }
    }
  }
}
```

responseは次のようになる。

```js
{
  "aggs": {
    "access": {
      "ip_range": {
        "field": "ip",
        "ranges": [
          { "from": "192.168.0.1", "to": "192.168.0.254" },
          { "mask": "192.168.1.0/24" }
        ]
      }
    }
  }
}
```

### The missing aggregation

あるfieldデータが定義されていないdocumentがいくつあるか確認したい場合、missing aggregationを用いる。

```bash
# create an index
$ curl -XPUT 'http://localhost:9200/library/'
# define mapping
$ curl -XPUT 'http://localhost:9200/library/book/_mapping' -d @json/mapping-book.json
# upsert example data
$ curl -XPOST 'http://localhost:9200/_bulk' --data-binary @json/library-book-data.json

# see data
$ curl -XGET 'http://localhost:9200/library/book/_search?q=*:*&pretty' | jq '.hits.hits[]._source'
```

「炎のゴブレット」×2だけ日本語タイトル(title field)が空とする。
これを確認するaggregationは次のようになる。

```js
{
  "aggs": {
    "missing_title": {
      "missing": {
        "field": "title"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/missing-title-aggs.json | jq '.aggregations'

{
  "missing_title": {
    "doc_count": 2
  }
}
```

#### Note

missing aggregationからわかることは、mapping definitionに `null_value`が定義されていて、｀null_value`は独立して数える必要があるということ。

### Nested aggregation

Chapter4でnested objectについて学んだ。ココでは、nested aggregationについて学ぶ。

例えば次のようなaggregationが考えられる。

```js
{
  "aggs": {
    "variations": {
      "nested": {
        "path": "variation"
      }
    }
  }
}
```

次のようなレスポンスが得られたら、`variation` typeの与えられたindexに、2つのnestしたdocumentがあるということ。

```js
{
  "variations": {
    "doc_count": 2
  }
}
```

### The histogram aggregation

histogram aggregationはbucketsを定義するaggregationの一つ。
次のようになる。

```js
{
  "aggs": {
    "years": {
      "histogram": {
        "field": "year",
        /* bucketsをつくるために使うrangeを定義 */
        "interval": 100
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/histogram-year-aggs.json | jq '.aggregations'
{
  "years": {
    "buckets": [
      {
        "doc_count": 3,
        "key": 1900
      },
      {
        "doc_count": 2,
        "key": 2000
      }
    ]
  }
}
```

+ `keyd` property: key propertyを指定。
+ `min_doc_count` property: bucketsを構成する最小のdocument数。`0`を指定すると、bucketsを構成するdocumentの数が0のものも含んだレスポンスを返す。

### The date_histogram aggregation

ranga aggregationのspecial form。date typeのためのhistogram aggragation。

```js
{
  "aggs": {
    "years": {
      "date_histogram": {
        "field": "published",
        "format": "strict_date_optional_time",
        "interval": "100d"
      }
    }
  }
}
```

+ `format` propertyは必須ではない。
+ `keyed`, `min_doc_count` propertyも使える。

100日間隔でdocumentをカウントする。

```bash
$ curl -XGET 'http://localhost:9200/library2/book/_search?search_type=count&pretty' \
-d @json/date-histogram-year.json | jq '.aggregations'
{
  "years": {
    "buckets": [
      {
        "doc_count": 1,
        "key": 1.24416e+12,
        "key_as_string": "2009-06-05T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.2528e+12,
        "key_as_string": "2009-09-13T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.26144e+12,
        "key_as_string": "2009-12-22T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.27008e+12,
        "key_as_string": "2010-04-01T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.27872e+12,
        "key_as_string": "2010-07-10T00:00:00.000Z"
      },
      {
        "doc_count": 2,
        "key": 1.28736e+12,
        "key_as_string": "2010-10-18T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.296e+12,
        "key_as_string": "2011-01-26T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.30464e+12,
        "key_as_string": "2011-05-06T00:00:00.000Z"
      },
      {
        "doc_count": 0,
        "key": 1.31328e+12,
        "key_as_string": "2011-08-14T00:00:00.000Z"
      },
      {
        "doc_count": 1,
        "key": 1.32192e+12,
        "key_as_string": "2011-11-22T00:00:00.000Z"
      }
    ]
  }
}
```

#### Time zones

ElasticsearchではUTC time zone ですべてのdateが保存される。

表示用にtime zone を定義できる。Elasticsearchではコンバート方法が2つある。

+ elementに代入前にdateから適切なbucketに変換
+ elementに代入後に変換

振る舞いを決める2つの属性値がある。

+ `pre_zone`
+ `post_zone`

これに加えて、基本的に`pre_zone`をセットする`time_zone`という属性値もある。

これらを使った3つの記法がある。

+ the hours offset ex)`pre_zone: -4` or `time_zone: 5`
+ the time format ex)`pre_zone: "-4:30"`
+ name of time zone ex)`time_zone: "Europe/Warsaw"`

#### Note

[available time zone: joda-time#timezones](http://joda-time.sourceforge.net/timezones/html)

### The geo_distance aggregation

略

### The geohash_grid aggregation

略

# Nesting aggregations

複雑なqueryを書くときに強力。
nested aggregationは親aggregationの結果をさらに解析する。

サンプル・データの準備

```bash
# create an index example
$ curl -XPUT 'http://localhost:9200/example/
# define mapping product type
$ curl -XPUT 'http://localhost:9200/example/product/_mapping' \
-d @json/mapping-product.json
# upsert example data
$ curl -XPOST 'http://localhost:9200/_bulk' --data-binary @json/nested-product-data.json
# see data
$ curl -XGET 'http://localhost:9200/example/product/_search?q=*:*&pretty' | jq '.hits.hits[]._source'
{
  "resellers": [
    {
      "price": 150,
      "name": "Bob"
    },
    {
      "price": 200,
      "name": "Ken"
    },
    {
      "price": 300,
      "name": "Becky"
    }
  ],
  "product_name": "Vegetable A",
  "id": 2
}
{
  "resellers": [
    {
      "price": 10000,
      "name": "Bob"
    },
    {
      "price": 9000,
      "name": "Ken"
    }
  ],
  "product_name": "Toy A",
  "id": 1
}
```

上のデータから、各documentのresellersのsizeの和を求めるqueryは次のようになる。

```js
{
  "aggs": {
    "resellers": {
      "nested": {
        "path": "resellers"
      },
      "aggs": {
        "sizes": {
          "terms": {
            "field": "resellers.size"
          }
        }
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/product/_search?search_type=count&pretty' \
-d @json/nested-resellers-with-size-aggs.json | jq '.aggregations'

{
  "resellers": {
    "sizes": {
      "buckets": [],
      "sum_other_doc_count": 0,
      "doc_count_error_upper_bound": 0
    },
    "doc_count": 5
  }
}
```

すべてのdocumentから`resellers`プロパティの要素の`price`プロパティの最小値を求めるqueryは次のようになる。

```js
{
  "aggs": {
    "resellers": {
      "nested": {
        "path": "resellers"
      },
      "aggs": {
        "min_price": { "min": { "field": "resellers.price" } }
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/example/product/_search?search_type=count&pretty' \
-d @json/nested-resellers-with-min-price-aggs.json | jq '.aggregations'

{
  "resellers": {
    "min_price": {
      "value": 150
    },
    "doc_count": 5
  }
}
```

年range別統計量を集計する。

```js
{
  "aggs": {
    "years": {
      "range": {
        "field": "year",
        "ranges": [
          { "from": 1997, "to": 1999 },
          { "from": 1999, "to": 2001 }
        ]
      },
      "aggs": {
        "statistics": {
          "stats": {}
        }
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/nested-book-range-year-stats-aggs.json | jq '.aggregations.years.buckets[]'

{
  "statistics": {
    "sum": 3995,
    "avg": 1997.5,
    "max": 1998,
    "min": 1997,
    "count": 2
  },
  "doc_count": 2,
  "to_as_string": "1999.0",
  "to": 1999,
  "from_as_string": "1997.0",
  "from": 1997,
  "key": "1997.0-1999.0"
}
{
  "statistics": {
    "sum": 5999,
    "avg": 1999.6666666666667,
    "max": 2000,
    "min": 1999,
    "count": 3
  },
  "doc_count": 3,
  "to_as_string": "2001.0",
  "to": 2001,
  "from_as_string": "1999.0",
  "from": 1999,
  "key": "1999.0-2001.0"
}
```

特定のContext(親aggregation)の集計に便利！


# Bucket ordering and nested aggregations

bucketのkeysやdocument countでソートできるか？

→　できる。nested aggregationの値も使える。

`book#copies`のカウントをとって、統計情報をとって、子aggregationの統計量：平均で降順ソート。

```js
{
  "aggs": {
    "availability": {
      "terms":  {
        "field": "copies",
        "order": { "numbers.avg": "desc" }
      },
      "aggs": {
        "numbers": { "stats": {} }
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/ordering-book-copies.json | jq '.aggregations.availability.buckets[]'

{
  "numbers": {
    "sum": 5,
    "avg": 5,
    "max": 5,
    "min": 5,
    "count": 1
  },
  "doc_count": 1,
  "key": 5
}
{
  "numbers": {
    "sum": 4,
    "avg": 4,
    "max": 4,
    "min": 4,
    "count": 1
  },
  "doc_count": 1,
  "key": 4
}
{
  "numbers": {
    "sum": 3,
    "avg": 3,
    "max": 3,
    "min": 3,
    "count": 1
  },
  "doc_count": 1,
  "key": 3
}
{
  "numbers": {
    "sum": 4,
    "avg": 2,
    "max": 2,
    "min": 2,
    "count": 2
  },
  "doc_count": 2,
  "key": 2
}
```


# Global and subsets

aggregation frameworkはすべてのindexから、queryでfilterしたデータを集計したり(subsets)、、反対にqueryを無視した集計(global)をすることできる。

`library#book#copies`の値あるなしをcountする。subsetsとglobalの違いに注目。

まず、データを確認。

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?pretty' | jq '.hits.hits[]._source' | jq '.otitle, .copies'
"Harry Potter and the Goblet of Fire - Part 2"
2
"Harry Potter and the Chamber of Secrets"
4
"Harry Potter and the Goblet of Fire - Part 1"
2
"Harry Potter and the Philosopher's Stone"
5
"Harry Potter and the Prisoner of Azkaban"
3
```

subsetsとglobalの違いをみるqueryは以下の通り。

```js
{
  "query": {
    /* 2.0.0-beta1でfiltered clauseはdeprecatedになっている */
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "available": "true"
        }
      }
    }
  },
  "aggs": {
    /* global: query節は無視 */
    "with_global": {
      "global": {},
      "aggs": {
        "copies": {
          "value_count": {
            "field": "copies"
          }
        }
      }
    },
    /* subsets: query節でfilterされたdocumentsから集計 */
    "without_global": {
      "value_count": {
        "field": "copies"
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/global-subsets-filter-aggs.json | jq '.aggregations'

{
  "with_global": {
    "copies": {
      "value": 5
    },
    "doc_count": 5
  },
  "without_global": {
    "value": 3
  }
}
```

globalを使わなくても、 queryは表現できる。

```js
{
  /* globalを使用しなくても,queryは書けるがnestが深くなる */
  "aggs": {
    "with_filter": {
      "filter": {
        "term": { "available": "true" }
      },
      "aggs": {
        "copies": {
          "value_count": {
            "field": "copies"
          }
        }
      }
    },
    "without_filter": {
      "value_count": {
        "field": "copies"
      }
    }
  }
}
```

#### Note

+ [Filtered Query](https://www.elastic.co/guide/en/elasticsearch/reference/2.0/query-dsl-filtered-query.html)
+ [The query/filter merge](https://www.elastic.co/blog/better-query-execution-coming-elasticsearch-2-0)

## Inclusions and exclusions

terms aggregationには集計の数をより小さくできるオプションがある。

→ include/exclude feature (string valueにてきようできる。)

```js
{
  "aggs": {
    "availability": {
      "terms": {
        "field": "characters",
        /* java.util.regex.Pattern の正規表現で指定 */
        "include": "h.*", /* hで始まるものをtermsの対象に(hermione, harry) */
        "exclude": "ha.*" /* haで始まるものはtermsの対象外に(harryは除かれる) */
      }
    }
  }
}
```

```bash
$ curl -XGET 'http://localhost:9200/library/book/_search?search_type=count&pretty' \
-d @json/exclude-include.json | jq '.aggregations'
{
  "availability": {
    "buckets": [
      {
        "doc_count": 5,
        "key": "hermione"
      }
    ],
    "sum_other_doc_count": 0,
    "doc_count_error_upper_bound": 0
  }
}
```

# Faceting



## The document structure
## Returned results
## Using queries for faceting calculations
## Using filters for faceting calculations
## Terms faceting
## Ranges based faceting
### Choosing different fields for an aggregated data calculation
## Numerical and date histogram faceting
### The date_histogram facet
## Computing numerical field statistical data
## Computing statistical data for terms
## Geographical faceting
## Filtering faceting results
## Memory considerations
# Using suggesters
## Available suggester types
## Including suggestions
### The suggester response
## The term suggester
### The term suggester configuration options
### Additional term suggester options
## The phrase suggester
### Configuration
## The completion suggester
### Indexing data
### Querying the indexed completion suggester data
### Custom weights
# Percolator
## The index
## Percolator preparation
## Getting deeper
### Getting the number of matching queries
### Indexed documents percolation
# Handling files
## Adding additional information about the file
# Geo
## Mappings preparation for spatial search
## Example data
## Sample queries
### Distance-based sorting
### Bounding box filtering
### Limiting the distance
## Arbitrary geo shapes
### Point
### Envelope
### Polygon
### Multipolygon
### An example usage
### Storing shapes in the index
# The scroll API
## Problem definition
## Scrolling to the rescue
# The terms filter
##Terms lookup
### The terms lookup query structure
### Terms lookup cache settings
# Summary

