# 08 집계

## 목차

---

01. stats 집계
02. terms 집계
03. significant_terms 집계
04. range 집계
05. histogram 집계
06. date_histogram 집계
07. filter 집계
08. filters 집계
09. global 집계
10. children 집계
11. nested 집계
12. top_hits 집계
13. matrix_stats 집계

<br>

# 들어가기에 앞서 이론.

## ElasticSearch에서의 집계(aggregations)란?
* SQL의 GROUP BY와 유사.
* 카운팅, 통계, 히스토리와 같은 데이터 추출할 때 사용.
* 항상 검색 조회 시 수행, 일반적으로 맵/리듀스 방식으로 연산.
    * mapper의 수행은 샤드에 분산, reducer는 호출 노드에서 실행 -> 정확한 reducer의 값을 취하기 위해.
    * 많은 메모리의 사용량 필요
* sub-aggs 지원(집계 트리로 쿼리 가능)

## ElasticSearch에서의 버킷이란?
* 특정 조건에 충족하는 도큐먼트들의 집합
    * terms 쿼리로 성별을 구분할 시 각 [여성, 남성] 버킷에 담기게 됨

## ElasticSearch에서의 집계 종류
1. 버킷 에그리게이터( bucketing aggregators )
    * SQL의 Group By라고 생각하면 됨.
    * 중첩 애그리게이터의 연산 시 각 버킷에서 이후 연산 진행.

2. 메트릭 애그리게이터( metric aggregator )
    * 특정 필드에 대해 연산 통계시 사용.
    * SQL의 count, min, max, avg, sum 와 같이 생각하면 됨.
3. 매트릭스 애그리게이터( matrix aggregator )
    * 다중 필드에서 동작, 추출한 값을 기반으로 행렬 결과를 만든다.
    * 향후 변경 또는 제거될 수 있음.
4. 파이프라인 애그리게이터( pipeline aggregator )
    * 다른 aggregator의 결과 값을 input으로 사용하는 aggregator.
    * buckets_path 를 통해 다른 aggs의 값을 받음.
    * 향후 변경 또는 제거될 수 있음.
    
## 01. stats 집계
가장 일반적으로 사용하는 메트릭 집계.

#### age에 대해 메트릭 집계 요청 쿼리
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "age_stats": {
      "extended_stats": { // ex) sum만 얻고 싶으면 sum 사용.
        "field": "age"
      } 
    }
  }
}
````
---

#### Result

````js
{
  "took" : 454,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 986,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "age_stats" : {
      "count" : 986, // age 필드를 가진 도큐먼트 갯수
      "min" : 1.0, // age의 최솟값
      "max" : 100.0, // age의 최댓값
      "avg" : 53.41480730223124, // age의 평균값
      "sum" : 52667.0, // age의 합
      "sum_of_squares" : 3616329.0, // age의 제곱합
      "variance" : 814.5348314537396, // age의 분산
      "std_deviation" : 28.54005661265828, // age의 표준편차
      "std_deviation_bounds" : {
        "upper" : 110.4949205275478,
        "lower" : -3.66530592308532
      }
    }
  }
}
````

## 02. terms 집계

가장 많이 사용하는 버킷 집계이며, 개별 텀값으로 버킷 생성.


#### tag 필드로 terms 집계 연산

````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "tag": {
      "terms": {
        "field": "tag", // group by 할 필드.
        "size": 11, // 반환될 텀 값 갯수
        "min_doc_count": 10, // 최소 버킷에 들어있어야 하는 도큐먼트 갯수.
        "include": ".*", // 정규 표현식으로 group by 할 필드 필터링 가능.
        // exclude 도 존재
        "order": { // 정렬 방법 지정. 기본  => _count
          "_key": "asc" // _term deprecated, _key 사용.
        } 
      } 
    }
  }
}
````
---
#### Result
````js
  "aggregations" : {
    "tag" : {
      "doc_count_error_upper_bound" : 25,
      "sum_other_doc_count" : 2660,
      "buckets" : [
        {
          "key" : "porro",
          "doc_count" : 21
        },
        {
          "key" : "ad",
          "doc_count" : 19
        },
        {
          "key" : "laborum",
          "doc_count" : 19
        },
        {
          "key" : "facilis",
          "doc_count" : 18
        },
        {
          "key" : "dolor",
          "doc_count" : 17
        },
        {
          "key" : "aperiam",
          "doc_count" : 16
        },
        {
          "key" : "ipsam",
          "doc_count" : 16
        },
        {
          "key" : "magnam",
          "doc_count" : 16
        },
        {
          "key" : "sit",
          "doc_count" : 16
        },
        {
          "key" : "deleniti",
          "doc_count" : 15
        },
        {
          "key" : "maiores",
          "doc_count" : 14
        }
      ]
    }
  }
}
````
#### terms의 경우 필터링 집계 연산으로 많이 사용
* ####  Ex. ) tag로 버킷 생성 후 각 버킷에서 평균 나이 집계.
````js
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "genders": {
      "terms": {
        "field": "tag",
        "order": { // 하위 집계 값을 기준으로 정렬.
          "avg_age": "desc"
        }
      },
      "aggs": {
        "avg_age":{
          "avg": {
            "field": "age"
          }
        }
      }
    }
  }
}
````
---
#### Result

