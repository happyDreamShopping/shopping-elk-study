# 04 기본작업

## 목차
### 1회차
01. 인덱스 생성
02. 인덱스 삭제
03. 인덱스 open & close
04. 인덱스에 mappings 입력
05. Mappings 조회
06. Reindex
07. 인덱스 refresh
08. 인덱스 flush
09. 인덱스 force merge
10. 인덱스 shrink

### 2회차
11. 인덱스 및 타입 존재 여부 확인
12. 인덱스 settings 관리
13. 인덱스 alias
14. 인덱스 roll over
15. Document 색인
16. Document 조회
17. Document 삭제
18. Document 업데이트
19. Bulk API (원자성 작업 속도 향상)
20. MultiGet API (GET 작업 속도 향상)

<br>

---

<br>

## 00. 들어가기 앞서
* ES 의 인덱스(index; 색인)는 RDB 의 데이터베이스와 유사.
* ES 의 타입(type) 은 RDB 의 테이블과 유사한 개념으로 쓰였으나, 여러 문제점으로 인해 기본 타입인 `_doc` 을 제외하곤 deprecated 됨. ([참고](https://github.com/occidere/notepad/issues/65))
* ES 의 도큐먼트(document) 는 RDB 의 레코드와 유사.

<br>

## 01. 인덱스 생성

말 그대로 인덱스를 생성한다. 인덱스 생성 시 mappings 및 replica 수, shards 수 등의 각종 settings 들을 사전 정의할 수 있다.

단, 이는 말 그대로 인덱스만 생성하는 것이며 document 를 생성하는 것은 아니다.

동일한 이름의 인덱스가 이미 존재한다면 400 에러 코드를 반환할 것이며, 성공적으로 생성했다면 200 을 반환한다.

> document 생성 시에는 지정한 인덱스가 없으면 해당 인덱스를 기본 세팅 값을 바탕으로 자동 생성해준다

<br>

인덱스 이름으로 지정할 수 있는 값들은 아래와 같다.
* ASCII 문자 [a-z]
* 숫자 [0-9]
* `.`, `-`, `&`, `_`

> 일반적으로 시스템 내부 용도로 사용되는 인덱스는 맨 앞에 `.` 을 붙인다. ex) .kibana-1

> 시간성을 띄는 시계열 데이터인 경우 인덱스 이름에 시간 값을 표현해 주는 것을 권장한다. ex) shopping_products-190303

<br>

인덱스 생성 예시는 아래와 같다.
````js
PUT shopping_products-190303
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  },
  "mappings": {
    "_doc": {
      "properties": {
        "product_id": {
          "type": "keyword",
          "store": true
        },
        "name": {
          "type": "text",
          "store": true
        }
      }
    }
  }
}
````

