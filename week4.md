## 05 검색 

## 목차

### 1일차

- 검색 실행
- 정렬
- 하이라이팅
- 스크롤 쿼리
- search_after

### 2일차

- 결과의 inner hits 반환
- 올바른 쿼리 제안
- 매치된 결과 카운트
- explain 쿼리
- ```match_all```, ```boolean``` 

## 검색 기본

- ES의 검색 결과는 단순히 도큐먼트의 집합 이외에도 더 많은 정보들을 제공함 

## 검색 결과 분석

### 공통 결과 

```json
"took": 8,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 472840,
    "max_score": 1,
```

- took: 쿼리 수행 소요시간(ms)
- timed_out: 타임아웃 발생 여부, 검색 파라미터로 timeout을 주면 작동함
    - 일정 시간보다 오래 걸리는 검색은 수행 안 됨
    - 검색 결과 짤리거나, 결과가 없을 수 있음 

- ```_shards``` : 검색이 수행된 샤드에 대한 정보 
- ```hits``` : 검색 결과에 대한 일반 정보

### Document별 결과 

```json
{ 
    "_index": "mall",
        "_type": "doc",
        "_id": "116504",
        "_score": 1,
        "_source": { ...
```

- ```_index``` : 인덱스
- ```_type``` : 도큐먼트의 타입. 
    - *Deprecated*
    - ```_doc``` 으로 고정되는게 좋음
- ```_score``` : 공식으로 계산된 쿼리 매칭 점수
- (여기엔 없지만) ```sort``` : 페이징되거나 정렬된 문서에 대해, 정렬/페이징에 사용한 기준값 

## 검색 질의 

- ```GET *index_name*/_search```

- 인덱스 이름은 복수로 지정 가능 
- ```POST``` 도 먹음 


### 자주 쓰는 검색옵션

- ```from ~ size``` : 페이지네이션 처리 
    - from부터 size개만큼 결과를 갖고 온다 
    - ```max_result_window``` 설정에 주의 

- ```_source``` : 도큐먼트에서 가져올 필드 지정함 
    - ```json
        "_source" : ["mall_bno", "adsr_type_cd"]
       ```
    - 프로젝션 설정을 하는 네 가지 방법이 존재합니다. 
    - 저도 지금 알았는데 ```"_source"``` 이거 느리다고 쓰지 말래요.
        - https://jjeong.tistory.com/1265

간단한 검색 시연

## 정렬

- ES의 기본 정렬 기준은 score 입니다.
    - 나중에 다룰 내용인 것 같지만...
    - https://qbox.io/blog/practical-guide-elasticsearch-scoring-relevancy
    - Term Frequency, Inverse Document Frequency, Field Length Normalization 

- 정렬 옵션을 통해 개발자가 원하는대로 정렬 기준을 통제 가능
    - 특정 포스팅의 작성 일자순 정렬
    - 작성자 이름 가나다순으로 정렬 등

### 간단한 예제

```json
GET mall/_search

  {
  "query" :{
    "match_all" : {
      
    }
  },
  "from" : 5,
  "size" : 10,
  "sort": [
    {
      "mall_bno": {
        "order": "desc"
      }
    }
  ]
}
```

- 모든 결과 중(match_all)
- ```mall_bno``` 필드를 기준으로
- 내림차순 정렬 

하라는 의미입니당.

이때 결과 값에는 아래와 같은 ```sort``` 필드가 추가됨. 

```json
"sort": ["c3930227"]
```

- ```unmapped_type``` : 값이 없는 경우 sort 파라미터의 자료형
- ```missing``` : 결과의 마지막 또는 시작 값을 입력해 누락을 관리함 
- ```mode``` 값이 여러개인 필드를 정렬하는 기준 
    - min
    - max
    - sum
    - avg
    - median 

## 하이라이팅

- 본문 중 검색어를 하이라이팅해주는 기능이 있음. 

```json
GET /_search
{
    "query" : {
        "match": { "content": "kimchy" }
    },
    "highlight" : {
        "fields" : {
            "content" : {}
        }
    }
}
```

- 필드 중
- 어떤 필드를 하이라이트 할것인지 명시해줌 

### 하이라이터의 종류 

세 가지가 있다. 

- unified
- plain
- fast vector

### 검색어 외에 다른 하이라이트를 주고 싶을 경우

- 하이라이트 쿼리 파라미터를 주면 된다.

```json
GET /_search
{
    "stored_fields": [ "_id" ],
    "query" : {
        "match": {
            "comment": {
                "query": "foo bar"
            }
        }
    },

    "highlight" : {
        "order" : "score",
        "fields" : {
            "comment" : {
                "fragment_size" : 150,
                "number_of_fragments" : 3,
                "highlight_query": {
                    "bool": {
                        "must": {
                            "match": {
                                "comment": {
                                    "query": "foo bar"
                                }
                            }
                        },
                        "should": {
                            "match_phrase": {
                                "comment": {
                                    "query": "foo bar",
                                    "slop": 1,
                                    "boost": 10.0
                                }
                            }
                        },
                        "minimum_should_match": 0
                    }
                }
            }
        }
    }
}
```