````js
"aggregations" : {
    "genders" : {
      "doc_count_error_upper_bound" : -1,
      "sum_other_doc_count" : 2828,
      "buckets" : [
        {
          "key" : "voluptatem",
          "doc_count" : 3,
          "avg_age" : {
            "value" : 97.33333333333333
          }
        },
        {
          "key" : "dolorum",
          "doc_count" : 2,
          "avg_age" : {
            "value" : 96.5
          }
        },
        {
          "key" : "culpa",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 95.0
          }
        },
        {
          "key" : "deserunt",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 95.0
          }
        },
        {
          "key" : "repudiandae",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 95.0
          }
        },
        {
          "key" : "distinctio",
          "doc_count" : 2,
          "avg_age" : {
            "value" : 94.0
          }
        },
        {
          "key" : "ducimus",
          "doc_count" : 5,
          "avg_age" : {
            "value" : 93.2
          }
        },
        {
          "key" : "adipisci",
          "doc_count" : 2,
          "avg_age" : {
            "value" : 93.0
          }
        },
        {
          "key" : "dicta",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 93.0
          }
        },
        {
          "key" : "perspiciatis",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 93.0
          }
        }
      ]
    }
  }
}
````


#### terms 집계의 제어를 위한 script 필드
* #### Ex. ) tag필드 terms 집계한 버킷 key 변경.
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "tag" : {
      "terms": {
        "field": "tag",
        "script": {
          "source" : "'geonyeong: '+doc['tag'].value"
        }
      }
    }
  }
}
````
---
#### Result

````js
"aggregations" : {
    "tag" : {
      "doc_count_error_upper_bound" : 12,
      "sum_other_doc_count" : 820,
      "buckets" : [
        {
          "key" : "geonyeong: ad",
          "doc_count" : 21
        },
        {
          "key" : "geonyeong: accusantium",
          "doc_count" : 19
        },
        {
          "key" : "geonyeong: alias",
          "doc_count" : 19
        },
        {
          "key" : "geonyeong: aliquam",
          "doc_count" : 19
        },
        {
          "key" : "geonyeong: aspernatur",
          "doc_count" : 17
        },
        {
          "key" : "geonyeong: cupiditate",
          "doc_count" : 17
        },
        {
          "key" : "geonyeong: ab",
          "doc_count" : 14
        },
        {
          "key" : "geonyeong: accusamus",
          "doc_count" : 14
        },
        {
          "key" : "geonyeong: aut",
          "doc_count" : 14
        },
        {
          "key" : "geonyeong: aperiam",
          "doc_count" : 12
        }
      ]
    }
  }
````


## 03. significant_terms 집계
* 버킷 집계 유형 중 하나의 집계
* 텀간의 관계발견을 위해 사용.
* 포그라운드, 백그라운드 연산을 하기때문에 CPU를 많이 사용하는 집계 쿼리

#### tag에 ["ullam", "in", "ex"] 있는 도큐먼트 필터 후 tag 필드 significant_terms 집계
````js
GET test-index/_search?size=0
{
  "query": {
    "terms": {
      "tag": ["ullam", "in", "ex"] 
    }
  },
  "aggs": {
    "significant_tags" : {
      "significant_terms": {
        "field": "tag"
      }
    }
  }
}
````
---

#### Result

