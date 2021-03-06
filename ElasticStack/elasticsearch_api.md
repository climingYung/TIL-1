# Elasticsearch APIs

> 김종민 - Elastic 테크 에반젤리스트
>
> **Reference**
>
> - https://www.elastic.co/kr/webinars/getting-started-elasticsearch
> - http://kimjmin.net

## Index 생성

- library 인덱스 생성
  - shard 수 : 5
  - replica 수 : 1

```json
PUT library
{
  "settings" : {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

## Bulk 색인

- 다량의 도큐먼트를 한꺼번에 색인 할 때는 bulk API를 사용

```json
POST library/books/_bulk
{"index":{"_id":1}}
{"title":"The quick brow fox", "price":5, "colors":["red", "green", "blue"]}
{"index":{"_id":2}}
{"title":"The quick brow fox jumps over the lazy dog", "price":15, "colors":["blue", "yellow"]}
{"index":{"_id":3}}
{"title":"The quick brow fox jumps over the quick dog", "price":8, "colors":["red", "blue"]}
{"index":{"_id":4}}
{"title":"brow fox brown dog", "price":2, "colors":["black", "yellow", "red", "blue"]}
{"index":{"_id":5}}
{"title":"Lazy dog", "price":9, "colors":["red", "blue", "green"]}
```

## 검색(_search)

### 전체 도큐먼트 검색

- 기본적으로 _search의 옵션을 주지 않으면 기본적으로 인덱스의 전체 도큐먼트를 검색한다.
- 이때는 score 검색을 하지 않아서, 모든 score는 1로 동등하다.

```json
GET library/_search
{
  "query": {
    "match_all": {}
  }
}
```

```json
GET library/_search
```

### 매핑 정보 검색

- RDB의 스키마에 해당하는 Elasticsearch의 매핑정보를 검색한다.
- 리턴되는 값을 참조하면
- library라는 인덱스 밑에 books라는 도큐먼트가 있고, 도큐먼트에는 3개의 필드 colors, price, title이 존재한다.
- 필드안에는 타입과 하위 필드인 keyword 타입이 있다. keyword 타입은 aggregation에 쓰이는 필드를 저장하고, 이 내용들은 분석을 하지 않는다. 즉, 검색어로 쪼개지 않고 저장한다.

```json
GET library/_mapping
```

```json
{
  "library": {
    "mappings": {
      "books": {
        "properties": {
          "colors": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          },
          "price": {
            "type": "long"
          },
          "title": {
            "type": "text",
            "fields": {
              "keyword": {
                "type": "keyword",
                "ignore_above": 256
              }
            }
          }
        }
      }
    }
  }
}
```

### 키워드가 포함된 도큐먼트 검색

- match query 사용해서 title필드에 fox가 포함된 도큐먼트들을  리턴

```json
GET library/_search
{
  "query": {
    "match": {
      "title": "fox"
    }
  }
}
```

### 복수의 키워드가 포함된 도큐먼트 검색

- match query 사용, " "(공백)으로 분리

```json
GET library/_search
{
  "query": {
    "match": {
      "title": "quick dog"
    }
  }
}
```

- 위의 검색 결과에서 중요한 점은, 검색의 정확도를 표현하는 **score 값이 리턴** 된다.
- 검색의 타겟인 quick과 dog이 많이 매칭 될수록 score 값이 높아진다.
  - [Elasticsearch에서 검색의 score(정확도)를 계산하는 방법]()

```json
{
  "took": 64,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 5,
    "max_score": 0.7564473,
    "hits": [
      {
        "_index": "library",
        "_type": "books",
        "_id": "2",
        "_score": 0.7564473,
        "_source": {
          "title": "The quick brow fox jumps over the lazy dog",
          "price": 15,
          "colors": [
            "blue",
            "yellow"
          ]
        }
      },
        
      ...
```

### 구문이 포함된 도큐먼트 검색

- match_phrase query 사용, 해당 문자열이 포함된 도큐먼트 리턴

```json
GET library/_search
{
  "query": {
    "match_phrase": {
      "title": "quick dog"
    }
  }
}
```

### 복합 쿼리 - bool 쿼리를 이용한 서브쿼리 조합

#### must

* "quick" 과 "lazy dog" 이 포함된 모든 문서 검색

```json
GET /library/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "quick"
          }
        },
        {
          "match_phrase": {
            "title": {
              "query": "lazy dog"
            }
          }
        }
      ]
    }
  }
}
```

#### must_not

* "quick" 또는 "lazy dog" 이 포함되지 않은 문서 검색

```json
GET /library/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "match": {
            "title": "quick"
          }
        },
        {
          "match_phrase": {
            "title": {
              "query": "lazy dog"
            }
          }
        }
      ]
    }
  }
}
```

### 특정 쿼리에 대한 가중치 조절(boost)

#### should

* OR(||)랑 비슷하지만 같지는 않다.

* 반드시 매칭 될 필요는 없지만, 매칭 되는 경우 더 높은 스코어를 가짐
* 기본 boost값은 1이다. 아래의 경우 `"lazy dog"` 에 가중치를 3으로 설정했으므로 `quick dog`
*  를 포함하고 있는 도큐먼트보다

```json
GET /library/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match_phrase": {
            "title": "quick dog"
          }
        },
        {
          "match_phrase": {
            "title": {
              "query": "lazy dog",
              "boost": 3
            }
          }
        }
      ]
    }
  }
}
```

#### must + should

* must와 should를 같이 사용하게 되면, must를 우선적으로 매칭하고 must 조건을 만족하는 도큐먼트 중에서 should 쿼리의 매칭 여부에 따라 score가 결정 된다.

```json
GET /library/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match_phrase": {
            "title": "lazy"
          }
        }
      ],
      "must": [  
        {
          "match_phrase": {
            "title": {
              "query": "dog"
            }
          }
        }
      ]
    }
  }
}
```

### Highlight - 매칭 된 부분 하이라이트

* 검색 결과값이 크고 여러 필드를 사용하는 경우 유용하다

#### highlight

* 검색결과 해당 단어와 매칭되는 단어를 highlight 표시

```json
GET /library/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match_phrase": {
            "title": "lazy"
          }
        },
        {
          "match_phrase": {
            "title": {
              "query": "dog"
            }
          }
        }
      ]
    }
  },
  "highlight" : {
    "fields" : {
      "title" : {}
    }
  }
}
```

* 검색 후 리턴 값의 highlight속성에는 em태그로 해당 키워드가 표시 된다.

```json
{
  "took": 110,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 0.7564473,
    "hits": [
      {
        "_index": "library",
        "_type": "books",
        "_id": "2",
        "_score": 0.7564473,
        "_source": {
          "title": "The quick brow fox jumps over the lazy dog",
          "price": 15,
          "colors": [
            "blue",
            "yellow"
          ]
        },
        "highlight": {
          "title": [
            "The quick brow fox jumps over the <em>lazy</em> <em>dog</em>"
          ]
        }
      },
        
        ...
```

### Filter - 검색 결과의 sub-set 도출

* 스코어를 계산하지 않고 캐싱되어 쿼리보다 대부분 빠르다

#### (bool) must + filter

* filter는 bool조건의 쿼리에서 사용할 수 있고, 아래의 쿼리는 `"dog"` 을 포함하고 있는 도큐먼트 중에서 price의 range가 5이상 10이하의 도큐먼트 들만 리턴한다.
* 알아둬야 할 것은 filter 구문을 제거하고 search만 수행했을 경우의 score와 filter를 설정했을 때의 score가 같다는 것이다. 스코어를 다시 계산하지 않고 검색 결과가 나온것 중에서 sub-set만 도출하기 때문에 쿼리보다 빠르다.

```json
GET /library/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "dog"
          }
        }
      ],
      "filter": {
        "range": {
          "price": {
            "gte": 5,
            "lte": 10
          }
        }
      }
    }
  }
}
```

#### score가 필요 없는 경우 filter만 사용

* score가 필요없이 특정 필드의 숫자 범위를 판단하는 상황에서는 filter만 사용할 수 있다. 이때는 true, false로 판단만 하게 된다.
* 결과는 price가 5보다 큰(>5) 도큐먼트들만 리턴된다

```json
GET /library/_search
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "price": {
            "gt": 5
          }
        }
      }
    }
  }
}
```

## 분석 - Analysis(_analyze)

### Tokenizer를 통해 문장을 검색어 term으로 분리

* text로부터 분리된 offset과 type, position 정보 표시

```json
GET library/_analyze
{
  "tokenizer": "standard",
  "text": "Brown fox brown dog"
}
```

```json
{
  "tokens": [
    {
      "token": "Brown",
      "start_offset": 0,
      "end_offset": 5,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "fox",
      "start_offset": 6,
      "end_offset": 9,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "brown",
      "start_offset": 10,
      "end_offset": 15,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "dog",
      "start_offset": 16,
      "end_offset": 19,
      "type": "<ALPHANUM>",
      "position": 3
    }
  ]
}
```

### Filter(토큰필터)를 통해 쪼개진 term들을 가공

* lowercase - 소문자로 변경

```json
GET library/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase"
  ],
  "text": "Brown fox brown dog"
}
```

```json
{
  "tokens": [
    {
      "token": "brown",
      "start_offset": 0,
      "end_offset": 5,
      "type": "<ALPHANUM>",
      "position": 0
    },
      ...
```

* unique - 중복 term 제거
  * 아래의 필터에서 lowercase를 제거하면, 대소문자를 구분하게 된다.

```json
GET library/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase",
    "unique"
  ],
  "text": "Brown brown brown fox brown dog"
}
```

```json
{
  "tokens": [
    {
      "token": "brown",
      "start_offset": 0,
      "end_offset": 5,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "fox",
      "start_offset": 18,
      "end_offset": 21,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "dog",
      "start_offset": 28,
      "end_offset": 31,
      "type": "<ALPHANUM>",
      "position": 2
    }
  ]
}
```

### (Tokenizer + Filter) 대신 Analyzer 사용

* 기본적인 standard analyzer는 lowercase와 standard tokenizer가 적용된다.

```json
GET library/_analyze
{
  "analyzer": "standard",
  "text": "Brown fox brown dog"
}
```

### 복합적인 문장 분석

* T: standard, F: lowercase
  * 아래의 결과를 통해서 중요하게 알고 넘어가야 할 것은 standard tokenizer와 lowercase filter로 텍스트를 저장했을 경우, 결과 값으로 리턴되는 분리된 token으로 검색해야만 원래의 문장을 검색할 수 있다.

```json
GET library/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase"  
  ],
  "text": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}