* **number_of_shards**: 색인 시 생성할 샤드 개수. 기본값은 5이다.
ES 의 Shard 는 1개의 Lucene Index 인데, LUCENE-5483 에 의하면 1개의 Lucene index 는 최대 약 21억개 (`Integer.MAX_VALUE - 128`)의 문서를 저장할 수 8있다 ([참고](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/_basic_concepts.html#getting-started-shards-and-replicas))
* **number_of_replicas**: 복제본 개수. 기본값은 1이며, 최소 1개 이상을 권장한다.

<br>

## 02. 인덱스 삭제
말 그대로 인덱스를 삭제한다. **한번 삭제한 인덱스는 절대 복구할 수 없으므로 반드시 주의**를 기울이도록 한다.

<br>

인덱스 삭제 예시는 아래와 같다.
````js
DELETE shopping_products-*,shopping_log-190303
````

또는 인덱스 명 대신 `_all` 을 지정하여 모든 인덱스를 제거할 수 있다.
````js
DELETE _all
````

그러나 그런 경우는 거의 없기 때문에 실수를 방지하기 위해 elasticsearch.yml 에 아래 값을 추가하여 해당 옵션을 비활성화 할 수도 있다.
````yml
action.destructive_requires_name: true
````

<br>

## 03. 인덱스 open & close
ES 의 인덱스는 검색 또는 쓰기를 하던 안하던 간에 일정량의 cpu, memory 등의 자원이 사용된다.

따라서 사용되지 않는 인덱스를 삭제하지는 않으면서 자원할당을 안 하기 위해 close 를 할 수 있다.
````js
POST shopping_logs-180903/_close
````
> Cerebro 를 사용한다면 클릭으로 간단히 닫을수 있다.

![image](https://user-images.githubusercontent.com/20942871/53694386-673d6400-3df1-11e9-9f87-25ecdfba51e4.png)

> 단, Cerebro 또는 Kibana 에선 인덱스가 close 되면 해당 용량 만큼이 줄은 것으로 표시되지만, 실제로는 그대로 자리를 차지하고 있는 것임에 주의한다!

<br>

반대로 닫힌 인덱스를 열고 싶으면 open 을 호출하면 된다.
````js
POST shopping*/_open
````

<br>

## 04. 인덱스에 mappings 입력
이미 존재하는 인덱스에 mappings 를 생성할 때 사용한다. 

단, 지정한 필드의 매핑이 이미 있는 경우라면 에러가 발생하며, `ignore_unavailable=true` 로 에러 발생한 필드를 무시하고 건너뛸 수 있다.

ex) `name: keyword` 로 이미 존재하는 상황에서 `name: text` 로 매핑을 지정하려 하면 에러가 발생하나, `ignore_unavailable=true` 를 추가해 요청을 날리면 에러가 발생한 name 필드는 무시하고 진행한다.

또한, 기존에 없던 새로운 필드의 매핑이 정의된 경우 기존 매핑값들은 유지한 상태로 새로 들어온 필드의 매핑값을 추가한다.

ex) 기존엔 `product_id` 필드만 있는 매핑에 `name` 이란 필드의 매핑이 추가되면, 새 매핑은 `product_id`, `name` 필드들을 설정해 놓은 상태가 된다.

<br>

매핑 생성 예시는 아래와 같다.
````js
PUT shopping_products-190303/_doc/_mapping?ignore_unavailable=true
{
  "_doc": {
    "properties": {
      "product_id": {
        "type": "keyword",
        "store": true
      },
      "name": {
        "type": "text",
        "store": true
      }
    }
  }
}
````

그러나 이는 어디까지나 신규 매핑을 추가하는 경우이며, **이미 생성된 필드의 매핑은 삭제하거나 변경할 수 없다.**

<br>

## 05. 매핑 조회
말 그대로 인덱스들의 매핑 현황을 조회한다.

````js
GET */_mapping
GET shopping_products-*/_mapping
GET shopping_logs-190303/_doc/_mapping
````

<br>

## 06. Reindex

기존의 인덱스를 다시 생성해서 저장할 때 사용한다.

reindex 가 필요한 대표적인 상황은 아래와 같다.
1. 매핑의 삭제 또는 변경이 필요한 경우
(매핑만 별도로 변경이 불가능하므로 매핑이 새롭게 정의된 인덱스를 새로 생성하는 방식을 적용)
2. 매핑의 analyzer 변경
(형태소 분석기 등)
3. 다른 클러스터의 인덱스를 가져와야 하는 경우
(주로 클러스터 이전 시 사용)

Local -> Local 및 Remote -> Local 상황으로 나눠 예시를 살펴보면 아래와 같다.

<br>

### Local -> Local Reindex 예시
````js
POST _reindex?wait_for_completion=false
{
  "source": {
    "index": "latency"
  },
  "dest": {
    "index": "recent_status"
  },
  "script": {
    "source": "ctx._source.postDate = ctx._source['@timestamp']; ctx._source.remove(\"@timestamp\");",
    "lang": "painless"
  }
}
````
* 기존의 latency 라는 인덱스를 recent_status 라는 이름의 인덱스로 새로 생성한다.
* painless 스크립트를 이용해 latency 의 `@timestamp` 필드의 값을 `postDate` 라는 필드를 새로 생성해 저장하고, 기존 `@timestamp` 필드는 삭제한다.
* `wait_for_completion=false` 로 지정하여 백그라운드에서 async 로 작업이 이뤄지게 한다.
이렇게 호출하면 Task 의 id 값이 반환되며, Task API 로 작업 상태를 조회할 수 있다.
만약 해당 값을 true 로 지정하면 완료될 때 까지 대기한다.

<br>

### Remote -> Local Reindex 예시
````js
POST _reindex?wait_for_completion=false
{
  "source": {
    "remote": {
      "host": "http://dev-occidere001-ncl:9200",
      "socket_timeout": "1m",
      "connect_timeout": "10s"
    },
    "index": "shopping_logs-180313",
    "size": 50000,
    "query": {
      "bool": {
        "filter": {
          "range": {
            "@timestamp": {
              "gte": "2018-03-13 12:00:00,000",
              "lte": "2018-03-13 23:59:59,999",
              "format": "yyyy-MM-dd HH:mm:ss,SSS",
              "time_zone": "+09:00"
            }
          }
        }
      }
    }
  },
  "dest": {
    "index": "shopping_logs_afternoon-180313"
  }
}
````
* dev-occidere001-ncl 클러스터에 있는 shopping_logs-180313 인덱스 중, KST 시간 기준 오후의 데이터만 현재 클러스터에 shopping_logs_afternoon-180313 라는 이름의 인덱스로 한번에 5만개 씩 가져온다.
* Remote -> Local 을 사용하려면 Local 의 elasticsearch.yml 의 whitelist 에 remote host 가 저장되어 있어야 한다.
    ````yml
    reindex.remote.whitelist: "otherhost:9200, another:9200, 127.0.10.*:9200, localhost:*"
    ````

<br>

이외의 각종 옵션 값은 [Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) 를 참고한다.

<br>

## 07. 인덱스 refresh

ES 에서 문서를 색인 한 뒤 검색을 하려면 반드시 refresh 를 해야 한다. 기본값으로 1초마다 자동 refresh 를 수행하나, reindex 를 하는 상황 같은 특수한 경우엔 수동으로 refresh 가 필요할 때도 있다.

refresh 의 예제는 아래와 같다.
````js
POST shopping_products-*/_refresh
POST _refresh
````

<br>

## 08. 인덱스 flush
ES 는 쓰기 작업의 발생 횟수를 줄여 I/O 오버헤드를 줄이고, 성능 향상을 위해 특정 데이터를 메모리와 트랜잭션 로그에 캐싱한 뒤 flush 를 하여 디스크에 쓴다.

이 때, 가용 메모리 확보하고 트랜잭션 로그를 비우고 데이터를 디스크에 안전히 쓰기 위해 인덱스 flush 가 필요하다.

원래 주기적으로 디스크에 flush 를 하나, 아래와 같은 이유로 강제 flush 가 필요할 수도 있다.
* 노드 종료 시 stale data 발생 방지
* 모든 데이터를 안전한 상태로 유지하기 위해
ex) bulk indexing 후 모든 데이터를 flush 후 refresh 함