````js
"aggregations" : {
    "significant_tags" : {
      "doc_count" : 45,
      "bg_count" : 986,
      "buckets" : [
        {
          "key" : "ullam", // group by 된 텀
          "doc_count" : 17, // 해당 텀이 가진 도큐먼트 갯수
          "score" : 7.899753086419754, // 포그라운드, 백그라운드의 빈도수 점수.
          "bg_count" : 17 // 전체 도큐먼트에서 key를 가지고 있는 도큐먼트 갯수
        },
        {
          "key" : "in",
          "doc_count" : 15,
          "score" : 6.97037037037037,
          "bg_count" : 15
        },
        {
          "key" : "ex",
          "doc_count" : 14,
          "score" : 6.50567901234568,
          "bg_count" : 14
        },
        {
          "key" : "vitae",
          "doc_count" : 3, // query 연산에서 filter 된 도큐먼트 중 vitae 태그를 가지고 있는 도큐먼트 갯수
          "score" : 0.6637037037037037,
          "bg_count" : 6 // 전체 도큐먼트에서 vitae를 가지고 있는 도큐먼트 갯수
        },
        {
          "key" : "necessitatibus",
          "doc_count" : 3,
          "score" : 0.3317171717171717,
          "bg_count" : 11
        }
      ]
    }
  }
````

## 04. range 집계
* 버킷 집계 유형 중 하나의 집계
* 범위형 집계

#### 가격과 기간별 버킷 생성 <br >price = [ ~10, 10~20, 20~100, 100~], date = [2012-01-01~2012-07-01, 2012-07-01~2012-12-31, 2013-01-01~2013-12-31]

````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "prices": {
      "range": {
        "field": "price",
        "ranges": [
          {
            "to": 10
          },{
            "from": 10, 
            "to": 20
          },{
            "from": 20, 
            "to": 100
          },{
            "from": 100
          }
        ]
      }
    },
    "range": {
      "range": {
        "field": "date",
        "ranges": [
          {
            "from": "2012-01-01",
            "to": "2012-07-01"
          },{
            "from": "2012-07-01",
            "to": "2012-12-31"
          },{
            "from": "2013-01-01",
            "to": "2013-12-31"
          }
        ]
      }
    }
  }
}
````

---
#### Result

````js
"aggregations" : {
    "range" : {
      "buckets" : [
        {
          "key" : "2012-01-01T00:00:00.000Z-2012-07-01T00:00:00.000Z", 
          "from" : 1.325376E12,
          "from_as_string" : "2012-01-01T00:00:00.000Z",
          "to" : 1.3411008E12,
          "to_as_string" : "2012-07-01T00:00:00.000Z",
          "doc_count" : 99
        },
        {
          "key" : "2012-07-01T00:00:00.000Z-2012-12-31T00:00:00.000Z",
          "from" : 1.3411008E12,
          "from_as_string" : "2012-07-01T00:00:00.000Z",
          "to" : 1.356912E12,
          "to_as_string" : "2012-12-31T00:00:00.000Z",
          "doc_count" : 87
        },
        {
          "key" : "2013-01-01T00:00:00.000Z-2013-12-31T00:00:00.000Z",
          "from" : 1.3569984E12,
          "from_as_string" : "2013-01-01T00:00:00.000Z",
          "to" : 1.388448E12,
          "to_as_string" : "2013-12-31T00:00:00.000Z",
          "doc_count" : 179
        }
      ]
    },
    "prices" : {
      "buckets" : [
        {
          "key" : "*-10.0",
          "to" : 10.0,
          "doc_count" : 103
        },
        {
          "key" : "10.0-20.0",
          "from" : 10.0,
          "to" : 20.0,
          "doc_count" : 105
        },
        {
          "key" : "20.0-100.0",
          "from" : 20.0,
          "to" : 100.0,
          "doc_count" : 778
        },
        {
          "key" : "100.0-*",
          "from" : 100.0,
          "doc_count" : 0
        }
      ]
    }
  }
````

#### date_range 집계(format 가능) 
* #### ip_range(data type ip 명시)도 가능.

