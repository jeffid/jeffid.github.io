
---
title: "Elasticsearch DSL的PHP写法示例"
date: 2020-04-02T18:11:20+08:00
draft: true
---


## 操作环境
下面的程序和运行环境都是使用Laradock部署的
- Elasticsearch 7.6.1
- Kibana 7.6.1
- PHP 7.2
- elasticsearch/elasticsearch ~7.6.0

## 示例
### 导入数据
所有的DSL语句都在Kibana的`Dev Tools`工具中执行.
```shell script
# 导入的数据取材自参考资料二
# 书籍文档信息的集合（有以下字段：title（标题）, authors（作者）, summary（摘要）, publish_date（发布日期）和 num_reviews（浏览数）
POST /bookdb_index/_bulk
{"index":{"_id":1}}
{"title":"Elasticsearch: The Definitive Guide","authors":["clinton gormley","zachary tong"],"summary":"A distibuted real-time search and analytics engine","publish_date":"2015-02-07","num_reviews":20,"publisher":"oreilly"}
{"index":{"_id":2}}
{"title":"Taming Text: How to Find, Organize, and Manipulate It","authors":["grant ingersoll","thomas morton","drew farris"],"summary":"organize text using approaches such as full-text search, proper name recognition, clustering, tagging, information extraction, and summarization","publish_date":"2013-01-24","num_reviews":12,"publisher":"manning"}
{"index":{"_id":3}}
{"title":"Elasticsearch in Action","authors":["radu gheorge","matthew lee hinman","roy russo"],"summary":"build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms","publish_date":"2015-12-03","num_reviews":18,"publisher":"manning"}
{"index":{"_id":4}}
{"title":"Solr in Action","authors":["trey grainger","timothy potter"],"summary":"Comprehensive guide to implementing a scalable search engine using Apache Solr","publish_date":"2014-04-05","num_reviews":23,"publisher":"manning"}
```

### DSL
代码
```shell script
# 下面这些筛选条件完全是为了举例写法才写这么多的...
GET /bookdb_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "elasticsearch guide",
            "fields": []
          }
        },
        {
          "match_phrase_prefix": {
            "title": {
              "query": "elastic",
              "slop": 3,
              "max_expansions": 10
            }
          }
        },
        {
          "wildcard": {
            "title": "elastic*"
          }
        },
        {
          "regexp": {
            "title": "elastic.*ch"
          }
        },
        {
          "range": {
            "num_reviews": {
              "from": 10,
              "to": 20,
              "include_lower": true,
              "include_upper": true
            }
          }
        }
      ],
      "must_not": [
        {
          "terms": {
            "publish_date": [
              "2015-12-04"
            ]
          }
        }
      ],
      "should": [
        {
          "match": {
            "publisher": "manning"
          }
        }
      ],
      "filter": [
        {
          "exists": {
            "field": "authors"
          }
        },
        {
          "range": {
            "num_reviews": {
              "gte": 12,
              "lt": 30
            }
          }
        }
      ]
    }
  },
  "highlight": {
    "fields": {
      "title": {},
      "summary": {}
    }
  },
  "aggs": {
    "sum_of_reviews": {
      "sum": {
        "field": "num_reviews"
      }
    },
    "group_of_date": {
      "terms": {
        "field": "publish_date",
        "size": 10
      },
      "aggs": {
        "avg_of_reviews": {
          "avg": {
            "field": "num_reviews"
          }
        }
      }
    }
  }
}
```

结果
```json
{
  "took" : 17,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 5.7691345,
    "hits" : [
      {
        "_index" : "bookdb_index",
        "_type" : "book",
        "_id" : "1",
        "_score" : 5.7691345,
        "_source" : {
          "title" : "Elasticsearch: The Definitive Guide",
          "authors" : [
            "clinton gormley",
            "zachary tong"
          ],
          "summary" : "A distibuted real-time search and analytics engine",
          "publish_date" : "2015-02-07",
          "num_reviews" : 20,
          "publisher" : "oreilly"
        },
        "highlight" : {
          "title" : [
            "<em>Elasticsearch</em>: The Definitive <em>Guide</em>"
          ]
        }
      },
      {
        "_index" : "bookdb_index",
        "_type" : "book",
        "_id" : "3",
        "_score" : 5.2062206,
        "_source" : {
          "title" : "Elasticsearch in Action",
          "authors" : [
            "radu gheorge",
            "matthew lee hinman",
            "roy russo"
          ],
          "summary" : "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms",
          "publish_date" : "2015-12-03",
          "num_reviews" : 18,
          "publisher" : "manning"
        },
        "highlight" : {
          "summary" : [
            "build scalable search applications using <em>Elasticsearch</em> without having to do complex low-level programming"
          ],
          "title" : [
            "<em>Elasticsearch</em> in Action"
          ]
        }
      }
    ]
  },
  "aggregations" : {
    "sum_of_reviews" : {
      "value" : 38.0
    },
    "group_of_date" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 1423267200000,
          "key_as_string" : "2015-02-07T00:00:00.000Z",
          "doc_count" : 1,
          "avg_of_reviews" : {
            "value" : 20.0
          }
        },
        {
          "key" : 1449100800000,
          "key_as_string" : "2015-12-03T00:00:00.000Z",
          "doc_count" : 1,
          "avg_of_reviews" : {
            "value" : 18.0
          }
        }
      ]
    }
  }
}

```

