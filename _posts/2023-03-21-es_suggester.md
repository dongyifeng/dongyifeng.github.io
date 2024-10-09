---
title: ES Suggester
date: 2023/3/21 21:37:00
comments: true
no_word_count: true 
reward: true 
copyright: true 
sticky: 96
categories: 
- 搜索
tags:
- 搜索
typora-root-url: ../../dongyifeng.github.io
index_img: /images/index_img/es_suggester.png
---



Elasticsearch里设计了4 种类别的 Suggester

- Term Suggester：词条建议器。对给输入的文本进进行分词，为每个分词提供词项建议。
- Phrase Suggester：短语建议器，在term的基础上，会考量多个term之间的关系
- Completion Suggester：它主要针对的应用场景就是"Auto Completion"
- Context Suggester：上下文建议器

# Term Suggester

每个 token 挑选 options 里的词，组合在一起返回给用户前端即可。



blogs 索引：text 字段。

```json
PUT /blogs/
{
  "mappings": {
    "tech": {
      "properties": {
        "body": {
          "type": "text"
        }
      }
    }
  }
}
```



Term suggester 查询：使用 <font color=red>suggest</font> 这种特殊的查询。

suggest_mode：

- missing： 只有词典里找不到词，才会为其提供相似的选项。
- popular：仅提供在索引词典中出现的词语。只返回在更多的文档中出现的建议词
- always：不管 token 是否存在于索引词典里都要给出相似项。

```json
POST /blogs/_search
{ 
  "suggest": {
    // 自定义名称
    "my-suggestion": {
      // query 
      "text": "lucne rock",
      // 表示 term suggester
      "term": {
        // 如果 query word 足够精准，就不需要 sug 了，也就是只会 sug，word 不全匹配的。
        "suggest_mode": "missing",
        // sug 的字段
        "field": "body"
      }
    }
  }
}
```



两个term的相似性是如何判断的?

ES使用了一种叫做 Levenstein edit distance的算法：其核心思想就是一个词改动多少个字符就可以和另外一个词一致。与编辑距离类似。



# Phrase Suggester

Phrase suggester在Term suggester的基础上，会考量多个term之间的关系，比如是否同时出现在索引的原文里，相邻程度，以及词频等等。



```json
POST /blogs/_search
{
  "suggest": {
    "my-suggestion": {
      "text": "lucne and elasticsear rock",
      "phrase": {
        "field": "body",
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```



执行结果：

```json
"suggest": {
    "my-suggestion": [
      {
        "text": "lucne and elasticsear rock",
        "offset": 0,
        "length": 26,
        "options": [
          {
            "text": "lucene and elasticsearch rock",
            "highlighted": "<em>lucene</em> and <em>elasticsearch</em> rock",
            "score": 0.004993905
          },
          {
            "text": "lucne and elasticsearch rock",
            "highlighted": "lucne and <em>elasticsearch</em> rock",
            "score": 0.0033391973
          },
          {
            "text": "lucene and elasticsear rock",
            "highlighted": "<em>lucene</em> and elasticsear rock",
            "score": 0.0029183894
          }
        ]
      }
    ]
  }
```



因为 lucene 和 elasticsearch 曾经在同一条原文里出现过，<font color=red>同时替换 2 个 term 的可信度更高</font>，所以打分较高，排在第一位返回

Phrase suggester 有相当多的参数用于控制匹配的<font color=red>模糊程度</font>。



# Completion Suggester

基于<font color=red>内存 FST </font>的数据结构，只能进行<font color=red>前缀检索</font>。

<font color=red>需要专门字段类型：completion</font>

创建索引

```json
PUT music
{
    "mappings": {
        "song" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion"
                },
                "title" : {
                    "type": "keyword"
                }
            }
        }
    }
}
```



插入数据：可以指定<font color=red>权重</font>。

```java
PUT music/song/1?refresh
{
    "suggest" : {
        "input": [ "Nevermind", "Nirvana" ],
        "weight" : 34
    }
}
```