#### tag 필드에 ["ullam", "in", "ex"]가 속해있는 도큐먼트 필터링 후 date=[~6개월 전, 6개월 전 ~ 현재] 집계
````js
GET test-index/_search?size=0
{
  "query": {
    "terms": {
      "tag": ["ullam", "in", "ex"]
    }
  },
  "aggs": {
    "data" :{
      "date_range": {
        "field": "date",
        "format": "MM-yyyy", 
        "ranges": [
          {
            "to": "now-6M/M" // 6개월 이전
          },
          {
            "from" :"now-6M/M" //  6개월 이전에서 현재까지
          }
        ]
      }
    }
  }
}
````
---
Result
````js
  "aggregations" : {
    "data" : {
      "buckets" : [
        {
          "key" : "*-10-2018",
          "to" : 1.538352E12,
          "to_as_string" : "10-2018",
          "doc_count" : 986
        },
        {
          "key" : "10-2018-*",
          "from" : 1.538352E12,
          "from_as_string" : "10-2018",
          "doc_count" : 0
        }
      ]
    }
  }
````

## 05. histogram 집계
* 버킷 집계 유형 중 하나의 집계
* 히스토리를 위한 집계
* 개별 샤드에서 분산 처리 후 검색 노드에서 집계 수행.
* 수치형 필드에만 동작(boolean, integer, long, float, date/datetime)
* script 지원

#### 20년 간격의 연령대 집계, 50달러 간격의 가격 집계

````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "age" : { // 20년 간격의 연령대 집계
      "histogram": {
        "field": "age",
        "interval": 20
      }
    },
    "price": { // 50달러 간격의 가격 집계
      "histogram": {
        "field": "price",
        "interval": 50.0
      }
    }
  }
}
````
---

#### Result

````js
"aggregations" : {
    "price" : {
      "buckets" : [
        {
          "key" : 0.0,
          "doc_count" : 484
        },
        {
          "key" : 50.0,
          "doc_count" : 502
        }
      ]
    },
    "age" : {
      "buckets" : [
        {
          "key" : 0.0,
          "doc_count" : 154
        },
        {
          "key" : 20.0,
          "doc_count" : 183
        },
        {
          "key" : 40.0,
          "doc_count" : 203
        },
        {
          "key" : 60.0,
          "doc_count" : 196
        },
        {
          "key" : 80.0,
          "doc_count" : 242
        },
        {
          "key" : 100.0,
          "doc_count" : 8
        }
      ]
    }
  }
````
#### script 사용 예제
* ####  age를 3 곱한 값들에 대해서 5년 간격으로 집계


````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "age" : {
      "histogram": {
        "field": "age",
        "script" : "_value*3",
        "interval": 5
      }
    }
  }
}
````
---
#### Result
````js
"aggregations" : {
    "age" : {
      "buckets" : [
        {
          "key" : 0.0,
          "doc_count" : 50
        },
        {
          "key" : 20.0,
          "doc_count" : 56
        },
        {
          "key" : 40.0,
          "doc_count" : 48
        },
        {
          "key" : 60.0,
          "doc_count" : 71
        },
        {
          "key" : 80.0,
          "doc_count" : 63
        },
        {
          "key" : 100.0,
          "doc_count" : 49
        },
        {
          "key" : 120.0,
          "doc_count" : 76
        },
        {
          "key" : 140.0,
          "doc_count" : 67
        },
        {
          "key" : 160.0,
          "doc_count" : 60
        },
        {
          "key" : 180.0,
          "doc_count" : 69
        },
        {
          "key" : 200.0,
          "doc_count" : 71
        },
        {
          "key" : 220.0,
          "doc_count" : 56
        },
        {
          "key" : 240.0,
          "doc_count" : 100
        },
        {
          "key" : 260.0,
          "doc_count" : 85
        },
        {
          "key" : 280.0,
          "doc_count" : 57
        },
        {
          "key" : 300.0,
          "doc_count" : 8
        }
      ]
    }
````

## 06. date_histogram 집계
* 시간대 히스토리 집계
* interval 종류 = [year, quarter, month, week, day, hour, minute, second]
* time_zone - 선택사항

#### 년도별, 분기별 집계
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "date_year": {
      "date_histogram": {
        "field": "date",
        "interval": "year"
      }
    },
    "date_quarter": {
      "date_histogram": {
        "field": "date",
        "interval": "quarter",
        "time_zone": "+01:00"
      }
    }
 
  }   
}
````
---
#### Result

