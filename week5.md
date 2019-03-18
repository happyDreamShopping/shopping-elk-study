## 05 검색 - 2일차 

## 목차

- 결과의 inner hits 반환
- 올바른 쿼리 제안
- 매치된 결과 카운트
- explain 쿼리
- ```match_all```, ```boolean``` 

### 결과의 inner hits 반환 

- ES는 nested doc을 사용할 수 있음
- 그러나, 쿼리를 전송했을 때 nested doc에 대한 검색 결과는 제공되지 않음 
- ```parent-child```, ```nested``` 모델을 검색하는 데 유용함 
    - ```parent-join``` 모델은 deprecated
    - ```join``` 모델의 쿼리를 위해 사용
- 실무적으로 join을 써본 적이 없어서 잘 모르겠습니다.


### 올바른 쿼리 제안 

- 잘못된 검색어를 사용자가 입력할 경우에 대비해 "제안" 기능 제공


**예시**
```json
GET mall/_search
{
  "suggest": {
    "suggest1": {
      "text": "11sk",
      "term": {
        "field": "mall_nm"
      }
    }
  }
}
```

**결과**
```json
{
  "took": 195,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 0,
    "max_score": 0,
    "hits": []
  },
  "suggest": {
    "suggest1": [
      {
        "text": "1",
        "offset": 0,
        "length": 1,
        "options": []
      },
      {
        "text": "11",
        "offset": 0,
        "length": 2,
        "options": []
      },
      {
        "text": "11s",
        "offset": 0,
        "length": 3,
        "options": []
      },
      {
        "text": "11sk",
        "offset": 0,
        "length": 4,
        "options": [
          {
            "text": "11st",
            "score": 0.75,
            "freq": 2
          },
          {
            "text": "119k",
            "score": 0.75,
            "freq": 1
          },
          {
        ...
```

- CPS 몰 중 하나인 "11st" 를 상정하고 "11sk" 를 고의적으로 오입력함 
- 최상위 제안으로 "11st"를 제안해줌

#### 제안자(Suggester)

- term 제안자
    - 사용자가 입력한 텍스트와 해당 필드만 있으면 된다.

- phrase 제안자 


### 매치된 결과 카운트

- 검색 결과로 받은 document의 결과를 센다. 
- 카운트 작업은 해당 index의 모든 shard에서 분산작업 후 집계한다. 

**예시**

```json
GET mall/_count
{
  "query": {
    "match_all": {}
  }
}
```

**응답**
```json
{
  "count": 472840,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  }
}
```

#### 예상되는 사용 케이스 

- 검색 결과에 대한 집계는 필요없고, 단순히 검색 결과의 갯수만을 리턴받고 싶을 때
    - 일래스틱서치 클러스터가 검색 결과를 연산하고, 각 스코어를 계산하는 것은 꽤 비용이 많이 드는 작업이다.
    - 검색결과를 받기 전 단순히 해당 검색에 해당하는 문서의 갯수만을 보고싶은 이슈가 있을 수 있다. 

#### 내부적인 동작 원리 

- ```_search``` 와 같게 작동한다.
    - ```from ~ size``` 에서 사용하던 ```size``` 옵션을 0으로 주고 검색하는 것이다. 

### explain 쿼리

- 원하는 검색결과가 나오지 않았을 때, 어떤 연산을 거쳐 이런 검색결과가 나왔는지를 분석해볼 수 있다. 
- index 이름, doctype, doc ID를 알아야 쿼리 가능.

**예시**
```json
GET mall/doc/212021/_explain
{
  "query": {
    "term" : {
      "mall_nm" : "**몰"
    } 
  }
}
```

- mall 인덱스에서
- "doc" doctype을 가진
- 212021번 도큐먼트를
- _explain 해다오.

**212021번 도큐먼트는 어떻게 생겼을까요**

![사내망에서 확인가능](https://media.oss.navercorp.com/user/10142/files/eb149c54-49b7-11e9-94dc-f9022657a566)

**쿼리 수행 결과**
```json
{
  "_index": "mall",
  "_type": "doc",
  "_id": "212021",
  "matched": true,
  "explanation": {
    "value": 15.973032,
    "description": "weight(mall_nm:**몰 in 3) [PerFieldSimilarity], result of:",
    "details": [
      {
        "value": 15.973032,
        "description": "score(doc=3,freq=1.0 = termFreq=1.0\n), product of:",
        "details": [
          {
            "value": 11.054419,
            "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:",
            "details": [
              {
                "value": 1,
                "description": "docFreq",
                "details": []
              },
              {
                "value": 94833,
                "description": "docCount",
                "details": []
              }
            ]
          },
          {
            "value": 1.4449455,
            "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:",
            "details": [
              {
                "value": 1,
                "description": "termFreq=1.0",
                "details": []
              },
              {
                "value": 1.2,
                "description": "parameter k1",
                "details": []
              },
              {
                "value": 0.75,
                "description": "parameter b",
                "details": []
              },
              {
                "value": 24.264349,
                "description": "avgFieldLength",
                "details": []
              },
              {
                "value": 6,
                "description": "fieldLength",
                "details": []
              }
            ]
          }
        ]
      }
    ]
  }
}
```

- matched: 도큐먼트가 쿼리에 매치되었는지.
    - 지금 쿼리가 (term: mall_nm = "켄즈몰") 도큐먼트의 내용과 정확히 일치.
    - 따라서 true

- explanation
    - value: 쿼리 섹션의 점수 
    - description: 매치된 토큰의 문자열 표현
        - ```"weight(mall_nm:**몰 in 3) [PerFieldSimilarity], result of:"```
    - details: explanation 객체의 선택사항 필드

    