## 06 텍스트 및 수치형 쿼리 - 1일차

6장에서는 텍스트와 숫자 값을 검색하는데 사용하는 쿼리를 살펴본다. 

<br>

### term 쿼리 사용

term 쿼리는 내부적으로 텀과 매치되는 모든 도큐먼트를 수집하고 점수로 정렬한다.

~~~javascript
POST /test-index/_search
{
  "query": {
      "term": {
          "uuid": "11111"
      }
  }
}
------
{
  "took" : 1,
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
    "max_score" : 0.9808292,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.9808292,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0
        }
      }
    ]
  }
}

~~~

<br>

filter로 실행하려면 bool 퀴리로 래핑된 쿼리를 사용해야 한다.

응답값 중 _score를 보면 filter 응답은 0.0이다. **score가 중요하지 않다면 filter 쿼리**를 사용하자. 

~~~javascript
POST /test-index/_search
{
  "query": {
      "bool": {
          "filter": {
              "term": {
                  "uuid": "11111"
              }
          }
      }
  }
}

-----
  
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
    "max_score" : 0.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.0,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0
        }
      }
    ]
  }
}

~~~

<br>

term 쿼리를 올바르게 사용하려면 **어떤 필드가 어떻게 색인되있는지** 주의해야 한다.

**KeywordAnalyzer는 기본적으로 토큰화되지 않은 필드에 사용**해서 단일 토큰처럼 문자열을 변형하지 않고 저장한다. keyword type을 사용하는 필드는 내부적으로 KeywordAnalyzer를 사용한다.

**StandardAnalyzer는 공백과 구두점으로 토큰을 만든다. 모든 토큰은 소문자로 변환**한다. Elasticsearch에서 사용하는 기본 분석기다. text type을 사용하는 필드는 내부적으로 StandardAnalyzer를 사용한다.

> Elasticsearch에서 제공하는 다양한 내장형 분석기 : https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html

```javascript
POST /test-index/_search
{
  "query": {
      "term": {
           //Joe 로 검색하면 안됨
          "parsedtext": "joe"
      }
  }
}

---
  
{
  "took" : 2,
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
    "max_score" : 1.0126973,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0126973,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0
        }
      }
    ]
  }
}
```

<br>

terms 쿼리 사용, 다중 term 검색

~~~javascript
POST /test-index/_search
{
  "query": {
      "terms": {
          "uuid": ["11111", "22222"]
      }
  }
}

-------
  
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0
        }
      }
    ]
  }
}
~~~

> 책에서 `"minimum_should_match": 2` 로 최소로 만족하는 값의 갯수를 사용하게 하는데 실제로 해보면 안됨.
>
> ES 6.0부터 지원안한다고 함, 대신 `bool, should` query를 사용하면 됨
>
> <https://stackoverflow.com/questions/40837678/terms-query-does-not-support-minimum-match-in-elasticsearch-2-3-3>


<br>


### prefix 쿼리 사용

term 의 시작 부분만 알고 있을 때 사용한다. --> like 검색

~~~javascript
POST /test-index/_search
{
  "query": {
      "prefix": {
          "uuid": "222"
      }
  }
}
-----
{
  "took" : 9,
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
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0
        }
      }
    ]
  }
}
~~~

<br>

prefix 쿼리로 텍스트 끝 부분을 검색할 수도 있다.

~~~javascript
PUT /test-index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "reverse_analyzer": {
          "type": "custom", 
          "tokenizer": "keyword",
          "filter": [
            "lowercase",
            "reverse"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "pos": {
        "type": "integer",
        "store": true
      },
      "uuid": {
        "store": true,
        "type": "keyword"
      },
      "parsedtext": {
        "term_vector": "with_positions_offsets",
        "store": true,
        "type": "text"
      },
      "name": {
        "term_vector": "with_positions_offsets",
        "store": true,
        "fielddata": true,
        "type": "text",
        "fields": {
          "raw": {
            "type": "keyword"
          }
        }
      },
      "title": {
        "term_vector": "with_positions_offsets",
        "store": true,
        "type": "text",
        "fielddata": true,
        "fields": {
          "raw": {
            "type": "keyword"
          }
        }
      },
      "filename": {
        "type": "keyword",
        "fields": {
          "rev": {
            "type": "text",
            "analyzer": "reverse_analyzer"
          }
        }
      }
    }
  }
}


---

POST /test-index/_search
{
  "query": {
      "prefix": {
          "filename.rev": "gnp."
      }
  }
}