````js
"aggregations" : {
    "date_year" : {
      "buckets" : [
        {
          "key_as_string" : "2010-01-01T00:00:00.000Z",
          "key" : 1262304000000,
          "doc_count" : 37
        },
        {
          "key_as_string" : "2011-01-01T00:00:00.000Z",
          "key" : 1293840000000,
          "doc_count" : 179
        },
        {
          "key_as_string" : "2012-01-01T00:00:00.000Z",
          "key" : 1325376000000,
          "doc_count" : 186
        },
        {
          "key_as_string" : "2013-01-01T00:00:00.000Z",
          "key" : 1356998400000,
          "doc_count" : 179
        },
        {
          "key_as_string" : "2014-01-01T00:00:00.000Z",
          "key" : 1388534400000,
          "doc_count" : 165
        },
        {
          "key_as_string" : "2015-01-01T00:00:00.000Z",
          "key" : 1420070400000,
          "doc_count" : 200
        },
        {
          "key_as_string" : "2016-01-01T00:00:00.000Z",
          "key" : 1451606400000,
          "doc_count" : 40
        }
      ]
    }
  }
````


## 07. filter 집계
* 버킷 집계 유형 중 하나의 집계

#### tag의 ullam이 있는 버킷, age 37인 버킷 생성
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ullam_docs": {
      "filter": {
        "term": {
          "tag": "ullam"
        }
      }
    },
    "age37_docs":{ 
      "filter": {
          "term": {
            "age": 37
          }
      }
    }
  }   
}
````
---
#### Result
````js
"aggregations" : {
    "age37_docs" : {
      "doc_count" : 6
    },
    "ullam_docs" : {
      "doc_count" : 17
    }
  }
````

#### 필드가 없는(or null)의 도큐먼트 카운팅 역시 filter를 사용.

````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "missing_code": { // 현재 test-index 에는 code 란 필드는 존재 X
      "missing": {
        "field": "code"
      }
    }
  } 
}
````
---
#### Result

````js
{
    "aggregations" : {
        "missing_code" : {
           "doc_count" : 986
        }
    }
}
````

## 08. filters 집계
* 버킷 집계 유형 중 하나의 집계
* ElasticSearch가 제공하는 모든 쿼리 사용 가능 - 사용자 정의 filter
* filters 에 있는 모든 쿼리는 새로운 버킷을 생성 -> 도큐먼트 중복 가능
* filters로 매치되지 않은 도큐먼트의 수집을 위해 아래와 같은 옵션 사용.
    * other_bucket -> true/ false -> 매치되지 않은 도큐먼트의 수집 여부
    * other_bucket_key -> string -> other_bucket의 버킷 이름
* other_bucket_key를 지정할 시 자동으로 other_bucket = true

#### 2016년 1월1일 이상이고 price가 50이상, 2016년 1월1일 이하이고 price가 50이상인 버킷 생성

