### Full Text Query

#### match_phrase query

##### 简单查询

示例
```js
GET /_search
{
    "query": {
        "match_phrase" : {
            "message" : "泡菜-杨小东"
        }
    }
}
```
> 类似 match 查询， match_phrase 查询首先将查询字符串解析成一个词项列表，然后对这些词项进行搜索，但只保留那些包含 全部 搜索词项，且 位置 与搜索词项相同的文档。 比如对于 quick fox 的短语搜索可能不会匹配到任何文档，因为没有文档包含的 quick 词之后紧跟着 fox 。
ElasticSearch引擎首先分析（analyze）查询字符串，从分析后的文本中构建短语查询，这意味着必须匹配短语中的所有分词，并且保证各个分词的相对位置不变
- 例如:
    - 使用ik中文分词器进行分词，在查询整个的message内容*泡菜-杨小东*能搜索到
    - 使用*泡菜*能搜索到
        - *泡* 无法搜索 （ik中文分词器的行为会将*泡菜*作为整个的词，所以无法通过单个词进行匹配）
        - *菜* 无法检索 （ik中文分词器的行为会将*泡菜*作为整个的词）
    - 使用*杨小东*可以搜索到
        - *杨小东* 查询时会被分词*杨*、*小*、*东*。且出现顺序一致
                                                    
    - 使用*小*能搜索到
        - *杨小东* 不是一个词，所以被拆分*杨*、*小*、*东*，因此能搜索到
    - 使用*东*能搜到
        - *杨小东* 不是一个词，所以被拆分*杨*、*小*、*东*，因此能搜索到
    - 使用*杨东*搜索不到
        -  *杨小东* 不是一个词，所以被拆分*杨*、*小*、*东*。查询时*杨东*被分词*杨*、*东*，但是因为出现顺序不匹配所以搜索不到
    - 使用*泡菜杨*搜索得到
        -  *泡菜-杨* 分词时会忽略*-*标点以及特殊字符。
- 对比match查询
```js
GET /_search
{"query": {
        "match" : {
           "name" :  "张振宇"
        }
    }
}

```
此查询就会把包含*张* 、*振*、 *宇*、只要有一个词匹配上就会从结果返回

##### 指定分词器进行分词查询

```js
GET /_search

{
    "query": {
        "match" : {
           "name" : {
                "query" : "张振宇",
                "analyzer" : "ik_max_word"
            }
        }
    }
}
```

#### Match Phrase Prefix Query

与Match Phrase类似，不同之处在于他支持前缀匹配的方式。例如 *Technical background:Major in v* 会以最后一个*v*作为前缀匹配。max_expansions参数会控制能够匹配该前缀的词条的数量。它会找到首个以v开头的词条然后开始收集(以字母表顺序)直到所有以*v*开头的词条都被遍历了或者得到了比max_expansions更多的词条。

> "aboutSelf": "Technical background:Major in vehicle engineering(undergraduate and graduate),comprehensive knowledge in Automotive field ;Participare in several project in BBAC ,familar with Automotive project management. ●Ability to apply English:Excellent oral and written skills in Chinese and English ●Organization and communication ability:Served as monitor and president of student union during graduate and undergraduate ;Strong coordination and organizational skills,strong interpersonal and communication skills ●Characteristics:Positive,responsible,adaptable,strong ability to cope with stress"

- 检索以上文档
```js
GET /_search

{"query": {
        "match_phrase_prefix" : {
           "aboutSelf" : {
                "query" : "Technical background:Major in v"
               
            }
        }
    }
}
```
- *Technical background:Major in v*，查询*Technical background:Major in* 紧接着以*v*开头的单词的文档


```js
GET /_search

{"query": {
        "match_phrase_prefix" : {
           "aboutSelf" : {
                "query" : "Technical background:Major in c",
                "slop":4
               
            }
        }
    }
}
```
- *slop* 表示允许major 到以c通配符开始的单词出现的距离-即在4个单词间隔内以c通配符开始的单词出现则命中。*in*为介词分词时会自动忽略。
    - major - - - - company 能匹配
    - major - - - - company slop=5 也能匹配.  *(-)表示一个单词*

#### Multi Match Query
