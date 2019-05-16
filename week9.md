# 10 - 클러스터와 노드 관리

> - 노드와 클러스터를 모니터링하여 성능과 상태를 관리하고 개선하는것이 중요
> - 클러스터 수준에서 발생할 수 있는 문제점
>     - 노드 오버헤드 - 일부 노드에 너무 많은 샤드가 할당되어 클러스터의 병목 발생
>     - 노드 셧다운 - 디스크 풀, 하드웨어 장애, 전원 문제 등
>     - 샤드 재배치/변형 - 샤드가 온라인 상태가 아닌 경우
>     - 빈 색인/빈 샤드 - 샤드에서 활성 스레드가 있어 빈 색인이나 샤드가 많은 경우 클러스터의 성능 저하

## [0] 목차

> 책 타이틀 - 공식 문서 타이틀

1. Cluster APIs
    - API를 통한 클러스터 헬스 제어 - [Cluster Health](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-health.html)
    - API를 통한 클러스터 상태 제어 - [Cluster State](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-state.html) 
    - API를 통한 클러스터 노드 정보 얻기 - [Nodes Info](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-info.htmlv) 
    - API를 통한 노드 통계 얻기 - [Nodes Stats](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-stats.html)
    - 작업 관리 API 사용 - [Task Management API](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/tasks.html)
    - 핫 스레드 API - [Nodes hot_threads](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-hot-threads.html) 
    - 샤드 할당 관리 - [Cluster Allocation Explain API](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-allocation-explain.html)