````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "expensive_docs": {
      "filters": {
        "other_bucket": true, 
        "other_bucket_key": "other_document", 
        "filters": {
          "2016_over_50": {
            "bool": {
              "must": [ 
                {
                  "range": {
                    "date" :{
                      "gte" : "2016-01-01"
                    }
                  }
                },{
                  "range": {
                    "price" :{
                      "gte" : "50"
                    }
                  }
                }
              ]
            }
          },
          "previous_2016_over_50":{
            "bool":{
              "must": [ 
                {
                  "range": {
                    "date" :{
                      "lt" : "2016-01-01"
                    }
                  }
                },{
                  "range": {
                    "price" :{
                      "gte" : "50"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  } 
}
````
---
#### Result

````js
"aggregations" : {
    "expensive_docs" : {
      "buckets" : {
        "2016_over_50" : {
          "doc_count" : 24
        },
        "previous_2016_over_50" : {
          "doc_count" : 478
        },
        "other_document" : { // filters에 걸리지 않은 도큐먼트의 갯수
          "doc_count" : 484
        }
      }
    }
  }
````

## 09. global 집계
* query 구문에 영향받지 않는 집계 연산


#### tag 필드에 ullam가 있는 버킷의 평균 나이, 전체 평균 나이 집계
````js
GET test-index/_search?size=0
{
  "query": {
    "term": {
      "tag": "ullam"
    }
  },
  "aggs": {
    "query_age_avg": { // tag에 ullam가 있는 도큐먼트들에서 집계 처리
      "avg": {
        "field": "age"
      }
    },
    "all_persons": { // 모든 도큐먼트에서 집계 처리
      "global": {},
      "aggs": {
        "age_global_avg": {
          "avg": {"field": "age"}
        }
      }
    }
  }
}
````
---
#### Result

````js
"aggregations" : {
    "all_persons" : {
      "doc_count" : 986,
      "age_global_avg" : {
        "value" : 53.41480730223124
      }
    },
    "query_age_avg" : {
      "value" : 53.470588235294116
    }
  }
````


## 10. children 집계
* 버킷 집계 유형 중 하나의 집계
* 6.0부터 parent-child 관계 지원 X -> join 사용.


#### parent-child index 명세
* #### join 필드 생성( type=join ), relations => 부모:자식
````js
PUT test-child
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  },
  "mappings": {
   "_doc":{
       "properties": {
            "join": {
                "type": "join",
                "relations": {
                  "question": "answer"
                }
            }
    }
  }  
   }
}
````
---
#### 샘플 데이터 추가

````js
PUT test-child/_doc/1
{
  "join": {
    "name": "question"
  },
  "body": "<p>I have Windows 2003 server and i bought a new Windows 2008 server...",
  "title": "Whats the best way to file transfer my site from server to a newer one?",
  "tags": [
    "windows-server-2003",
    "windows-server-2008",
    "file-transfer"
  ]
}
````
---

#### tag로 group by 후 해당 태그 key에 맞는 answer에 대해서 집계 수행 <br> tag 필드는 question에만 존재.

````js
POST test-child/_search?size=0
{
  "aggs": {
    "top-tags": {
      "terms": { // 먼저 tags로 aggregation
        "field": "tags.keyword",
        "size": 10
      },
      "aggs": {
        "to-answers": {
          "children": { // join 필드에서 answer인 것들에 대해서 집계 명시
            "type" : "answer" 
          },
          "aggs": {
            "top-names": {
              "terms": { // 이름으로 aggregation
                "field": "owner.display_name.keyword",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}
````
---
#### Result
````js
"aggregations" : {
    "top-tags" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "windows-server-2003", 
          "doc_count" : 3, // key에 해당하는 도큐먼트 갯수
          "to-answers" : {
            "doc_count" : 2, // windows-server-2003 태그가 있는 질문에 응답한 도큐먼트 갯수.
            "top-names" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [ // 응답한 작성자의 이름으로 버킷생성.
                {
                  "key" : "Sam",
                  "doc_count" : 1
                },
                {
                  "key" : "Troll",
                  "doc_count" : 2
                }
              ]
            }
          }
        },
        {
          "key" : "windows-server-2008",
          "doc_count" : 3,
          "to-answers" : {
            "doc_count" : 2,
            "top-names" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "Sam",
                  "doc_count" : 1
                },
                {
                  "key" : "Troll",
                  "doc_count" : 2
                }
              ]
            }
          }
        },
        {
          "key" : "file-transfer",
          "doc_count" : 1,
          "to-answers" : {
            "doc_count" : 1,
            "top-names" : {
              "doc_count_error_upper_bound" : 0,
              "sum_other_doc_count" : 0,
              "buckets" : [
                {
                  "key" : "Sam",
                  "doc_count" : 1
                }
              ]
            }
          }
        }
      ]
    }
  }
````

## 11. nested 집계
* 버킷 집계 유형 중 하나의 집계
* nested 도큐먼트 분석
* nested 지정 이후에는 어떠한 유형의 집계 가능.


#### nested 구조 index 생성
````js
PUT /test_nested
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  },
  "mappings": {
   "_doc": {
      "properties": {
        "resellers": {
          "type": "nested",
          "properties" : {
            "name" : { "type" : "text" },
            "price" : { "type" : "double" }
          }
        }
      }
    }
  }
}
````
---
#### resellers의 sub document에 대해서 가격별 집계
````js
GET test_nested/_search?size=0
{
    "aggs" : {
        "resellers" : {
            "nested" : {
                "path" : "resellers" // nested의 key 필드 지정.
            },
            "aggs" : {
                "price_count" : { "terms": { "field" : "resellers.price" } }
            }
        }
    }
}
````

---
#### Result

````js
"aggregations" : {
    "resellers" : { // resellers의 하위 도큐먼트의 price 필드에 대해서 집계
      "doc_count" : 6,
      "price_count" : {
        "doc_count_error_upper_bound" : 0,
        "sum_other_doc_count" : 0,
        "buckets" : [
          {
            "key" : 200.0,
            "doc_count" : 4
          },
          {
            "key" : 100.0,
            "doc_count" : 2
          }
        ]
      }
    }
  }
````


## 12. top_hits 집계
* 메트릭 집계 유형 중 하나
* 검색 hits를 가진 버킷 반환


#### tag로 집계한 상위 2개 버킷에서 age로 내림차순 정렬하여 2개의 hits 정보 (name, age) 추출
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "tags": {
      "terms": {
        "field": "tag", // tag로 aggregation
        "size": 2
      },
      "aggs": {
        "top_tag_hits": { // 각 tag 버킷에서 나이 많은 도큐먼트 상위 2개 
          "top_hits": {
            "sort": [{"age": {"order": "desc"}}], // 정렬
            "_source": {"includes": ["name", "age"]}, // hits 도큐먼트의 내용 projection(exclude 존재)
            "size": 2 // hits 사이즈
            // "from" hits의 시작 offset
          }
        }
      }
    }
  }
}
````
---
#### Result

````js

"aggregations" : {
    "tags" : {
      "doc_count_error_upper_bound" : 30,
      "sum_other_doc_count" : 2812,
      "buckets" : [
        {
          "key" : "laborum",
          "doc_count" : 19,
          "top_tag_hits" : {
            "hits" : {
              "total" : 19,
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "test-index",
                  "_type" : "_doc",
                  "_id" : "648",
                  "_score" : null,
                  "_source" : {
                    "name" : "Pulse",
                    "age" : 86
                  },
                  "sort" : [
                    86
                  ]
                },
                {
                  "_index" : "test-index",
                  "_type" : "_doc",
                  "_id" : "161",
                  "_score" : null,
                  "_source" : {
                    "name" : "Brainchild",
                    "age" : 84
                  },
                  "sort" : [
                    84
                  ]
                }
              ]
            }
          }
        },
        {
          "key" : "ipsam",
          "doc_count" : 16,
          "top_tag_hits" : {
            "hits" : {
              "total" : 16,
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "test-index",
                  "_type" : "_doc",
                  "_id" : "387",
                  "_score" : null,
                  "_source" : {
                    "name" : "Stephen Colbert",
                    "age" : 99
                  },
                  "sort" : [
                    99
                  ]
                },
                {
                  "_index" : "test-index",
                  "_type" : "_doc",
                  "_id" : "913",
                  "_score" : null,
                  "_source" : {
                    "name" : "Copycat",
                    "age" : 95
                  },
                  "sort" : [
                    95
                  ]
                }
              ]
            }
          }
        }
      ]
    }
  }