### PHP
代码
```php
// 要事先安装好elasticsearch/elasticsearch扩展包,注意:扩展包要匹配当前使用的Elasticsearch版本
use \Elasticsearch\ClientBuilder;

$client = ClientBuilder::fromConfig([
    'hosts' => ['elasticsearch:9200'],
    'retries' => 2,
    'handler' => ClientBuilder::singleHandler()
]);

$search = [
    // 指定索引
    'index' => 'bookdb_index',
    // 搜索主体
    'body' => [
        'query' => [
            'bool' => [
                // `must`里的每个条件都要满足
                'must' => [
                    // 多字段条件
                    [
                        'multi_match' => [
                            'query' => 'elasticsearch guide',
                            // 空数组表示搜索任意字段
                            'fields' => []
                            // 也可以指定搜索那些字段
                            // 'fields' => ['title', 'summary']
                        ]
                    ],
                    // 短语前缀条件
                    [
                        'match_phrase_prefix' => [
                            // 搜索`title`字段以`elastic`开头的文档
                            'title' => [
                                'query' => 'elastic',
                                // 调整单词顺序和不太严格的相对位置,大概是查询的分词之间最多能容纳多少个别的词
                                'slop' => 3,
                                // 用来限制查询项的数量，降低对资源需求的强度
                                'max_expansions' => 10
                            ]
                        ]
                    ],
                    // 通配符条件
                    [
                        'wildcard' => [
                            // `title`字段以`elastic`开头
                            'title' => 'elastic*'
                        ]
                    ],
                    // 正则条件
                    [
                        'regexp' => [
                            'title' => 'elastic.*ch'
                        ]
                    ],
                    // 范围条件
                    [
                        'range' => [
                            // 10≤`num_reviews`≤20
                            'num_reviews' => [
                                'from' => 10,
                                'to' => 20,
                                // 包括边界值
                                'include_lower' => true,
                                'include_upper' => true
                            ]
                        ]
                    ]
                ],
                // 必须不在`must_not`的条件范围内
                'must_not' => [
                    [
                        'terms' => [
                            'publish_date' => ['2015-12-04']
                        ]
                    ]
                ],
                // `should`里的条件只要不跟`must`和`must_not`的冲突,那满足`should`的也算(非必须满足)
                'should' => [
                    [
                        'match' => [
                            'publisher' => 'manning'
                        ]
                    ]
                ],
                // 对上面的筛选结果进行补充筛选
                'filter' => [
                    // 要求存在`authors`字段
                    [
                        'exists' => [
                            'field' => 'authors'
                        ]
                    ],
                    // 范围条件的另一种写法
                    [
                        'range' => [
                            'num_reviews' => [
                                'gte' => 12,
                                'lt' => 30
                            ]
                        ]
                    ]
                ]
            ]
        ],
        // 返回相关部分的标记html
        'highlight' => [
            'fields' => [
                // 键是表示要返回哪些字段的相关标记,值是空对象
                'title' => new stdClass(),
                'summary' => new stdClass()
            ]
        ],
        // 聚合
        'aggs' => [
            // 符合全部条件的文件的`num_reviews`字段值的总和,这个总和以`sum_of_reviews`记录下来(这个键名是自定义的)
            'sum_of_reviews' => [
                'sum' => [
                    'field' => 'num_reviews',
                ]
            ],
            // 分组
            'group_of_date' => [
                // 按`publish_date`字段分组
                'terms' => [
                    'field' => 'publish_date',
                    'size' => 10
                ],
                // 计算每个小组的`num_reviews`字段平均分
                'aggs' => [
                    'avg_of_reviews' => [
                        'avg' => [
                            'field' => 'num_reviews'
                        ]
                    ]
                ]
            ]
        ]
    ]
];

echo '<pre>';
var_dump($client->search($search));
        
```

