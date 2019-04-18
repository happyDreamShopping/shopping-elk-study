## 06 텍스트 및 수치형 쿼리 - 2일차

<br>

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

match_phrase로 구문 검색도 할 수 있다 

~~~javascript
POST /test-index/_search
{
  "query": {
      "match_phrase": {
        "parsedtext": "nice guy"
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
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.6739625,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.6739625,
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
        "_score" : 0.6739625,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 0.6069386,
        "_source" : {
          "position" : 3,
          "parsedtext" : "Bill is not\n                nice guy",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      }
    ]
  }
}
~~~


<br>


match_phrase_prefix로  구문 prefix 검색도 할 수 있다 

```javascript
POST /test-index/_search
{
  "query": {
      "match_phrase_prefix": {
        "parsedtext": "nice gu"
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
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 0.6739625,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.6739625,
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
        "_score" : 0.6739625,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 0.6069386,
        "_source" : {
          "position" : 3,
          "parsedtext" : "Bill is not\n                nice guy",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      }
    ]
  }
}
```


<br>


### query_string 쿼리 사용

query_string 쿼리는 필드 규칙을 혼합해 복잡한 쿼리를 정의할 수 있는 특수한 쿼리 유형이다.

쿼리는 루씬 쿼리 파서로 파싱된다.

<br>

- fields, default_field 는 쿼리가 사용할 기본 필드를 지정한다.
- default_operator는 쿼리 파라미터에서 사용할 기본 연산자다.
- 앞서 언급한 와일드 카드나 프리픽스로 검색도 가능하다.
- p 322
- https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html

<br>

nice guy 라는 parsedtext 구문을 검색하면서 not이라는 텍스트가 없고 5보다 작은 price를 조건을 같은 쿼리

- ^5 -> 부스트 연산자로 "_score" 값을 조정할 수 있다.

~~~javascript
POST /test-index/_search
{
  "query": {
    "query_string": {
      "fields": ["parsedtext^5"],
      "query": "\"nice guy\" -parsedtext:not price:{ * TO 5}",
      "default_operator": "AND"
    }
  }
}

---
    
{
  "took" : 29,
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
    "max_score" : 4.369812,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 4.369812,
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


### simple_query_string 쿼리 사용

프로그래머는 불리언 쿼리와 다른 쿼리 유형을 사용해 복잡한 쿼리를 작성한다.

elasticsearch는 사용자에게 여러 연산자가 포함된 문자열 쿼리를 작성할 수 있는 쿼리를 제공한다.

> 구글에서 + - 연산자를 허용하는 것과 비슷한 케이스 

<br>

nice guy 텍스트를 검색하지만 not이라는 텍스트는 제외

~~~javascript
POST /test-index/_search
{
  "query": {
    "simple_query_string": {
      "query": "\"nice guy\" -not",
      "fields": ["parsedtext^5"],
      "default_operator": "AND"
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
    "max_score" : 4.369812,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 4.369812,
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
        "_score" : 4.369812,
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

### range 쿼리 사용

값 범위 검색(날짜/시간 필드도 가능)

Integer 필드 position에 대해 3이상 4미만으로 조회하라

~~~javascript
POST /test-index/_search
{
  "query": {
    "range": {
      "position": {
        "from": 3,
        "to" : 4,
        "include_lower":true,
        "include_upper": false
      }
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "position" : 3,
          "parsedtext" : "Bill is not\n                nice guy",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      },
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
~~~

<br>

### ID 쿼리 사용

Id 값으로 도큐먼트 조회

~~~javascript
POST /test-index/_search
{
  "query": {
    "ids": {
      "values": [
        "1","2"
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
          "price" : 5.0,
          "filename" : "test.png"
        }
      }
    ]
  }
}
~~~

<br>

### function_score 쿼리 사용

function_score 쿼리는 쿼리가 반환하는 도큐먼트의 점수를 제어하는 함수를 정의할 수 있다.

<https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#score-functions>



전체 필드에서 bill이라는 텀을 가지고 있고, uuid가 22222 이거나 name이 Bill Baloney인 문서를 찾아서 가중치(weight)에 따라 score를 계산하라

~~~javascript
POST /test-index/_search
{
  "query": {
    "function_score": {
      "query": {
        "query_string": {
          "query": "bill"
        }
      },
      "functions": [
        {
          "filter": { "match": { "uuid": "22222" } },
          "weight": 1
        },
        {
          "filter": { "match": { "name": "Bill Baloney" } },
          "weight": 3
        }
      ],
      "score_mode": "sum"
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
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 2.6195009,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 2.6195009,
        "_source" : {
          "position" : 2,
          "parsedtext" : "Bill Testere nice guy",
          "name" : "Bill Baloney",
          "uuid" : "22222",
          "price" : 5.0,
          "filename" : "test.png"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.7692485,
        "_source" : {
          "position" : 3,
          "parsedtext" : "Bill is not\n                nice guy",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0700248,
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
~~~

<br>

### exists 쿼리 사용

elasticsearch의 특징 중 하나는 스키마리스 색인 기능이다.

그래서 특정 필드가 포함된(or 포함되지 않은) 문서를 검색할 때 exists 쿼리를 사용한다.

~~~javascript
POST /test-index/_search
{
  "query": {
    "exists": {
      "field": "parsedtext"
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
      "value" : 4,
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
          "price" : 5.0,
          "filename" : "test.png"
        }
      },
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 1.0,
        "_source" : {
          "position" : 3,
          "parsedtext" : "Bill is not\n nice guy",
          "name" : "Bill Clinton44",
          "uuid" : "33333",
          "price" : 6.0,
          "filename" : "test.jpg"
        }
      },
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
~~~

<br>

포함되지 않은 문서를 검색

~~~javascript
POST /test-index/_search
{
  "query": {
    "bool": {
      "must_not": {
        "exists": {
          "field": "parsedtext"
        }
      }
    }
  }
}
~~~


<br>


### template 쿼리 사용

elastixsearch 는 템플릿과 이를 채울 수 있는 파라미터를 제공한다. 

~~~javascript
POST /test-index/_search/template
{
  "source": {
    "query": {
      "term": {
        "uuid": "{{value}}"
      }
    }
  },
  "params": {
    "value": "22222"
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
    "max_score" : 1.2039728,
    "hits" : [
      {
        "_index" : "test-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.2039728,
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

> 예제 데이터 및 쿼리 : <https://github.com/aparo/elasticsearch-cookbook-third-edition>