搜索：在检索时 Query 也会经过 <font color=red>analyze</font> 阶段，也会讲 Query 中停用词剔除。

```json
POST blogs_completion/_search?pretty
{ "size": 0,
  "suggest": {
    "blog-suggest": {
      "prefix": "elastic i",
      "completion": {
        "field": "body"
      }
    }
  }
}
```



值得注意的一点是 Completion Suggester 在索引原始数据的时候也要经过 <font color=red>analyze</font> 阶段，取决于选用的analyzer不同，某些词可能会被 <font color=red>转换</font>，某些词可能 <font color=red>被去除</font>，这些会影响 FST 编码结果，也会 <font color=red>影响查找匹配</font>的效果。

比如：english analyzer会<font color=red>剥离掉 stop word</font>，而 is 就是其中一个，被剥离掉了！

结果：

```json
"suggest": {
    "blog-suggest": [
      {
        "text": "elastic i",
        "offset": 0,
        "length": 9,
        "options": [
          {
            "text": "Elastic is the company behind ELK stack",
            "_index": "blogs_completion",
            "_type": "tech",
            "_id": "AVrXFyn-cpYmMpGqDdcd",
            "_score": 1,
            "_source": {
              "body": "Elastic is the company behind ELK stack"
            }
          }
        ]
      }
    ]
  }
```



- skip_duplicates：是否应过滤掉重复的建议（默认为`false`）。



## 支持拼音

如果要支持拼音：

下载拼音插件：wget https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.4.0/elasticsearch-analysis-pinyin-7.4.0.zip

当我们创建索引时可以自定义分词器，通过指定映射去匹配自定义分词器



## 支持模糊搜索

**fuzzy**

```json
POST music/_search?pretty
{
  "suggest": {
    "song-suggest": {
      "prefix": "nor",
      "completion": {
        "field": "suggest",
        "fuzzy": {
          "fuzziness": 2
        }
      }
    }
  }
}
```

- fuziness 为1，表示是针对每个词语而言的，而不是总的错误的数值。



模糊性是拼写错误的简单解决方案，但具有很高的 CPU 开销和非常低的精度。



## 支持正则



```json
POST music/_search?pretty
{
  "suggest": {
    "song-suggest": {
      "regex": "n[ever|i]r",
      "completion": {
        "field": "suggest"
      }
    }
  }
}
```



# Context Suggester

completion suggester 是在索引中所有文档进行匹配，有时我们希望在一个增加过滤条件，以提高搜索的准确度。



有两种类型：**category** 和**geo**

```json
PUT place
{
    "mappings": {
        "shops" : {
            "properties" : {
                "suggest" : {
                    "type" : "completion",
                    "contexts": [
                        { # 1
                            "name": "place_type",
                            "type": "category",
                            "path": "cat"
                        },
                        { # 2
                            "name": "location",
                            "type": "geo",
                            "precision": 4
                        }
                    ]
                }
            }
        }
    }
}
```



## 类别上下文（**Category Context**）



插入数据：这些 **suggestions** 将与 cafe 和 food 类别相关联。

```json
PUT place/shops/1
{
    "suggest": {
        "input": ["timmy's", "starbucks", "dunkin donuts"],
        "contexts": {
            "place_type": ["cafe", "food"] ①
        }
    }
}
```



查询：**suggestions**可以按一个或多个类别进行过滤。 以下过滤了多个类别的**suggestions** ：

```json
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "place_type": [ "cafe", "restaurants" ]
            }
        }
    }
}
```

<font color=red>注意：当在查询时未提供类别时，将考虑所有索引文档。 应避免在类别启用完成字段上没有类别的查询，因为它会降低搜索性能。</font>



Boost 类别提权：

对某些类别的**suggestions**可以比其他类别更高。 以下内容按类别过滤**suggestions**，并增加与某些类别相关联的**suggestions**：

```json
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "place_type": [ ①
                    { "context" : "cafe" },
                    { "context" : "restaurants", "boost": 2 }
                 ]
            }
        }
    }
}
```