---
  
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
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      }
    ]
  }
}

~~~

<br>

### wildcard 쿼리 사용

wildcard 쿼리는 term의 일부분을 알고 있을 때 사용한다.

~~~javascript
POST /test-index/_search
{
  "query": {
      "wildcard": {
          "uuid": "22?2*"
      }
  }
}

---

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
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      }
    ]
  }
}
~~~

<br>

> *: 0개 이상 문자가 매치된다는 의미
>
> ?: 한 개의 문자가 매치된다는 의미



### regexp( 정규 표현식 ) 쿼리

regexp 쿼리 속도를 올리려면 와일드카드로 시작하지 않는 정규 표현식을 지정하는 것이 좋다.

~~~javascript
POST /test-index/_search
{
  "query": {
      "regexp": {
        "parsedtext":{
          "value": "j.*"
        }
      }
  }
}

---


{
  "took" : 1,
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
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0,
          "filename" : "test.txt"
        }
      }
    ]
  }
}

~~~

<br>

 `flags` 연산자는 특정 연산자를 활성화 / 비활성화 할 수 있습니다.

<https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-regexp-query.html#regexp-syntax>

~~~javascript
POST /test-index/_search
{
  "query": {
      "regexp": {
        "parsedtext":{
          "value": "aaa.+&.+bbb",
          "flags" : "INTERSECTION|COMPLEMENT|EMPTY"
        }
      }
  }
}

---

{
  "took" : 1,
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
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "position" : 3,
          "parsedtext" : "aaabbb",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      }
    ]
  }
}


----------------------



POST /test-index/_search
{
  "query": {
      "regexp": {
        "parsedtext":{
          "value": "aaa.+&.+bbb",
          "flags" : "COMPLEMENT|EMPTY"
        }
      }
  }
}


---
  
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}

~~~

<br>

### span 쿼리 사용

 span 쿼리들은 **텍스트 토큰 위치를 통해 텍스트 토큰 시퀀스를 제어하는 쿼리 그룹**이다.

<br>

#### span_first

span_term이 첫 토큰이나 그 근처에서 매치해야하는 쿼리를 지정한다.

- 5번째 토큰안에서 "joe"를 찾아라

~~~javascript
POST /test-index/_search
{
  "query": {
      "span_first": {
        "match":{
          "span_term": {
            "parsedtext" : "joe"
          }
        },
        "end": 5
      }
  }
}

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
    "max_score" : 1.1374958,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.1374958,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0,
          "filename" : "test.txt"
        }
      }
    ]
  }
}
~~~

<br>

#### span_or

span_or 쿼리는 span 쿼리에서 다중 값을 지정하는데 사용한다.

~~~javascript
POST /test-index/_search
{
  "query": {
      "span_or": {
        "clauses": [
          {
            "span_term": {
              "parsedtext" : "testere"
            }
          },
          {
            "span_term": {
              "parsedtext" : "joe"
            }
          }
        ]
      }
  }
}

---

{
  "took" : 1,
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
    "max_score" : 2.5077808,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 2.5077808,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0,
          "filename" : "test.txt"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.792371,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      }
    ]
  }
}

~~~

<br>

#### span_multi

wildcard, prefix, range 와 같은 다중 텀 쿼리를 감싸는 span_multi 쿼리가 있다.

~~~javascript
POST /test-index/_search
{
  "query": {
      "span_multi": {
        "match": {
          "prefix": {
            "parsedtext": {
              "value": "jo"
            }
          }
        }
      }
  }
}

---

{
  "took" : 2,
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
    "max_score" : 1.1374958,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.1374958,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0,
          "filename" : "test.txt"
        }
      }
    ]
  }
}

~~~

<br>

더 많은 span 쿼리들을 알고 싶다면 여기를 보세요

span queries : <https://www.elastic.co/guide/en/elasticsearch/reference/current/span-queries.html>



### match 쿼리 사용

testere과 joe를 포함한 parsedtext 문자열을 가진 문서를 찾아라

~~~javascript
POST /test-index/_search
{
  "query": {
      "match": {
        "parsedtext": {
          "query": "testere joe",
          "operator": "and"
        }
      }
  }
}

---
  
{
  "took" : 1,
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
    "max_score" : 1.792371,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.792371,
        "_source" : {
          "position" : 1,
          "parsedtext" : "Joe Testere nice guy",
          "name" : "Joe Tester",
          "uuid" : "11111",
          "price" : 4.0,
          "filename" : "test.txt"
        }
      }
    ]
  }
}
~~~

<br>





