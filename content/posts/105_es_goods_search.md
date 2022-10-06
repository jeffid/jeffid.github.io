
---
title: "电商商品信息Elasticsearch索引结构与查询示例"
date: 2020-04-15T23:39:00+08:00
draft: false
tags: ["Elasticsearch", "PHP"]
series: ["ELK"]
categories: ["后端"]
---


## 说明

本文是学习[Learnku](https://learnku.com/laravel)的 [电商课程](https://learnku.com/courses/ecommerce-advance/6.x/the-basic-concept-of-elasticsearch/5861) Elasticsearch(下面将称之为:ES)部分的学习笔记,主要介绍常见的电商商品数据如何存入ES和查询出来.ES作为搜索引擎,相比数据库的SQL搜索语句可以实现更多丰富的筛选条件.

常见的使用方法是:先按用户的搜索条件从ES中查询出关键信息(如id),然后直接列表返回给用户,或是根据ES结果作为SQL条件再从数据库中查询.

本文仅以DSL作为查询示例,不会涉及在各个编程语言中的实现代码.

### 操作环境

- Elasticsearch 7.6.x

- 与ES相匹配的IK分词插件

  

## 商品信息数据结构

下面的SQL是创建商品相关表和插入数据的语句, 本文主要着重介绍的是在这种数据结构下ES的索引结构和查询写法,所以这些SQL不实际执行也不会影响到后面说到的DSL操作的.

```sql
# 商品信息主表
CREATE TABLE `products` (
 `id` bigint unsigned NOT NULL AUTO_INCREMENT,
 `type` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL DEFAULT 'normal',
 `category_id` bigint unsigned DEFAULT NULL,
 `title` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `long_title` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `description` text COLLATE utf8mb4_unicode_ci NOT NULL,
 `image` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `on_sale` tinyint(1) NOT NULL DEFAULT '1',
 `rating` double(8,2) NOT NULL DEFAULT '5.00',
 `sold_count` int unsigned NOT NULL DEFAULT '0',
 `review_count` int unsigned NOT NULL DEFAULT '0',
 `price` decimal(10,2) NOT NULL,
 `created_at` timestamp NULL DEFAULT NULL,
 `updated_at` timestamp NULL DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `products_category_id_foreign` (`category_id`),
 KEY `products_type_index` (`type`)
) ENGINE=InnoDB AUTO_INCREMENT=100 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

# 端口的sku表,每个库存单位占一条,如:同款手机的不同版本就各算作一个库存单位,对应的价格可能也不一样的,主要用来确定货单价和库存的
# 与主表关联
CREATE TABLE `product_skus` (
 `id` bigint unsigned NOT NULL AUTO_INCREMENT,
 `title` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `description` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `price` decimal(10,2) NOT NULL,
 `stock` int unsigned NOT NULL,
 `product_id` bigint unsigned NOT NULL,
 `created_at` timestamp NULL DEFAULT NULL,
 `updated_at` timestamp NULL DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `product_skus_product_id_foreign` (`product_id`),
 CONSTRAINT `product_skus_product_id_foreign` FOREIGN KEY (`product_id`) REFERENCES `products` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=295 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

# 商品的详细属性,每个属性占一条,不同商品之间允许存在同名的属性,如:A手机的`内存`属性值为`8G`,主要用来展示属性和筛选商品的
# 与主表关联
CREATE TABLE `product_properties` (
 `id` bigint unsigned NOT NULL AUTO_INCREMENT,
 `product_id` bigint unsigned NOT NULL,
 `name` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 `value` varchar(255) COLLATE utf8mb4_unicode_ci NOT NULL,
 PRIMARY KEY (`id`),
 KEY `product_properties_product_id_foreign` (`product_id`),
 CONSTRAINT `product_properties_product_id_foreign` FOREIGN KEY (`product_id`) REFERENCES `products` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB AUTO_INCREMENT=27 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci

```

对应的数据记录
```sql

INSERT INTO `products` (`id`, `type`, `category_id`, `title`, `long_title`, `description`, `image`, `on_sale`, `rating`, `sold_count`, `review_count`, `price`, `created_at`, `updated_at`) VALUES
(91, 'normal', 13, 'Kingston/金士顿 HX424C15FB/8', '金士顿 骇客神条 ddr4 2400 8g 台式机 电脑 四代内存条 吃鸡内存', '<p><img src=\"https://img.alicdn.com/imgextra/i3/704392951/TB25akyqsuYBuNkSmRyXXcA3pXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i1/704392951/TB288x6y25TBuNjSspmXXaDRVXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i1/704392951/TB2ck46y7CWBuNjy0FaXXXUlXXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2_OV3y1uSBuNjSsziXXbq8pXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i3/704392951/TB2F9KZiP7nBKNjSZLeXXbxCFXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i4/704392951/TB2XQ06y7CWBuNjy0FaXXXUlXXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB20Tl7y4SYBuNjSspjXXX73VXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2QygAqDdYBeNkSmLyXXXfnVXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i3/704392951/TB2C6S5qyCYBuNkHFCcXXcHtVXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2J_pByYGYBuNjy0FoXXciBFXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2520Ny29TBuNjy1zbXXXpepXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i4/704392951/TB2ozkLyFmWBuNjSspdXXbugXXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i4/704392951/TB2S9IFiOAnBKNjSZFvXXaTKXXa_!!704392951.jpg\" /></p><p><img alt=\"\" src=\"https://gdp.alicdn.com/imgextra/i4/704392951/TB2KpHwfviSBuNkSnhJXXbDcpXa_!!704392951.jpg\" /></p>', 'https://img.alicdn.com/bao/uploaded/i2/TB1iqkaLVXXXXagXXXXObG1FpXX_091208.jpg_b.jpg', 1, 5.00, 0, 0, '399.00', '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(92, 'normal', 13, 'AData/威刚 8G DDR4 2400 (XPG 单条） ', 'ADATA/威刚 8G 16G 3200 3000 2666 2400游戏台式机内存条DDR4套条', '<p><img src=\"https://img.alicdn.com/imgextra/i4/2133729733/TB2LYbVxFOWBuNjy0FiXXXFxVXa_!!2133729733.jpg\" /><br /><a href = \"https://detail.tmall.com/item.htm?spm=a1z10.5-b-s.w4011-16853183550.96.20b86fd1MBVKRL&id=40645526570&rn=d717312a898e0fb53e74b1c2db2c2232&abbucket=12\" target = \"_self\" ><img src = \"https://img.alicdn.com/imgextra/i2/2133729733/TB2zEdobrZnBKNjSZFhXXc.oXXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i1/2133729733/TB2W3VPbWmWBuNjy1XaXXXCbXXa_!!2133729733.jpg\" /></a ><br /><img src = \"https://img.alicdn.com/imgextra/i1/2133729733/TB2NLaeaQyWBuNjy0FpXXassXXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i4/2133729733/TB2hvRtfamgSKJjSsphXXcy1VXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i2/2133729733/TB2DFptaXXXXXaOXXXXXXXXXXXX_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i4/2133729733/TB2mAUhkCFjpuFjSszhXXaBuVXa_!!2133729733.jpg_q90.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i1/2133729733/TB2aU8kaXXXXXbbXpXXXXXXXXXX_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i3/2133729733/TB2Nhf8cRfM8KJjSZFrXXXSdXXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i1/2133729733/TB2h0oEhSYH8KJjSspdXXcRgVXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i2/2133729733/TB202q8gP3z9KJjy0FmXXXiwXXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i3/2133729733/TB2kRllh0nJ8KJjSszdXXaxuFXa_!!2133729733.jpg\" /><img src = \"https://img.alicdn.com/imgextra/i3/2133729733/TB2BXY3cqzB9uJjSZFMXXXq4XXa_!!2133729733.jpg\" /></p >', 'https://img.alicdn.com/bao/uploaded/i4/TB1URYGHVXXXXXsaXXXtD198VXX_032444.jpg_b.jpg', 1, 5.00, 0, 0, '489.00', '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(93, 'normal', 13, 'Kingston/金士顿 金士顿DDR3 1600 8GB', 'Kingston/金士顿 DDR3 1600 8G 台式机电脑 三代 内存条 兼容1333', '<p><img src=\"https://img.alicdn.com/imgextra/i4/704392951/TB2Y5OKqOOYBuNjSsD4XXbSkFXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i1/704392951/TB2cQS8y29TBuNjy0FcXXbeiFXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i4/704392951/TB2GrWfqIyYBuNkSnfoXXcWgVXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2.Onyy7yWBuNjy0FpXXassXXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i3/704392951/TB2yEnCy29TBuNjy1zbXXXpepXa_!!704392951.jpg\" /><img src=\"https://img.alicdn.com/imgextra/i2/704392951/TB2Urm1y7KWBuNjy1zjXXcOypXa_!!704392951.jpg\" /></p><p><img alt = \"\" src = \"https://gdp.alicdn.com/imgextra/i4/704392951/TB2KpHwfviSBuNkSnhJXXbDcpXa_!!704392951.jpg\" /></p >', 'https://img.alicdn.com/bao/uploaded/i5/TB1up8DGXXXXXaAaXXXszso_pXX_060025.jpg_b.jpg', 1, 5.00, 0, 0, '239.00', '2020-03-23 16:57:28', '2020-03-23 16:57:28');


INSERT INTO `product_skus` (`id`, `title`, `description`, `price`, `stock`, `product_id`, `created_at`, `updated_at`) VALUES
(271, '8GB 黑色', '8GB 2400 DDR4 黑色', '549.00', 999, 91, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(272, '8GB 绿色', '8GB 2400 DDR4 绿色', '529.00', 999, 91, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(273, '16GB', '2400 16GB', '1299.00', 999, 91, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(274, '4GB', '2400 4GB', '399.00', 999, 91, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(275, '8GB DDR4 2400', '8GB DDR4 2400 XPG单条', '489.00', 999, 92, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(276, '4GB 万紫千红 DDR4 2133', '4GB 万紫千红 DDR4 2133', '489.00', 999, 92, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(277, 'DDR3 1600 8G', 'DDR3 1600 8G', '439.00', 999, 93, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(278, 'DDR3 1600 4G', 'DDR3 1600 4G', '239.00', 999, 93, '2020-03-23 16:57:28', '2020-03-23 16:57:28'),
(279, 'DDR3 1333 4G', 'DDR3 1333 4G', '259.00', 999, 93, '2020-03-23 16:57:28', '2020-03-23 16:57:28');


INSERT INTO `product_properties` (`id`, `product_id`, `name`, `value`) VALUES
(1, 91, '品牌名称', '金士顿'),
(2, 91, '内存容量', '8GB'),
(3, 91, '传输类型', 'DDR4'),
(4, 91, '内存容量', '4GB'),
(5, 91, '内存容量', '16GB'),
(6, 92, '品牌名称', '威刚'),
(7, 92, '传输类型', 'DDR4'),
(8, 92, '内存容量', '4GB'),
(9, 92, '内存容量', '8GB'),
(10, 93, '品牌名称', '金士顿'),
(11, 93, '传输类型', 'DDR3'),
(12, 93, '内存容量', '4GB'),
(13, 93, '内存容量', '8GB');
```



## ES操作

下面是重点

### 存入

#### 创建索引

```shell
# 创建索引`products`,`pretty`参数表示返回的结果以美化的格式输出
curl -XPUT http://localhost:9200/products?pretty

# 设置索引的属性,`skus`后的一个嵌套属性`properties`并不是关键字,是自定义的名称,指代`product_properties`表的数据
curl -XPUT http://localhost:9200/products/_mapping/?pretty -H 'Content-Type: application/json' -d '{
  "properties": {
    "type": { "type": "keyword" } ,
    "title": { "type": "text", "analyzer": "ik_smart" }, 
    "long_title": { "type": "text", "analyzer": "ik_smart" }, 
    "category_id": { "type": "integer" },
    "category": { "type": "keyword" },
    "category_path": { "type": "keyword" },
    "description": { "type": "text", "analyzer": "ik_smart" },
    "price": { "type": "scaled_float", "scaling_factor": 100 },
    "on_sale": { "type": "boolean" },
    "rating": { "type": "float" },
    "sold_count": { "type": "integer" },
    "review_count": { "type": "integer" },
    "skus": {
      "type": "nested",
      "properties": {
        "title": { "type": "text", "analyzer": "ik_smart", "copy_to": "skus_title" }, 
        "description": { "type": "text", "analyzer": "ik_smart", "copy_to": "skus_description" },
        "price": { "type": "scaled_float", "scaling_factor": 100 }
      }
    },
    "properties": {
      "type": "nested",
      "properties": {
        "name": { "type": "keyword" }, 
        "value": { "type": "keyword", "copy_to": "properties_value" }, 
        "search_value": { "type": "keyword"}
      }
    }
  }
}'
```


> `"analyzer": "ik_smart"` 代表这个字段需要使用 IK 中文分词器分词，在前面的章节也介绍过了。
>
> 还有有一些字段的类型是 `keyword`，这是字符串类型的一种，这种类型是告诉 Elasticsearch 不需要对这个字段做分词，通常用于邮箱、标签、属性等字段。
>
> `scaled_float` 代表一个小数位固定的浮点型字段，与 Mysql 的 `decimal` 类型类似，后面的 `scaling_factor` 用来指定小数位精度，100 就代表精确到小数点后两位。
>
> `skus` 和 `properties` 的字段类型是 `nested`，代表这个字段是一个复杂对象，由下一级的 `properties` 字段定义这个对象的字段。有同学可能会问，我们的『商品 SKU』和『商品属性』明明是对象数组，为什么这里可以定义成对象？这是 Elasticsearch 的另外一个特性，每个字段都可以保存多个值，这也是 Elasticsearch 的类型没有数组的原因，因为不需要，每个字段都可以是数组。

> 注意看 `skus.title` 字段的定义里加入了 `copy_to` 参数，值是 `skus_title`，Elasticsearch 就会把这个字段值复制到 `skus_title` 字段里，这样就可以在 `multi_match` 的 `fields` 里通过 `skus_title` 来匹配。`skus.description` 和 `properties.name` 同理。


>  请确保 Elasticsearch 返回了 `"acknowledged" : true`，否则就要检查提交的内容是否有问题。

#### 导入数据到ES

shell命令连接ES按照索引的数据结构导入3条数据

category

```shell
curl -XPUT http://localhost:9200/products/_doc/91?pretty -H'Content-Type: application/json' -d'{"id":91,"type":"normal","category_id":13,"title":"Kingston\/金士顿 HX424C15FB\/8","long_title":"金士顿 骇客神条 ddr4 2400 8g 台式机 电脑 四代内存条 吃鸡内存","on_sale":true,"rating":5,"sold_count":0,"review_count":0,"price":"399.00","category":["电脑配件","内存"],"category_path":"-10-","description":"","skus":[{"title":"8GB 黑色","description":"8GB 2400 DDR4 黑色","price":"549.00"},{"title":"8GB 绿色","description":"8GB 2400 DDR4 绿色","price":"529.00"},{"title":"16GB","description":"2400 16GB","price":"1299.00"},{"title":"4GB","description":"2400 4GB","price":"399.00"}],"properties":[{"name":"品牌名称","value":"金士顿","search_value":"品牌名称:金士顿"},{"name":"内存容量","value":"8GB","search_value":"内存容量:8GB"},{"name":"传输类型","value":"DDR4","search_value":"传输类型:DDR4"},{"name":"内存容量","value":"4GB","search_value":"内存容量:4GB"},{"name":"内存容量","value":"16GB","search_value":"内存容量:16GB"}]}'

curl -XPUT http://localhost:9200/products/_doc/92?pretty -H'Content-Type: application/json' -d'{"id":92,"type":"normal","category_id":13,"title":"AData\/威刚 8G DDR4 2400 (XPG 单条） ","long_title":"ADATA\/威刚 8G 16G 3200 3000 2666 2400游戏台式机内存条DDR4套条","on_sale":true,"rating":5,"sold_count":0,"review_count":0,"price":"489.00","category":["电脑配件","内存"],"category_path":"-10-","description":"","skus":[{"title":"8GB DDR4 2400","description":"8GB DDR4 2400 XPG单条","price":"489.00"},{"title":"4GB 万紫千红 DDR4 2133","description":"4GB 万紫千红 DDR4 2133","price":"489.00"}],"properties":[{"name":"品牌名称","value":"威刚","search_value":"品牌名称:威刚"},{"name":"传输类型","value":"DDR4","search_value":"传输类型:DDR4"},{"name":"内存容量","value":"4GB","search_value":"内存容量:4GB"},{"name":"内存容量","value":"8GB","search_value":"内存容量:8GB"}]}'

curl -XPUT http://localhost:9200/products/_doc/93?pretty -H'Content-Type: application/json' -d'{"id":93,"type":"normal","category_id":13,"title":"Kingston\/金士顿 金士顿DDR3 1600 8GB","long_title":"Kingston\/金士顿 DDR3 1600 8G 台式机电脑 三代 内存条 兼容1333","on_sale":true,"rating":5,"sold_count":0,"review_count":0,"price":"239.00","category":["电脑配件","内存"],"category_path":"-10-","description":"","skus":[{"title":"DDR3 1600 8G","description":"DDR3 1600 8G","price":"439.00"},{"title":"DDR3 1600 4G","description":"DDR3 1600 4G","price":"239.00"},{"title":"DDR3 1333 4G","description":"DDR3 1333 4G","price":"259.00"}],"properties":[{"name":"品牌名称","value":"金士顿","search_value":"品牌名称:金士顿"},{"name":"传输类型","value":"DDR3","search_value":"传输类型:DDR3"},{"name":"内存容量","value":"4GB","search_value":"内存容量:4GB"},{"name":"内存容量","value":"8GB","search_value":"内存容量:8GB"}]}'
```



### 查询

```shell
# 直接用curl进行查询请求
curl -XGET http://localhost:9200/products/_search/?pretty -H 'Content-Type: application/json' -d '{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "on_sale": true
          }
        }
      ],
      "must": [
        {
          "multi_match": {
            "query": "金士顿",
            "type": "best_fields",
            "fields": [
              "title^3",
              "long_title^2",
              "category^2",
              "description",
              "skus_title",
              "skus_description",
              "properties_value"
            ]
          }
        }
      ]
    }
  },
  "aggs": {
    "properties_count": {
      "nested": {
        "path": "properties"
      },
      "aggs": {
        "properties_name": {
          "terms": {
            "field": "properties.name"
          },
          "aggs": {
            "properties_value": {
              "terms": {
                "field": "properties.value"
              }
            }
          }
        }
      }
    }
  }
}'
```
对上面DSL语句的一些解释:
- 以`金士顿`作为关键字在多个字段中进行查询,`title^3`表示提升从`title`字段查询出来的结果的权重
- `properties_count`是自定义的聚合结果名称,同理后面的`properties_*`亦然
- `properties_count`聚合的作用是相当于在查询出来的结果中,将嵌套属性`properties`全部查询出来
- `properties_name`是在上一层的基础上按`properties.name`即属性名分组
- `properties_value`同理,在上一层的基础上按属性值分组



查询结果:

```json

{
  "took" : 3,
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
    "max_score" : 1.9678952,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "93",
        "_score" : 1.9678952,
        "_source" : {
          "id" : 93,
          "type" : "normal",
          "category_id" : 13,
          "title" : "Kingston/金士顿 金士顿DDR3 1600 8GB",
          "long_title" : "Kingston/金士顿 DDR3 1600 8G 台式机电脑 三代 内存条 兼容1333",
          "on_sale" : true,
          "rating" : 5,
          "sold_count" : 0,
          "review_count" : 0,
          "price" : "239.00",
          "category" : [
            "电脑配件",
            "内存"
          ],
          "category_path" : "-10-",
          "description" : "",
          "skus" : [
            {
              "title" : "DDR3 1600 8G",
              "description" : "DDR3 1600 8G",
              "price" : "439.00"
            },
            {
              "title" : "DDR3 1600 4G",
              "description" : "DDR3 1600 4G",
              "price" : "239.00"
            },
            {
              "title" : "DDR3 1333 4G",
              "description" : "DDR3 1333 4G",
              "price" : "259.00"
            }
          ],
          "properties" : [
            {
              "name" : "品牌名称",
              "value" : "金士顿",
              "search_value" : "品牌名称:金士顿"
            },
            {
              "name" : "传输类型",
              "value" : "DDR3",
              "search_value" : "传输类型:DDR3"
            },
            {
              "name" : "内存容量",
              "value" : "4GB",
              "search_value" : "内存容量:4GB"
            },
            {
              "name" : "内存容量",
              "value" : "8GB",
              "search_value" : "内存容量:8GB"
            }
          ]
        }
      },
      {
        "_index" : "products",
        "_type" : "_doc",
        "_id" : "91",
        "_score" : 1.6602381,
        "_source" : {
          "id" : 91,
          "type" : "normal",
          "category_id" : 13,
          "title" : "Kingston/金士顿 HX424C15FB/8",
          "long_title" : "金士顿 骇客神条 ddr4 2400 8g 台式机 电脑 四代内存条 吃鸡内存",
          "on_sale" : true,
          "rating" : 5,
          "sold_count" : 0,
          "review_count" : 0,
          "price" : "399.00",
          "category" : [
            "电脑配件",
            "内存"
          ],
          "category_path" : "-10-",
          "description" : "",
          "skus" : [
            {
              "title" : "8GB 黑色",
              "description" : "8GB 2400 DDR4 黑色",
              "price" : "549.00"
            },
            {
              "title" : "8GB 绿色",
              "description" : "8GB 2400 DDR4 绿色",
              "price" : "529.00"
            },
            {
              "title" : "16GB",
              "description" : "2400 16GB",
              "price" : "1299.00"
            },
            {
              "title" : "4GB",
              "description" : "2400 4GB",
              "price" : "399.00"
            }
          ],
          "properties" : [
            {
              "name" : "品牌名称",
              "value" : "金士顿",
              "search_value" : "品牌名称:金士顿"
            },
            {
              "name" : "内存容量",
              "value" : "8GB",
              "search_value" : "内存容量:8GB"
            },
            {
              "name" : "传输类型",
              "value" : "DDR4",
              "search_value" : "传输类型:DDR4"
            },
            {
              "name" : "内存容量",
              "value" : "4GB",
              "search_value" : "内存容量:4GB"
            },
            {
              "name" : "内存容量",
              "value" : "16GB",
              "search_value" : "内存容量:16GB"
            }
          ]
        }
      }
    ]
  },
  "aggregations" : {
    "properties_count" : {
      "doc_count" : 9,
      "properties_name" : {
        "doc_count_error_upper_bound" : 0,
        "sum_other_doc_count" : 0,
        "buckets" : [
          {
            "key" : "内存容量",
            "doc_count" : 5,
            "properties_value" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "4GB",
                  "doc_count" : 2
                },
                {
                  "key" : "8GB",
                  "doc_count" : 2
                },
                {
                  "key" : "16GB",
                  "doc_count" : 1
                }
              ]
            }
          },
          {
            "key" : "传输类型",
            "doc_count" : 2,
            "properties_value" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "DDR3",
                  "doc_count" : 1
                },
                {
                  "key" : "DDR4",
                  "doc_count" : 1
                }
              ]
            }
          },
          {
            "key" : "品牌名称",
            "doc_count" : 2,
            "properties_value" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "金士顿",
                  "doc_count" : 2
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```



## 参考资料

- [《L06 Laravel 教程 - 电商进阶 ( Laravel 6.x )》](https://learnku.com/courses/ecommerce-advance/6.x)

