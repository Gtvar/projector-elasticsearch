# Index & Autocomplete in Elasticsearch

### Load data
```
PUT _bulk
{ "create" : { "_index" : "movies", "_id" : "135569" } }
{ "id": "135569", "title" : "Star Trek Beyond", "year":2016 , "genre":["Action", "Adventure", "Sci-Fi"] }
{ "create" : { "_index" : "movies", "_id" : "122886" } }
{ "id": "122886", "title" : "Star Wars: Episode VII - The Force Awakens", "year":2015 , "genre":["Action", "Adventure", "Fantasy", "Sci-Fi", "IMAX"] }
{ "create" : { "_index" : "movies", "_id" : "109487" } }
{ "id": "109487", "title" : "Interstellar", "year":2014 , "genre":["Sci-Fi", "IMAX"] }
{ "create" : { "_index" : "movies", "_id" : "58559" } }
{ "id": "58559", "title" : "Dark Knight, The", "year":2008 , "genre":["Action", "Crime", "Drama", "IMAX"] }
{ "create" : { "_index" : "movies", "_id" : "1924" } }
{ "id": "1924", "title" : "Plan 9 from Outer Space", "year":1959 , "genre":["Horror", "Sci-Fi"] }
```

### Just show standard tokenizer

```
POST /movies/_analyze?pretty
{
   "tokenizer" : "standard",
   "filter": [{"type":"edge_ngram", "min_gram": 1, "max_gram": 4}],
   "text" : "Star"
}
```

```
{
  "tokens" : [
    {
      "token" : "S",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "St",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "Sta",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "Star",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    }
  ]
}

```

### Add new index for autocomplete with mappings

```
PUT /autocomplete
{
  "settings": {
    "number_of_shards": 1
  },
  "mappings": {
       "properties": {
           "title": {
               "type": "search_as_you_type"
           },
           "genre": {
               "type": "search_as_you_type"
           }
       }
   }
}
```

### Reindex data from `movies` index to `autocomplete`

```
POST _reindex?pretty
{
 "source": {
   "index": "movies"
 },
 "dest": {
   "index": "autocomplete"
 }
}
```

### Show mapping index `autocomplete`

```
GET /autocomplete/_mapping?pretty
```

```
{
  "autocomplete" : {
    "mappings" : {
      "properties" : {
        "genre" : {
          "type" : "search_as_you_type",
          "doc_values" : false,
          "max_shingle_size" : 3
        },
        "id" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "search_as_you_type",
          "doc_values" : false,
          "max_shingle_size" : 3
        },
        "year" : {
          "type" : "long"
        }
      }
    }
  }
}

```

### Try to autocomplete search

```
GET autocomplete/_search
{
  "size": 5,
   "query": {
       "multi_match": {
           "query": "Sta",
           "type": "bool_prefix",
           "fields": [
               "title",
               "title._2gram",
               "title._3gram"
           ]
       }
   }
}
```

### Results:

```
{
  "took" : 6,
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
        "_index" : "autocomplete",
        "_id" : "135569",
        "_score" : 1.0,
        "_source" : {
          "id" : "135569",
          "title" : "Star Trek Beyond",
          "year" : 2016,
          "genre" : [
            "Action",
            "Adventure",
            "Sci-Fi"
          ]
        }
      },
      {
        "_index" : "autocomplete",
        "_id" : "122886",
        "_score" : 1.0,
        "_source" : {
          "id" : "122886",
          "title" : "Star Wars: Episode VII - The Force Awakens",
          "year" : 2015,
          "genre" : [
            "Action",
            "Adventure",
            "Fantasy",
            "Sci-Fi",
            "IMAX"
          ]
        }
      }
    ]
  }
}
```

## Install

```
docker-compose up
curl -X POST  'localhost:9200/_security/user/kibana_system/_password' -H 'Content-Type: application/json'   -u elastic:qwerty  -d '{"password":"qwerty"}'
```

### Access to console
```
http://localhost:5601/app/dev_tools#/console
elastic
qwerty
```