- context，要过滤/提升的类别的值，这是强制性的。
- boost，应该提高建议分数的因素，通过将boost乘以建议权重来计算分数，默认为1。
- prefix，是否应该将类别实为前缀，例如，如果设置为true，则可以通过指定类型的类别前缀来过滤type1，type2等类别，默认为false。







## **地理位置上下文**

一个geo上下文允许我们将一个或多个地理位置或geohash与在索引时间的建议关联，在查询时，如果建议位于地理位置特定的距离内，则可以过滤和提升建议。

在内部，地位置被编码为具有指定精度的地理位置。



地理上下文可以利用**suggestions**被显式地设置或者经由路径参数从文档中的地理点字段索引，类似于类别上下文。 将多个地理位置上下文与**suggestion**关联，将对每个地理位置的**suggestion**建立索引。 以下对具有两个地理位置上下文的**suggestion**进行索引：



插入数据：

```json
PUT place/shops/1
{
    "suggest": {
        "input": "timmy's",
        "contexts": {
            "location": [
                {
                    "lat": 43.6624803,
                    "lon": -79.3863353
                },
                {
                    "lat": 43.6624718,
                    "lon": -79.3873227
                }
            ]
        }
    }
}
```



<font color=red>查询</font>

**suggestions**可以根据它们与一个或多个地理点的接近程度而被过滤和提升。 以下过滤**suggestions**落在由地理点的编码**geohash**表示的区域内：

```json
POST place/_suggest
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "location": {
                    "lat": 43.662,
                    "lon": -79.380
                }
            }
        }
    }
}
```

> 当指定在查询时具有较低精度的位置时，将考虑落入该区域内的所有**suggestions**。



位于由**geohash**表示的区域内的**suggestions**也可以比其他**suggestion**更高，如下所示：

```json
POST place/_suggest?pretty
{
    "suggest" : {
        "prefix" : "tim",
        "completion" : {
            "field" : "suggest",
            "size": 10,
            "contexts": {
                "location": [ ①
                    {
                        "lat": 43.6624803,
                        "lon": -79.3863353,
                        "precision": 2
                    },
                    {
                        "context": {
                            "lat": 43.6624803,
                            "lon": -79.3863353
                        },
                        "boost": 2
                    }
                 ]
            }
        }
    }
}
```

上下文查询过滤的**suggestions**落在由（43.662，-79.380）的**geohash**表示的地理位置（精度为2）下方的**suggestions**，并提升落在（43.6624803，-79.3863353）的**geohash**表示形式下的默认精度为6的**suggestions**乘以因子2。



# 总结

1. 在用户<font color=red>刚开始输入</font>的过程中，使用 <font color=red>Completion Suggester</font> 进行关键词<font color=red>前缀匹配</font>、，刚开始匹配项会比较多，随着用户输入字符增多，匹配项越来越少。如果用户输入比较精准，可能 Completion Suggester 的结果已经够好，用户已经可以看到理想的备选项了。
2. 如果 <font color=red>Completion Suggester</font> 已经到了<font color=red>零匹配</font>，那么可以猜测是否用户有输入错误，这时候可以尝试一下 <font color=red>Phrase Suggester</font>。
3. 如果 <font color=red>Phrase Suggester没有找到任何 option</font>，开始尝试 <font color=red>Term Suggester</font>。
4. 需要一个搜索词库/语料库，不要和业务索引库在一起，方便维护和升级语料库
5. 支持拼音插件



<font color=red>精准程度</font>上( Precision )看： <font color=red>Completion >  Phrase > Term，</font> 而<font color=red>召回率上( Recall )则反之</font>。从性能上看，Completion Suggester是最快的，如果能满足业务需求，只用 Completion Suggester 做前缀匹配是最理想的。 Phrase 和 Term 由于是做<font color=red>倒排索引</font>的搜索，相比较而言<font color=red>性能应该要低不少</font>，应尽量控制 Suggester 用到的索引的数据量，最理想的状况是经过一定时间预热后，索引可以全量 map 到内存。
