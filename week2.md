# 매핑 관리

매핑(Mapping)은 도큐먼트(document)와 필드들(fields)을 어떻게 저장, 색인, 처리 방식을 정의하는 과정이다. 
Elasticsearch는 문서 지향 엔진으로 명시적으로 스키마를 정의하지 않아도 된다(Schemaless). 
따라서 색인 시 매핑 타입이나 필드를 정의하지 않으면 기본 매핑을 생성하고, 데이터 필드에서 구조를 유추하여 도큐먼트를 작성한다.


`String`
* *keyword (String, VarChar), text (String, VarChar, Text)*

* 형태소 분석이 가능한 text와 형태소 분석이 불가능한 keyword 타입이 제공된다.
* `keyword` (not analyzed): 주로 필터링, 정렬, 집계를 위한 목적으로 사용되며, 정확한 값(exact value)으로만 검색이 가능하다.
  * emall address, hostname, status code, zip code, tag
     
* `text` (analyzed): Analyzer에 의해 여러 개의 term들의 리스트로 변형되어 인덱싱된다.
  * email body, product of description
  * `text` 필드에 다음의 parameter를 적용할 수 있다 : store, index, null_value, boost, search_analyzer, analyzer, include_in_all, norms, copy_to, ignore_above
```  
# text type, index_prefixes parameter mapping 예시
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "full_name": {
          "type":  "text",
          "index_prefixes" : {
            "min_chars" : 1,    
            "max_chars" : 10    
          }
        }
      }
    }
  }
}
```
* ref. <https://www.elastic.co/guide/en/elasticsearch/reference/6.4/text.html>



다음은 쇼핑몰의 주문서 레코드를 매핑하는 방법을 설명한 것이다.
```
"order" : {
    "id" : "ORD00001",
    "date" : "20190226",
    "customer_id" : "ian.nam.kr",
    "sent" : "N",
    "name" : "Clova Speaker",
    "quantity" : 1,
    "vat" : 0.01
}
```

#### 기본타입 매핑
기본적으로 위 주문서 레코드를 Elasticsearch에 매핑하는 경우 아래와 같다. 
```
{
    "order" : {
        "properties" : {
            "id" : {"type" : "keyword"}
            "date" : {"type" : "date"}
            "customer_id" : {"type": "keyword", "store": "yes"}
            "sent" : {"type" : "boolean"}
            "name" : {"type": "text"}
            "quantity" : {"type" : "integer"}
            "vat" : {"type" : "double"}
            }

        }
    }
}
```

#### 오브젝트(Object) 매핑
Elasticsearch는 객체 타입을 파싱할 때 필드를 추출하고 정의된 매핑으로 처리해보고 
안되면 리플렉션을 사용해서 객체의 구조를 확인한다.
```
{
    "order" : {
        "properties" : {
            "id" : {"type" : "keyword"}
            "date" : {"type" : "date"}
            "customer_id" : {"type": "keyword", "store": "yes"}
            "sent" : {"type" : "boolean"}
            "item" : {
                "type" : "object",
                "properties" : {
                    "name" : {"type": "text"}
                    "quantity" : {"type" : "integer"}
                    "vat" : {"type" : "double"}
                }
            }

        }
    }
}
```
Object mapping은 부가적으로 하나 이상의 property tag를 적용할 수 있다.
* dynamic: true인 경우 동적으로 변경을 허용하며, 추가적인 성능상의 오버헤드가 발생하지 않는다. 
또한 한번 필드가 추가되면 타입은 변경될 수 없다.
* enabled: false인 경우 인덱스에서 배제한다.
* include_in_all: object 타입 수준에서 하위 매핑까지 인덱스를 적용한다.

#### 중첩(Nested) 매핑
```
"blogpost": {
     "properties": {
        "comments": {
          "type": "nested", 
          "properties": {
            "name":    { "type": "string"  },
            "comment": { "type": "string"  },
            "age":     { "type": "short"   },
            "stars":   { "type": "short"   },
            "date":    { "type": "date"    }
          }
        }
      }
}
```
* nested object 는 각각 별도의 문서로 취급되고 저장한다.
* nested object 의 개수가 많으면 이에 따른 resource가 많아지므로 
index.mapping.nested_field.limit을 통해 최대 개수를 정한다.

#### 동적 템플릿
* 매핑 타입을 미리 정의하지 않아도, 특정 규칙이나 패턴에따라 자동으로 매핑시킬 수 있다. 
```
"_doc": {
      "dynamic_templates": [
        {
          "integers": {
            "match_mapping_type": "long",
            "mapping": {
              "type": "integer"
            }
          }
        },
        {
          "strings": {
            "match_mapping_type": "string",
            "mapping": {
              "type": "text",
              "fields": {
                "raw": {
                  "type":  "keyword",
                  "ignore_above": 256
                }
              }
            }
          }
        }
      ]
    }
```
```
"_doc": {
      "dynamic_templates": [
        {
          "longs_as_strings": {
            "match_mapping_type": "string",
            "match":   "long_*",
            "unmatch": "*_text",
            "mapping": {
              "type": "long"
            }
          }
        }
      ]
    }
```

#### References
https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-templates.html
https://www.slideshare.net/dahlmoon/20160613