flush 의 예제는 아래와 같다.
````js
POST shopping_products-190303/_flush
````

단, 빈번한 flush 는 성능 하락으로 이어짐을 유의한다.

<br>

## 09. 인덱스 force merge
ES 의 document 는 내부적으로 Lucene Segments 로 쪼개져 저장된다.

세그먼트 개수가 많을수록 문서 검색 시 읽어야 하는 세그먼트가 많으므로 검색 속도가 감소한다.

또한 ES 의 문서 삭제는 진짜 디스크에서 제거하는 것이아닌 삭제 표시만 하는 것인데, 가용 공간을 늘리려면 force merge 를 통해 삭제된 document 를 제거해야 한다.

이러한 force merge 는 미사용 세그먼트 및 삭제된 도큐먼트를 제거하며, 더 적은 수의 세그먼트로 합치는 I/O 부하가 높은 작업이다.

force merge 시의 장점은 아래와 같다.
* 파일 디스크립터 사용 절감
* 세그먼트 reader 가 사용하는 메모리 확보
* 적은 수의 세그먼트 관리로 검색 성능 향상

> force merge 를 수행하는 동안 제대로 검색이 안될 수 있으며, 작업 중 일시적으로 용량이 증가할 수 있고 시간이 오래 소요된다.

> 가급적 변동사항이 없는 과거 데이터들에 적용하는 것을 강력히 권장한다.

<br>

force merge 의 예시는 아래와 같다.
````js
POST shopping_products-190303/_forcemerge?max_num_segments=1
````
* max_num_segments: 기본 값은 autodetect 이며, 병합 시 생성될 최대 세그먼트 수를 지정한다. 전체 최적화의 경우 이 값을 1로 설정한다.
* only_expunge_deletes: 기본값은 false 이며, 이 값을 true 로 하면 삭제 처리된 document 의 segments 들만 병합한다.
* flush: 기본값은 true 이다.

<br>

## 10. 색인 shrink
force merge 가 루씬 세그먼트 수를 줄이는 것이였다면, shrink 는 ES 샤드 수를 줄이는 것이다.

일반적으로 샤드 수와 검색 속도는 반비례하고, 쓰기 속도와는 정비례 한다.

shrink api 를 사용하기 위해선 아래의 조건이 충족되어야 한다.
* 전체 primary shards 는 동일한 node 에 위치해야 한다.
* 사본 인덱스는 없어야 된다.
* 사본의 shards 수는 원본 인덱스 shards 수의 배수가 되야 한다.
* 인덱스가 읽기전용 상태여야 한다.

<br>

작업 과정은 아래와 같다.
### 1. 작업 대상 인덱스를 특정 노드에 위치시킨다.
````js
PUT test-190214/_settings
{
  "settings": {
    "index.routing.allocation.require._name": "elasticsearch_data-1",
    "index.blocks.write": true
  }
}
````
![image](https://user-images.githubusercontent.com/20942871/53695360-f2bcf200-3dfd-11e9-809b-cfa80c9bbbe6.png)


### 2. shrink api 를 호출한다.
````js
POST test-190214/_shrink/reduced_test-190214
{
  "settings": {
    "index.routing.allocation.require._name": null, 
    "index.blocks.write": false,
    "index.number_of_replicas": 1,
    "index.number_of_shards": 1,
    "index.codec": "best_compression"
  },
  "aliases": {
    "test_alias": {}
  }
}
````
![image](https://user-images.githubusercontent.com/20942871/53695466-7d522100-3dff-11e9-8ab8-5593947c72e7.png)
* 이 과정에서 ES 는 원본 인덱스에서 사본 인덱스로 세그먼트를 하드 링크한다.
리눅스는 하드링크를 기본적으로 지원하나, 그렇지 않은 경우 모든 세그먼트를 직접 복사해야 하므로 시간이 더 걸린다.


### 3. 기존의 원본 인덱스를 원상복구시킨다.
````js
PUT test-190214/_settings
{
  "settings": {
    "index.routing.allocation.require._name": null,
    "index.blocks.write": false
  }
}
````
![image](https://user-images.githubusercontent.com/20942871/53695479-8b07a680-3dff-11e9-9cd8-c9a16ba9b740.png)


