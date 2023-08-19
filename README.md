# What is this?
- Official ES kuromoji plugin uses ipadic, which is quite outdated.
- https://github.com/codelibs/elasticsearch-analysis-kuromoji-ipadic-neologd is the elasticsearch plugin to add neologd on top of kuromoji, but unfortunately, it does not work for ES7.
- This repo re-packages https://github.com/codelibs/elasticsearch-analysis-kuromoji-ipadic-neologd so that it works in ES7.

# lucene-analyzers-kuromoji-ipadic-neologd-8.11.1-20200910
- As ES analyzer is just a wrapper on top of lucene analyzer, we need to work with lucene-analyzers-kuromoji to add additional vocabulary.
- Fortunately, someone has this [repo](https://github.com/kazuhira-r/kuromoji-with-mecab-neologd-buildscript) to download everything, extract, prune from dictionary source and package it to a jar file (./lucene-analyzers-kuromoji-ipadic-neologd-8.11.1-20200910)


# packge this ES plugin
- Now the lucene analyzer is available as a local jar file, we can use this to package an elasticsearch plugin

```
mvn install:install-file -Dfile=./lucene-analyzers-kuromoji-ipadic-neologd-8.11.1-20200910.jar -DgroupId=org.codelibs -DartifactId=lucene-analyzers-kuromoji-ipadic-neologd -Dversion=8.11.1-20200910 -Dpackaging=jar

mvn package
```

- The plugin should be then available at ./target/releases/elasticsearch-analysis-kuromoji-ipadic-neologd-7.17.1-SNAPSHOT.zip

# Install ES plugin
- I am working with a ES cluster within docker image, here is the instruction
```
docker ps # get container id, eg. e435db2b9247
docker cp /<dir>/elasticsearch-analysis-kuromoji-ipadic-neologd/target/releases/elasticsearch-analysis-kuromoji-ipadic-neologd-7.17.1-SNAPSHOT.zip e435db2b9247:/usr/share/elasticsearch/

>>> docker cli, install plugin
sh-5.0# ./bin/elasticsearch-plugin install file:///usr/share/elasticsearch/elasticsearch-analysis-kuromoji-ipadic-neologd-7.17.1-SNAPSHOT.zip
sh-5.0# ./bin/elasticsearch-plugin list # list  plugins, analysis-kuromoji-ipadic-neologd should show up.
analysis-icu
analysis-kuromoji
analysis-kuromoji-ipadic-neologd
>>>>>>>>>>>>>>>>>>>>>>>

# restart ES by restarting docker 
docker stop e435db2b9247
docker start e435db2b9247
```

# Test plugin

## Test Katakana with official Kuromoji plugin
Noop ipadic is not enough
```
GET _analyze
{
  "analyzer": "kuromoji",
  "text": ["キャメルトランスポート"]
}

>>>>>
{
  "tokens" : [
    {
      "token" : "キャメルトランスポート",
      "start_offset" : 0,
      "end_offset" : 11,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

## Test Katakana with kuromoji_ipadic_neologd plugin
```
PUT index-00001
{
  "settings": {
    "index": {
      "analysis": {
        "tokenizer": {
          "kuromoji_ipadic_neologd_search_mode": {
                "type": "kuromoji_ipadic_neologd_tokenizer",
                "mode": "search"
              }
        },
        "analyzer": {
          "kuromoji_normalize": {                 
            "char_filter": [
              "icu_normalizer"                    
            ],
            "tokenizer": "kuromoji_ipadic_neologd_search_mode",
            "filter": [
              "kuromoji_ipadic_neologd_part_of_speech",
              "cjk_width",
              "ja_stop",
              "kuromoji_ipadic_neologd_stemmer",
              "lowercase"
            ]
          }
        }
      }
    }
  }
}

POST /index-00001/_analyze
{
  "analyzer": "kuromoji_normalize",
  "text": "キャメルトランスポート"
}

>>>>>>>>>>>>

{
  "tokens" : [
    {
      "token" : "キャメル",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "キャメルトランスポート",
      "start_offset" : 0,
      "end_offset" : 11,
      "type" : "word",
      "position" : 0,
      "positionLength" : 2
    },
    {
      "token" : "トランスポート",
      "start_offset" : 4,
      "end_offset" : 11,
      "type" : "word",
      "position" : 1
    }
  ]
}
```
Yayy!! tokens!!!