## Scroll API / Search-After 

### ES 강습회에 가서 들었던 말 

김종민님이 하신 말씀입니다. 

> 검색엔진은 페이지네이션을 위한 도구가 아니에요.

> 어떤 검색엔진도 페이지 끝까지 가는 검색엔진은 없어요.

> 구글도 20페이지 까지밖에 안 가요. 

재열님의 고민입니다. 

> 우리는 **개의 상품이 있고..... 

> ***존 서비스에서는 각 판매자들이 끝까지 탐색하고 싶어하는 니즈가 있는데.... 

> 이걸 지금 이전 다음 버튼으로 풀고 있고.... Solr 에서는 [CursorMark](https://lucene.apache.org/solr/guide/6_6/pagination-of-results.html) 라는 도구로 풀고 있는데....

> ES에선.... 어떻게 할지.... 넘 어려운감... ^^; 

### 검색엔진이 페이지네이션을 위한 도구가 아닌 이유 

- 검색 결과는 계속 바뀐다. 
- 검색결과가 정렬된 순서 또한 계속 바뀐다. 
- 특정 컨텍스트를 검색엔진 안에서 유지시키려면, 그에 수반되는 IO/Memory 등 비용 
    - ES ```max_result_window``` 설정


### Scroll API의 취지 

- 페이지네이션을 위한 도구는 아니다. 
- 매우 큰 결과 집합을 반복하는데 매우 유용하다
    - 같은 결과물을 계속 반복적으로 얻고 싶을때....
    - reindex 해야할 때....
        - reindex API 쓰는게 나음 

### 요청 예제

```json
GET mall/_search?scroll=10m
  {
  "query" :{
    "match_all" : {
      
    }
  },
  
  "sort": [
    {
      "mall_bno": {
        "order": "desc"
      }
    }
  ]
}
```

- 스크롤 파라메터에 시간 단위를 명시해줘야 함
    - 해당 시간동안 scroll context가 살아있는 것임 

### 응답 예시 

```json
{
  "_scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAABBBFkR1d1pfYTc4UUZpMm9ERGlBZEtJSVEAAAAAAAPPBxZ0WVhIY3ViNFFCcTNzRzRUeVRBUU53AAAAAAABzhYWOU5zakJpYVhRcG1XUWJFZWh3WjJYQQAAAAAAA4cAFi1JRWlmTDh6VG42ZERzaU9MTDJURHcAAAAAAAdo7RZnRnhRTjBLRFFtR0ZLYS1ITWR0TTRR",
  "took": 103,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 472840,
    "max_score": null,
    "hits": [
      {
          ...
```

- ```scroll_id``` : 스크롤 레코드에 사용하는 값. 
    - 다음 검색에 아래와 같이 전송할 수 있겠음 

    ```json
    GET _search/scroll?scroll=10m
    { 
        "scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAABBBFkR1d1pfYTc4UUZpMm9ERGlBZEtJSVEAAAAAAAPPBxZ0WVhIY3ViNFFCcTNzRzRUeVRBUU53AAAAAAABzhYWOU5zakJpYVhRcG1XUWJFZWh3WjJYQQAAAAAAA4cAFi1JRWlmTDh6VG42ZERzaU9MTDJURHcAAAAAAAdo7RZnRnhRTjBLRFFtR0ZLYS1ITWR0TTRR"
    }
    ```

### Search-After

**이거 다음부터 검색해!**

- ```from-size``` : from 까지 모든 검색결과를 연산한 후, 버림 
    - 성능 매우 나쁨
    - 따라서 설정값으로 클러스터에 지나친 부하를 주지 않도록 제한하고 있음 
- Scroll API: Scroll ID가 발급될때마다 검색 컨텍스트가 메모리에 살아 있다. 
    - 빈번한 유저 요청에 사용할만한 기능이 못 됨 

**답은 서치 애프터 이다.**

```json
GET mall/_search
{
  "query" : {
    "match_all": {}
  },
  "sort": [
    {
      "mall_bno": {
        "order": "desc"
      }
    }
  ]
}
```

**Response**

```json
  "regm_stat_cd": "M17002",
          "mall_url": "https://smartstore.naver.com/globalshoppings",
          "mls_mall_yn": "N",
          "npay_yn": "Y",
          "mall_bno": "c3930227",
          "@timestamp": "2019-02-25T01:21:50.274Z",
          "mall_id": "globalshoppingmall"
        },
        "sort": [
          "c3930227"
        ]
```

- ```sort``` : 다음 search_after 요청에서 사용할 정렬 키 값. 

**다음 요청**

```json
GET mall/_search
{
  "query" : {
    "match_all": {}
  },
  "search_after" : ["c3930227"],
  "sort": [
    {
      "mall_bno": {
        "order": "desc"
      }
    }
  ]
}
```

### 실제 사용해본 경험

- ****의 페이지네이션에 적용해보려고 시도함.
- 좋음.
- 코드가 매우 지저분해짐 
- 이걸 나이스하게 추상화해서 풀어보고 싶다.
    - elasticsearch 프로젝트 같이 하실 분 모집. 