输出
```shell script
array(5) {
  ["took"]=>
  int(18)
  ["timed_out"]=>
  bool(false)
  ["_shards"]=>
  array(4) {
    ["total"]=>
    int(1)
    ["successful"]=>
    int(1)
    ["skipped"]=>
    int(0)
    ["failed"]=>
    int(0)
  }
  ["hits"]=>
  array(3) {
    ["total"]=>
    array(2) {
      ["value"]=>
      int(2)
      ["relation"]=>
      string(2) "eq"
    }
    ["max_score"]=>
    float(5.7691345)
    ["hits"]=>
    array(2) {
      [0]=>
      array(6) {
        ["_index"]=>
        string(12) "bookdb_index"
        ["_type"]=>
        string(4) "book"
        ["_id"]=>
        string(1) "1"
        ["_score"]=>
        float(5.7691345)
        ["_source"]=>
        array(6) {
          ["title"]=>
          string(35) "Elasticsearch: The Definitive Guide"
          ["authors"]=>
          array(2) {
            [0]=>
            string(15) "clinton gormley"
            [1]=>
            string(12) "zachary tong"
          }
          ["summary"]=>
          string(50) "A distibuted real-time search and analytics engine"
          ["publish_date"]=>
          string(10) "2015-02-07"
          ["num_reviews"]=>
          int(20)
          ["publisher"]=>
          string(7) "oreilly"
        }
        ["highlight"]=>
        array(1) {
          ["title"]=>
          array(1) {
            [0]=>
            string(53) "Elasticsearch: The Definitive Guide"
          }
        }
      }
      [1]=>
      array(6) {
        ["_index"]=>
        string(12) "bookdb_index"
        ["_type"]=>
        string(4) "book"
        ["_id"]=>
        string(1) "3"
        ["_score"]=>
        float(5.2062206)
        ["_source"]=>
        array(6) {
          ["title"]=>
          string(23) "Elasticsearch in Action"
          ["authors"]=>
          array(3) {
            [0]=>
            string(12) "radu gheorge"
            [1]=>
            string(18) "matthew lee hinman"
            [2]=>
            string(9) "roy russo"
          }
          ["summary"]=>
          string(152) "build scalable search applications using Elasticsearch without having to do complex low-level programming or understand advanced data science algorithms"
          ["publish_date"]=>
          string(10) "2015-12-03"
          ["num_reviews"]=>
          int(18)
          ["publisher"]=>
          string(7) "manning"
        }
        ["highlight"]=>
        array(2) {
          ["summary"]=>
          array(1) {
            [0]=>
            string(114) "build scalable search applications using Elasticsearch without having to do complex low-level programming"
          }
          ["title"]=>
          array(1) {
            [0]=>
            string(32) "Elasticsearch in Action"
          }
        }
      }
    }
  }
  ["aggregations"]=>
  array(2) {
    ["sum_of_reviews"]=>
    array(1) {
      ["value"]=>
      float(38)
    }
    ["group_of_date"]=>
    array(3) {
      ["doc_count_error_upper_bound"]=>
      int(0)
      ["sum_other_doc_count"]=>
      int(0)
      ["buckets"]=>
      array(2) {
        [0]=>
        array(4) {
          ["key"]=>
          int(1423267200000)
          ["key_as_string"]=>
          string(24) "2015-02-07T00:00:00.000Z"
          ["doc_count"]=>
          int(1)
          ["avg_of_reviews"]=>
          array(1) {
            ["value"]=>
            float(20)
          }
        }
        [1]=>
        array(4) {
          ["key"]=>
          int(1449100800000)
          ["key_as_string"]=>
          string(24) "2015-12-03T00:00:00.000Z"
          ["doc_count"]=>
          int(1)
          ["avg_of_reviews"]=>
          array(1) {
            ["value"]=>
            float(18)
          }
        }
      }
    }
  }
}
```

## 参考资料
下面的资料使用的Elasticsearch版本较低,有部分功能在当前7.x版本已被废弃,需要有选择地参考和使用
- [ElasticSearch学习文档](https://juejin.im/post/5d82e5955188251b764b76ff) 这篇是基础,初学者大概了解下就好,可以当作手册使用
- [23个最有用的Elasticsearch检索技巧](https://juejin.im/post/5b7fe4a46fb9a019d92469a9) 这篇很重要,帮你快速上手DSL
- [《Elasticsearch: 权威指南》](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html) 基于2.x版
- [Elasticsearch-PHP](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/index.html) php的ES扩展包说明文档