# Elasticsearch中文同义词



Elasticsearch的标准版本及以上是支持设置同义词功能的, 其实也就是除了OSS(开源)版以外其它的都支持.



 ## 环境说明

- Elasticsearch 7.6.x

- 与ES相匹配的IK分词插件

- 示例中会分别使用到shell命令和Kibana, 以`$`开头的代表是shell命令, 否则表示Kibana的console命令

  
## 操作

> 同义词可以使用 `synonym` 参数来内嵌指定，或者必须 存在于集群每一个节点上的同义词文件中。 同义词文件路径由 `synonyms_path` 参数指定，应绝对或相对于 Elasticsearch `config` 目录。

下面以同义词的两种设置方式来介绍:



### 同义词文件方式

#### 设置同义词文件

```shell
# 进入Elasticsearch目录执行,生成文件
$ echo '"iPhone,苹果手机 => iPhone,苹果手机",
    "2233,22娘,33娘 => bilibili,B站"' > config/analysis/synonyms.txt
```

#### 创建索引

```shell
PUT /goods2
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "updateable": true,
          "synonyms_path": "analysis/synonyms.txt"
        }
      },
      "analyzer": {
        "my_synonyms_analyzer": {
          "tokenizer": "ik_smart",
          "filter": [
            "my_synonym_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_smart",
        "search_analyzer": "my_synonyms_analyzer"
      }
    }
  }
}
```

> `my_synonym_filter`是自定义的`词汇过滤器`, `my_synonyms_analyzer`是自定义的分析器, 可以看出后者是包含并引用了前者的.
>
> 在本索引中自定义的词汇过虑器和分析器也只能在当前索引中使用.
>
> `updateable`指示**能否动态更新**, 必须为`true`才能动态更新同义词
>
> `synonyms_path`指示同义词文件的位置
>
> `analysis.analyzer.tokenizer`指示在这个分析器里用`ik_smart`的分词器, 在这个索引中的分析链是`原始文本 => 分词器 => 词汇过滤器`, 即原始文本先经过分词的结果再用来给`词汇过滤器`处理(在这个索引的作用是同义词).
>
> `mappings.properties.title.search_analyzer`指示`title`字段在查询时使用`my_synonyms_analyzer`分析器, 同理`mappings.properties.title.analyzer`指示其在索引时使用的分析器.



#### 查看分析结果

##### 第一行分词的效果

```shell
# 字母大小写没有影响
GET goods2/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "iphone"
}

GET goods2/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "苹果手机"
}
```

上面两条语句的结果是一样的

```json
{
  "tokens" : [
    {
      "token" : "iphone",
      "start_offset" : 0,
      "end_offset" : 6,
      "type" : "ENGLISH",
      "position" : 0
    },
    {
      "token" : "苹果",
      "start_offset" : 0,
      "end_offset" : 6,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "手机",
      "start_offset" : 0,
      "end_offset" : 6,
      "type" : "SYNONYM",
      "position" : 1
    }
  ]
}

```

##### 第二行分词的效果

```
GET goods2/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "2233"
}

GET goods2/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "22娘"
}
```

结果

```json
{
  "tokens" : [
    {
      "token" : "bilibili",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "b",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "站",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 1
    }
  ]
}

```



#### 变更同义词并更新索引

```shell
# 进入Elasticsearch目录执行,生成文件
# `iPhone,苹果手机 => iPhone,苹果手机`与`iPhone,苹果手机`的效果是一样的
# 内容中的双引号`"`和行末的逗号`,`不是必须的(没有的话须要有换行符), 这里只是为了和和内嵌式的保持一致才这么写的
$ echo '"iPhone,苹果手机",
    "2233,22娘,33娘 => bilibili,B站,二次元"' > config/analysis/synonyms.txt
```

```shell
# 使新的同义词生效
POST /goods2/_reload_search_analyzers
```



##### 变更同义词后的第二行分词的效果

```
GET goods2/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "2233"
}
```

结果

```json
{
  "tokens" : [
    {
      "token" : "bilibili",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "b",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "二次元",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 0
    },
    {
      "token" : "站",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "SYNONYM",
      "position" : 1
    }
  ]
}

```



### 内嵌方式

#### 创建索引

同义词配置就在`synonyms`属性里

```shell
PUT /goods3
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "iPhone,苹果手机 => iPhone,苹果手机",
            "2233,22娘,33娘 => bilibili,B站"
          ]
        }
      },
      "analyzer": {
        "my_synonyms_analyzer": {
          "tokenizer": "ik_smart",
          "filter": [
            "my_synonym_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_smart",
        "search_analyzer": "my_synonyms_analyzer"
      }
    }
  }
}
```



#### 查看分析结果

下面的结果跟同义词文件方式的是一样的

```shell

GET goods3/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "iphone"
}

GET goods3/_analyze
{
  "analyzer": "my_synonyms_analyzer",
  "text": "2233"
}
```



#### 变更同义词并更新索引

```shell
# 须要先关闭索引才能变更设置
POST /goods3/_close

PUT /goods3/_settings/
{
  "analysis": {
    "filter": {
      "my_synonym_filter": {
        "type": "synonym",
        "synonyms": [
          "iPhone,苹果手机",
          "2233,22娘,33娘 => bilibili,B站,二次元"
        ]
      }
    }
  }
}

# 重新开启索引
POST /goods3/_open
```



### 查询实践

以索引`goods2`为例

```shell
# 插入一条数据
POST /goods2/_doc/1
{
  "title":"bilibili是个好平台"
}

# 通过`2233`关键词查找
GET /goods2/_search
{
  "query": {
    "match": {
      "title": "2233"
    }
  }
}
```

结果

```json
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "goods2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "title" : "bilibili是个好平台"
        }
      }
    ]
  }
}
```



## 总结

- 在Elasticsearch中设置同义词有`内嵌式`和`同义词文件式`两种

- `同义词文件式`可以在不关闭索引的情况下动态更新同义词

  

## 参考资料

- [Elasticsearch Reference 7.6 重载分析器API ](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/indices-reload-analyzers.html#indices-reload-analyzers-api-example) 
- [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/cn/index.html) 
- [Elastic Stack 各版本的区别](https://www.elastic.co/cn/subscriptions)