2. Indices APIs
    + 세그먼트 API로 세그먼트 모니터링 - [Indices Segments](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-segments.html#indices-segments)
    + 캐시 정리 - [Clear Cache](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-clearcache.html)

## [1] Cluster APIs

### [1. Cluster Health](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-health.html)

**클러스터 상태 관리**

```bash
GET _cluster/health

# 반환 결과
{
  "cluster_name" : "testcluster",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 1,
  "active_shards" : 1,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 1, 
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 0,
  "active_shards_percent_as_number": 50.0
}
```

- status: 클러스터의 상태 
    - green: 정상
    - yellow: 활성 샤드의 사본이 하나씩은 있어서 클러스터 기능에는 문제가 없지만, 복제본이 없는 경우
    - red: primary shard가 누락되어 index를 쓸 수 없는 경우
- unassigned_shards: 노드에 할당되지 않은 샤드의 수
- delayed_unassigned_shards: 할당될 샤드의 수

**클러스터에서 index의 상태 관리**

```bash
GET /_cluster/health/test1,test2
```

**추가 요청 파라미터**

```bash
GET /_cluster/health?wait_for_status=yellow&timeout=50s
```

- level: 반환된 헬스 정보의 상세 레벨 관리
- wait_for_status: 제공된 상태(green, yellow, red)가 될 때까지 기다리는 것을 허용
- wait_for_relocating_shjards: 서버가 재배치된 샤드의 개수에 도달할 때까지 기다리는 것을 혀용
- wait_for_nodes: ㅋ클러스터에서 정의한 수의 노드를 사용할 수 있을 때까지 대기하는 것을 허용
    - `>N, >=N, <N, <=N, ge(N), gt(N), le(N), lt(N)
- timeout: wait_for_* 파라미터의 대기 시간 설정

**보류 중인 작업 확인하기**

```bash
GET /_cluster/pending_tasks
```

### [2. Cluster State](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-state.html)

**클러스터 상태에 대한 상세 정보 얻기**

```bash
GET /_cluster/state

# 반환 결과
{
  "cluster_name" : "dev-sa-elk",
  "cluster_uuid" : "3reD02rVQ1Sqb0PzbkOMVA",
  "version" : 178,
  "state_uuid" : "ycscl9ZaQYam4sJcfCdjfA",
  "master_node" : "gnlbnhQ2RhODKPqcn3ceZw",
  "blocks" : { },
  "nodes" : { ... },
  "metadata" : {
      "cluster_uuid" :  "3reD02rVQ1Sqb0PzbkOMVA",
      "cluster_coordination" : { ... },
      "templates" : { ... },
      "indices" : { ... },
      "index-graveyard" : { ... },
      "index_lifecycle" : { ... },
      "ingest" : { ... }
  },
  "routing_table" : { ... },
  "routing_nodes" : { ... }
}
```

- 일반 클러스터 정보
- 노드 주소 정보
- 클러스터 메타데이터 정보
    - templates: 생성한 index의 동적 매핑을 제어하는 템플릿
    - indices: 클러스터에 존재하는 index의 정보
        - state: index가 열려있는지 닫혀있는지를 나타냄
        - setting: 색인 설정
            - index.number_of_replicas: index의 복제본 수(index 업데이트로 변경 가능)
            - index.number_of_shards: index의 shard 수(색인에서 변경 불가능)
            - index.codec: index 데이터를 저장하는 데 사용하는 코덱(기본은 LZ4, 압축률을 높이려면 best_compression과 DEFLATE 사용 - 쓰기 성능이 약간 느려짐)
            - index.version.created: 색인 버전
    - ingest: 시스템에 정의한 모든 인제스트 파이프라인
    - mappings: `‼️Deprecated`
    - alias: index의 앨리어스 목록
- 라우팅 테이블 정보
- 라우팅 노드 정보

**클러스터 상태의 일부분만 요청하기**

```bash
GET /_cluster/state/{metrics}
GET /_cluster/state/{metrics}/{indices}
```

- {metrics}는 다음 옵션의 쉼표로 구분된 목록.
- version: 클러스터 상태 버전을 표시.
- master_node: 응답의 선택된 master_node 부분을 표시
- nodes: 응답의 nodes 부분을 표시.
- routing_table: 응답의 routing_table 부분을 표시. 쉼표로 구분된 index 목록을 제공하는 경우 반환된 결과에는 이러한 index에 대한 라우팅 테이블만 포함된다.
- metadata: 응답의 metadata 부분을 표시. 쉼표로 구분된 index 목록을 제공하는 경우 반환된 결과에는 이러한 index 대한 메타데이터만 포함된다.
- blocks: 응답의 blocks 부분을 표시.
- _all: 모든 metrics를 표시.

`foo` 및 `bar` index에 대한 메타데이터 및 라우팅_테이블 데이터만 반환하기.

```bash
GET /_cluster/state/metadata,routing_table/foo,bar
```

`foo` 및 `bar` index에 대한 모든 것을 반환하기.

```bash
GET /_cluster/state/_all/foo,bar
```

블록 메타데이터만 반환하기.

```bash
GET /_cluster/state/blocks
```

### [3. Nodes Info](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-info.htmlv)

**노드 수준의 상태 정보 얻기** 

```bash
GET /_nodes
GET /_nodes/nodeId1,nodeId2
```

- host: 호스트의 이름
- ip: 호스트의 IP
- total_indexing_in_bytes: 디스크에 쓰기 전에 최근에 인덱싱된 문서를 보관하는 데 사용할 수 있는 총 힙.
- transport_address: 클러스터 통신을 위해 노드에서 사용하는 주소
- version: 노드에서 실행중인 elasticsearch 버전

**섹션을 필터링하여 정보 얻기**

- setting, os, process, jvm, thread_pool, transport, http, plugins, ingest, indices 에서 선택 가능

```bash
# return just process
GET /_nodes/process

# same as above
GET /_nodes/_all/process

# return just jvm and process of only nodeId1 and nodeId2
GET /_nodes/nodeId1,nodeId2/jvm,process

# same as above
GET /_nodes/nodeId1,nodeId2/info/jvm,process

# return all the information of only nodeId1 and nodeId2
GET /_nodes/nodeId1,nodeId2/_all
```

### [4. Nodes Stats](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-stats.html)

**노드 통계 수집하기**

```bash
GET /_nodes/stats
GET /_nodes/nodeId1,nodeId2/stats
```

- cluster_name: 클러스터 이름과 노드 섹션을 설명하는 헤더
- indices: 색인과 관련된 통계, 필드와 캐시의 사용량 및 get/indexing/flush/merge/refresh/warmer와 같은 연산에 대한 통계
- os: 운영체제와 관련된 통계, CPU 사용량, 노드의 로드, 메모리와 스왑, 가동 시간에 관한 통계
- process: elasticsearch 프로세스와 관련된 통계, ES가 사용하는 메모리, 열린 파일 디스크립터에 대한 통계
- jvm: jvm과 관련된 통계, 버퍼/풀/가비지 컬렉터/메모리/스레드/가동 시간에 대한 통계
- thread_pool: 스레드 풀과 관련된 통계, 사용 가능한 모든 스레드 풀을 모니터한 통계
- fs: 노드의 파일 시스템 통계, 장비의 여유 공간, 마운트 지점, 읽기 및 쓰기 등
- transport: 노드 간 통신 관련 통계
- http: HTTP 연결과 관련된 통계, 현재 열려 있는 소켓의 수와 최대 수
- breakers: 브레이커 캐시와 관련된 통계, 써킷 브레이커를 모니터함
- script: 통계 관련 스크립트
- discovery: 클러스터 상태 큐
- ingest: 인제스트 통계

**섹션을 필터링하여 통계 수집하기**

- indices, fs, http, jvm, os, process, thread_pool, transport, breaker, discovery, ingest 에서 선택 가능

```bash
# return just indices
GET /_nodes/stats/indices

# return just os and process
GET /_nodes/stats/os,process

# return just process for node with IP address 10.0.0.1
GET /_nodes/10.0.0.1/stats/process
```

### [5. Task Management API](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/tasks.html)

**작업 정보 얻기**

```bash
GET _tasks 
GET _tasks?nodes=nodeId1,nodeId2 
GET _tasks?nodes=nodeId1,nodeId2&actions=cluster:*

# 반환 결과
{
  "nodes" : {
    "oTUltX4IQMOUUVeiohTt8A" : {
      "name" : "H5dfFeA",
      "transport_address" : "127.0.0.1:9300",
      "host" : "127.0.0.1",
      "ip" : "127.0.0.1:9300",
      "tasks" : {
        "oTUltX4IQMOUUVeiohTt8A:124" : {
          "node" : "oTUltX4IQMOUUVeiohTt8A",
          "id" : 124,
          "type" : "direct",
          "action" : "cluster:monitor/tasks/lists[n]",
          "start_time_in_millis" : 1458585884904,
          "running_time_in_nanos" : 47402,
          "cancellable" : false,
          "parent_task_id" : "oTUltX4IQMOUUVeiohTt8A:123"
        },
        "oTUltX4IQMOUUVeiohTt8A:123" : {
          "node" : "oTUltX4IQMOUUVeiohTt8A",
          "id" : 123,
          "type" : "transport",
          "action" : "cluster:monitor/tasks/lists",
          "start_time_in_millis" : 1458585884904,
          "running_time_in_nanos" : 236042,
          "cancellable" : false
        }
      }
    }
  }
} 
```

- node: 작업을 실행하는 노드를 정의
- id: 작업의 고유 ID를 정의
- action: 액션의 이름 "액션 타입:세부 액션"으로 구성됨
- cancellable: 작업을 취소할 수 있는지 정의
    - 쿼리로 삭제, 업데이트, 재색인 작업은 취소할 수 있음
- parent_task_id: 태스크 그룹을 정의. 일부 작업을 여러 하위 작업으로 나눠서 실행할 수 있음.

작업의 id를 사용해서 응답을 필터링하기

```bash
GET _tasks/oTUltX4IQMOUUVeiohTt8A:124
```

작업 그룹의 parent_task_id를 사용해서 필터링하기

```bash
GET _tasks?parent_task_id=oTUltX4IQMOUUVeiohTt8A:123
```

**작업 취소하기**

```bash
POST _tasks/oTUltX4IQMOUUVeiohTt8A:12345/_cancel
```

- 작업을 취소하면 도큐먼트를 부분적으로업데이트하거나 삭제하기 때문에 데이터 불매치가 발생할 수 있음
- 하지만 재색인할때는 취소하는것이 합리적일 수 있

작업 그룹의 경우 쿼리 인자를 사용해 취소하기

```bash
POST _tasks/_cancel?nodes=nodeId1,nodeId2&actions=*reindex
```

### [6. Nodes hot_threads](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-nodes-hot-threads.html)

**핫 스레드 모니터링하기**

```bash
GET /_nodes/hot_threads
GET /_nodes/nodeId1,nodeId2/hot_threads
```

- 단일 스레드의 속도 저하 원인을 확인할 수 있음
- 추가 요청 파라미터
    - threads: 제공할 핫 스레드의 수
    - interval: 스레드 샘플링 간격
    - type: 다른 타입의 핫 스레드를 제어(cpu/wait/block)
    - ignore_idle_threads: 유휴 스레드를 필터링하는데 사용

### [7. Cluster Allocation Explain API](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cluster-allocation-explain.html)

**샤드 할당 정보 얻기**

```bash
GET /_cluster/allocation/explain
{
  "index": "myindex",
  "shard": 0,
  "primary": true
}

# 반환 결과
{
  "index" : "idx",
  "shard" : 0,
  "primary" : true,
  "current_state" : "unassigned",                 
  "unassigned_info" : {
    "reason" : "INDEX_CREATED",                   
    "at" : "2017-01-04T18:08:16.600Z",
    "last_allocation_status" : "no"
  },
  "can_allocate" : "no",                          
  "allocate_explanation" : "cannot allocate because allocation is not permitted to any of the nodes",
  "node_allocation_decisions" : [
    {
      "node_id" : "8qt2rY-pT6KNZB3-hGfLnw",
      "node_name" : "node-0",
      "transport_address" : "127.0.0.1:9401",
      "node_attributes" : {},
      "node_decision" : "no",                     
      "weight_ranking" : 1,
      "deciders" : [
        {
          "decider" : "filter",                   
          "decision" : "NO",
          "explanation" : "node does not match index setting [index.routing.allocation.include] filters [_name:\"non_existent_node\"]"  
        }
      ]
    }
  ]
}
```

- 샤드가 노드에 할당되지 않는 경우 할당하지 않은 이유를 조사
- deciders에서 샤드를 할당할수 없는 이유를 찾을 수 있음
- body에 사용된 파라미터
    - index: 샤드가 속한 색인
    - shard: 샤드의 숫자(0부터 시작)
    - primary: 검사할 샤드가 기본 샤드인지 여부 확인

## [2] Indices APIs

### [1. Indices Segments](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-segments.html#indices-segments)

**index 세그먼트에 대한 정보 얻기**

```bash
GET /test/_segments

# 반환 결과
{
  "_shards": ...
  "indices": {
    "test": {
      "shards": {
        "0": [
          {
            "routing": {
              "state": "STARTED",
              "primary": true,
              "node": "zDC_RorJQCao9xf9pg3Fvw"
            },
            "num_committed_segments": 0,
            "num_search_segments": 1,
            "segments": {
              "_0": {
                "generation": 0,
                "num_docs": 1,
                "deleted_docs": 0,
                "size_in_bytes": 3800,
                "memory_in_bytes": 1410,
                "committed": false,
                "search": true,
                "version": "7.0.0",
                "compound": true,
                "attributes": {
                }
              }
            }
          }
        ]
      }
    }
  }
}
```

- num_docs: 색인에 저장된 도큐먼트의 수
- deleted_docs: 색인에서 삭제된 도큐먼트의 수. 이 값이 크면 삭제 표시에 많은 공간이 낭비됨
- size_in_bytes: 세그먼트의 바이트 크기. 이 값이 크면 쓰기 속도가 느려짐
- memory_in_bytes: 세그먼트가 차지하는 바이트 단위의 메모리
- committed: 세그먼트가 디스크에 커밋됐는지 여부
- search: 세그먼트가 검색에 사용되는지 여부
- version: 색인 생성에 사용되는 루씬 버전
- compound: 색인이 복합 색인인지 여부

### [2. Clear Cache](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/indices-clearcache.html)

**캐시 지우기**

```bash
POST /twitter/_cache/clear

# 반환 결과
{
    "_shards" : {
        "total" : 10,
        "successful" : 5,
        "failed" : 0
    }
}
```

- ES는 검색 속도를 높이기 위해 캐시 결과, 항목, 필터 결과를 캐시함.
- 메모리를 확보하기 위해서 캐시 API를 사용
- ES가 내부적으로 캐시를 관리하고 정리하지만, 노드의 메모리가 부족할 경우 편리함