````

## 13. matrix_stats 집계

* 매트릭스 집계 유형 중 하나

#### 해당 인덱스에서 age, price 필드에 대해서 분석 결과 추출
````js
GET test-index/_search?size=0
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "matrixstats": {
      "matrix_stats": {
        "fields": ["age", "price"]
      }
    }
  }
}
````
---
#### Result
````js

"aggregations" : {
    "matrixstats" : {
      "doc_count" : 986,
      "fields" : [
        {
          "name" : "price",
          "count" : 986, // 필드 존재 도큐먼트 갯수
          "mean" : 50.169132895448435, // 필드 value의 평균
          "variance" : 830.5496433546824, // 평균으로부터 분산돼 있는지 측정한 값
          "skewness" : -0.04346720039378677, // 평균값 주위에 비대칭 분포를 정량화하는 측정값
          "kurtosis" : 1.8142328517750304, // 분산의 형태를 정량화하는 측정값
          "covariance" : {
            "price" : 830.5496433546824, // 다른 필드와의 연관성의 정량적 값
            "age" : 5.53695735189194
          },
          "correlation" : { // -1 ~ 1사이의 covariance 값(sigmoid?)
            "price" : 1.0,
            "age" : 0.00672842177866286
          }
        },
        {
          "name" : "age",
          "count" : 986,
          "mean" : 53.41480730223124,
          "variance" : 815.3617703689213,
          "skewness" : -0.13363545591769496,
          "kurtosis" : 1.816529045399446,
          "covariance" : {
            "price" : 5.53695735189194,
            "age" : 815.3617703689213
          },
          "correlation" : {
            "price" : 0.00672842177866286,
            "age" : 1.0
          }
        }
      ]
    }
  }
````