```

```json
{
  "tokens": [
    {
      "token": "the",
      "start_offset": 0,
      "end_offset": 3,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "quick.brown_fox",
      "start_offset": 4,
      "end_offset": 19,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "jumped",
      "start_offset": 20,
      "end_offset": 26,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "19.95",
      "start_offset": 29,
      "end_offset": 34,
      "type": "<NUM>",
      "position": 3
    },
    {
      "token": "3.0",
      "start_offset": 37,
      "end_offset": 40,
      "type": "<NUM>",
      "position": 4
    }
  ]
}
```

* T:letter, F:lowercase
  * letter tokenizer는 실제로 알파벳 문자가 아닌 모든것을 구분자로 판단한다. 따라서, lowercase의 분리된 문자열만 리턴된다.

```json
GET library/_analyze
{
  "tokenizer": "letter",
  "filter": [
    "lowercase"  
  ],
  "text": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}
```

```json
{
  "tokens": [
    {
      "token": "the",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 0
    },
    {
      "token": "quick",
      "start_offset": 4,
      "end_offset": 9,
      "type": "word",
      "position": 1
    },
    {
      "token": "brown",
      "start_offset": 10,
      "end_offset": 15,
      "type": "word",
      "position": 2
    },
    {
      "token": "fox",
      "start_offset": 16,
      "end_offset": 19,
      "type": "word",
      "position": 3
    },
    {
      "token": "jumped",
      "start_offset": 20,
      "end_offset": 26,
      "type": "word",
      "position": 4
    }
  ]
}
```

### Email, URL  분석

* T: standard

```json
GET library/_analyze
{
  "tokenizer": "standard",
  "text": "elastic@example.com website: https://elastic.co"
}
```

```json
{
  "tokens": [
    {
      "token": "elastic",
      "start_offset": 0,
      "end_offset": 7,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "example.com",
      "start_offset": 8,
      "end_offset": 19,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "website",
      "start_offset": 20,
      "end_offset": 27,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "https",
      "start_offset": 29,
      "end_offset": 34,
      "type": "<ALPHANUM>",
      "position": 3
    },
    {
      "token": "elastic.co",
      "start_offset": 37,
      "end_offset": 47,
      "type": "<ALPHANUM>",
      "position": 4
    }
  ]
}
```

* T: uax_url_email

```json
GET library/_analyze
{
  "tokenizer": "uax_url_email",
  "text": "elastic@example.com website: https://elastic.co"
}
```

```json
{
  "tokens": [
    {
      "token": "elastic@example.com",
      "start_offset": 0,
      "end_offset": 19,
      "type": "<EMAIL>",
      "position": 0
    },
    {
      "token": "website",
      "start_offset": 20,
      "end_offset": 27,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "https://elastic.co",
      "start_offset": 29,
      "end_offset": 47,
      "type": "<URL>",
      "position": 2
    }
  ]
}
```

## 집계(Aggregation)

* aggregation은 기본적으로 document value에 있는 값들을 사용한다. 그중에서도 저장했을때 타입이 keyword인 값들만 사용 가능
* aggregation은 쿼리랑 같은 레벨에서 사용 가능

### aggs terms를 이용한 colors.keyword 필드 값 집계

* colors.keyword 타입을 집계하면, library 인덱스의 colors keyword별로 집계된 값들이 리턴 된다.

```json
GET library/_search
{
  "size": 0,
  "aggs": {
    "popular-colors": {
      "terms": {
        "field": "colors.keyword"
      }
    }
  }
}
```

```json
{
  "took": 13,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 5,
    "max_score": 0,
    "hits": []
  },
  "aggregations": {
    "popular-colors": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "blue",
          "doc_count": 5
        },
        {
          "key": "red",
          "doc_count": 4
        },
        {
          "key": "green",
          "doc_count": 2
        },
        {
          "key": "yellow",
          "doc_count": 2
        },
        {
          "key": "black",
          "doc_count": 1
        }
      ]
    }
  }
}
```

### 검색(query)와 집계(aggregation) 동시에 사용

* 검색 쿼리와 집계를 동시에 사용하면, 검색을 먼저 수행하고 그 결과로부터 aggregation을 수행한다.
* 아래의 구문에서는 dog을 포함하고 있는 document 들의 colors.keyword타입을 집계한다.
* size value를 0으로 주면, hits결과는 표시되지 않고 aggregations 결과값만 출력한다.
  * `"size": 0`

```json
GET library/_search
{
  "query": {
    "match": {
      "title": "dog"
    }
  },
  "aggs": {
    "popular-colors": {
      "terms": {
        "field": "colors.keyword"
      }
    }
  }
}
```

```json
{
  "took": 54,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 0.2876821,
    "hits": [
      {
        "_index": "library",
        "_type": "books",
        "_id": "5",
        "_score": 0.2876821,
        "_source": {
          "title": "Lazy dog",
          "price": 9,
          "colors": [
            "red",
            "blue",
            "green"
          ]
        }
      },
      
      ...검색 결과 중략...
      ,  
  "aggregations": {
    "popular-colors": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "blue",
          "doc_count": 4
        },
        {
          "key": "red",
          "doc_count": 3
        },
        {
          "key": "yellow",
          "doc_count": 2
        },
        {
          "key": "black",
          "doc_count": 1
        },
        {
          "key": "green",
          "doc_count": 1
        }
      ]
    }
  }
}
```

### 여러개의 aggregation, sub-aggs 사용

* aggregation 안에, 다시 sub-aggregation을 정의한다.
* 아래의 코드는 두개의 aggregation을 구하게 된다.
* 먼저 price-statistic로 정의된 aggregation은 library 인덱스에 대해서 price필드의 stats(통계 - count, min, max, avg, sum)을 구하고,
* popular-colors는 먼저 colors.keyword로 terms를 정의하고, sub-aggs로 정의된 avg-price-per-color에서는 colors.keyword로 집계된 결과를 바탕으로 price의 평균값을 구하게 된다.
  * 반환되는 결과에는 bucket이 정의되고, bucket은 colors.keyword를 key로 구분한다.
  * 그 안에는 해당 컬러가 포함된 도큐먼트의 수와 price average값이 반환 된다.

```json
GET library/_search
{
  "size": 0,
  "aggs": {
    "price-statistics": {
      "stats": {
        "field": "price"
      }
    },
    "popular-colors": {
      "terms": {
        "field": "colors.keyword"
      },
      "aggs": {
        "avg-price-per-color": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}
```

```json
  
  ...

  },
  "aggregations": {
    "popular-colors": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "blue",
          "doc_count": 5,
          "avg-price-per-color": {
            "value": 7.8
          }
        },
        {
          "key": "red",
          "doc_count": 4,
          "avg-price-per-color": {
            "value": 6
          }
        },
        {
          "key": "green",
          "doc_count": 2,
          "avg-price-per-color": {
            "value": 7
          }
        },
        {
          "key": "yellow",
          "doc_count": 2,
          "avg-price-per-color": {
            "value": 8.5
          }
        },
        {
          "key": "black",
          "doc_count": 1,
          "avg-price-per-color": {
            "value": 2
          }
        }
      ]
    },
    "price-statistics": {
      "count": 5,
      "min": 2,
      "max": 15,
      "avg": 7.8,
      "sum": 39
    }
  }
}
```

## Document Update

* 동일한 URL에 데이터 입력시 기존 데이터가 대체됨
  * 반환되는 결과를 확인하면 update 된 경우 version이 +1 된다.
  * elasticsearch에서는 transaction 개념이 없기때문에 유의해야 한다.

```json
GET library/books/1
```

```json
POST library/books/1
{
  "title": "The quick brow fox",
  "price": 10,
  "colors": ["red","green","blue"]
}
```

```json
{
  "_index": "library",
  "_type": "books",
  "_id": "1",
  "_version": 3,
  "found": true,
  "_source": {
    "title": "The quick brow fox",
    "price": 10,
    "colors": [
      "red",
      "green",
      "blue"
    ]
  }
}
```

### _update API 이용

* elasticsearch에서 _update API를 사용하더라도 title만 업데이트 하는 것이 아닌, 전체 도큐먼트를 delete 후 insert하는 개념으로 업데이트가 이루어 진다.

```json
POST library/books/1/_update
{
  "doc": {
    "title": "The quick fantastic fox"
  }
}
```

## Mapping

### 매핑 정보 확인

- 데이터가 색인될 때, elasticsearch는 스스로 매핑정보를 만든다.
- 저장되는 도큐먼트의 필드값을 확인하면서 타입을 판단한다.

```json
GET library/_mapping
```

### 직접 매핑 설정한 인덱스 생성

- famous-librarians라는 인덱스를 새로 생성한다.
- settings에는 인덱스가 가지고 있는 시스템적인 설정값을 넣는다.
  - 샤드 수, 복제본 수, analysis(analyzer) 정보 등
  - analysis의 analyzer를 my-desc-analyzer로 정의하고,
  - custom type, uax_url_email 토크나이저, lowercase 필터를 사용하도록 설정한다.
- mappings에는 매핑값을 성정한다.
  - famous-librarians라는 인덱스 밑에 librarian이라는 타입이 정의되고,
  - 5개의 필드가 정의 된다.
    - 각 필드의 타입과 포맷 그리고 해당 필드를 저장할 때 사용할 analyzer를 등록한다.
- 필드의 매핑값을 설정해 놓으면, 삽입되는 데이터들은 필드의 타입에 맞게 색인 된다.
  - 만약 매핑값이 정의되지 않은 데이터가 들어오게 되면, 삽입되는 순간 매핑정보를 판단해서 등록한다.
- **주의할 점**은 필드의 타입은 수정이 불가하기 때문에, 수정을 하고싶다면 인덱스와 매핑정보를 다시 만들고 데이터를 처음부터 다시 넣어야 한다.

```json
PUT famous-librarians
{
  "settings": {
    "number_of_shards": 2,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "my-desc-analyzer": {
          "type": "custom",
          "tokenizer": "uax_url_email",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "librarian": {
      "properties": {
        "name": {
          "type": "text"
        },
        "favourite-colors": {
          "type": "keyword"
        },
        "birth-date": {
          "type": "date",
          "format": "year_month_day"
        },
        "hometown": {
          "type": "geo_point"
        },
        "description": {
          "type": "text",
          "analyzer": "my-desc-analyzer"
        }
      }
    }
  }
}
```

### 예제 데이터 삽입 1

* favaorite-colors는 keyword 타입으로 삽입되고,
* hometown에는 object 타입으로 데이터를 삽입할 수 있는데, object 타입 중에 geo_point 타입은 latitude, longitude 쌍으로 데이터를 넣는다.
  * geo_point 타입의 경우 거리를 기준으로하는 쿼리가 가능하다.

```json
PUT famous-librarians/librarian/1
{
  "name": "Sarah Byrd Askew",
  "favourite-colors": [
    "Yellow",
    "light-grey"
  ],
  "bitrh-date": "1877-02-15",
  "hometown": {
    "lat": 32.349722,
    "lon": -87.641111
  },
  "description": "An American public librarian who pioneered the establishment of county libraries in the United States - https://en.wikipedia.org/wiki/Sarah_Byrd_Askew"
}
```

```json
PUT famous-librarians/librarian/2
{
  "name": "John J. Beckley",
  "favourite-colors": [
    "Red",
    "off-white"
  ],
  "bitrh-date": "1757-08-07",
  "hometown": {
    "lat": 51.507222,
    "lon": -0.1275
  },
  "description": "An American political campaign manager and the first Librarian of the United States Congress, - https://en.wikipedia.org/wiki/John_J._Beckley"
}
```

### query_string - keyword 필드 확인

* keyword 타입으로 저장된 favourite-colors 필드는 분석과정(analyzer를 통해서)을 거치지 않기 때문에, 대문자 Yellow가 저장되어 있다.
* 따라서, yellow OR off-white 쿼리에 검색되지 않는다.
* keyword 타입은 aggregation을 위해 원본데이터가 저장이 되어야 한다. 하지만, 검색을 위해서는 데이터를 가공해서 저장해야 하는데, 그 과정을 거치지 않으므로 keyword 타입을 검색 할때 주의해야한다.

```json
GET famous-librarians/_search
{
  "query": {
    "query_string": {
      "fields": [
        "favourite-colors"
      ],
      "query": "yellow OR off-white"
    }
  }
}
```

### range - 날짜 범위 검색

* date 타입을 검색할 때에는 now 키워드를 사용할 수 있다.
  * h - 시간
  * d - 일
  * M - 달
  * y - 년
  * `now-200y` 의 경우 지금으로부터 200년 전이다.

```json
GET famous-librarians/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": {
        "range": {
          "birth-date": {
            "gte": "now-200y",
            "lte": "2000-01-01"
          }
        }
      }
    }
  }
}
```

### geo_distance - 특정 지점에서 반경 100km 거리 검색

* 특정 지점으로부터 반경 이내에 있는 특정 위치 검색하기 등 거리 탐색에 사용

```json
GET famous-librarians/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "100km",
          "hometown": {
            "lat": 32.41,
            "lon": -86.92
          }
        }
      }
    }
  }
}
```