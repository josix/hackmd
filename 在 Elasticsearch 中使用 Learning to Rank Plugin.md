---
tags: Tool Sharing, draft
---
# 在 Elasticsearch 中使用 Learning to Rank Plugin

[TOC]

---

## 什麼是 Learning to Rank

Learning to Rank 又稱為 Machine-Learned Rankingy 為機器學習的任務之一，Learning to Rank 任務主要為給定可以透過特定給分（例如相關與否、回饋評分高低等）以進行排序的資料集，學習的目標為產生一個排序該排序順序可以讓高分的項目位置在列表的越前面，反之不相關、低分的項目位置則在列表的後面。因此，相較於回歸（Regression）問題，Learning to Rank 注重的是結果的排序，訓練時期給予每個排序項目的給分數值本身並不重要，最重要的是在於排序的順位。

Learning to Rank 經常使用在資訊檢索（Information Retrievel）的情境當中，透過給定的 query 、專家建立的 Judgement List 包含各個 query 及該檢索結果的給分列表以及有助於提高相關性的特徵，如 query 出現的詞頻（Term Frequency）、詞數（Term Count）、檔案頻率（Document Frequency）、逆向檔案頻率（Inverse Document Frequency, IDF）、詞位置（Term Position）等數值，訓練模型，以便預測有相似的 query 時可以得到相關性高的排序輸出。


## Elasticsearch Learning to Rank (LTR) Plugin 提供功能
為了在 Elasticsearch 中進行 Learning to Rank 的任務，Elasticsearch LTR Plugin 主要提供的功能的主要功能如下：
- 協助透過 Elasticsearch 既有功能建立特徵
- 於 Elasticsearch 既有功能上建立其他特徵工程（Feature Engineering）
- 紀錄特徵值至 Judgement List 以利下游任務套用如 XGBoost, LightGBM, Ranklib 等不同模型訓練
- 接受訓練完成的模型，透過 Elasticsearch LTR 自定義的 DSL primitive 使該模型可以應用於 Elasticsearch 搜尋時使用，產出較好的排序結果。

特別要提到的是，Elasticsearch LTR 不提供模型訓練的功能，由於模型種類繁多並且效能依照不同領域也會有不同的呈現，因此 Elasticsearch LTR 保留此彈性讓使用者可以自行使用自己偏好的模型。另外，Elasticsearch LTR 也不提供 Judgement LIst 的生成，此列表也需要各個領域的專家進行標記，因此保留此彈性讓使用者自行建立。
## 如何使用 Elasticsearch Learning to Rank (LTR) Plugin
### 安裝 Elasticsearch LTR Plugin
安裝 Elasticsearch LTR 僅需要在已安裝好的 Elasticsearch 中安裝對應的 Elasticsearch LTR Plugin 版本即可：

```bash
./bin/elasticsearch-plugin install https://github.com/o19s/elasticsearch-learning-to-rank/releases/download/v1.5.4-es7.11.2/ltr-plugin-v1.5.4-es7.11.2.zip
```
其他版本對應可以參考 [Elasticsearch LTR Release History](https://github.com/o19s/elasticsearch-learning-to-rank/releases)

若使用 Open Distro for Elasticsearch 可以參考 [Open Distro for Elasticsearch Version History](https://opendistro.github.io/for-elasticsearch-docs/version-history/) 找尋對應的 Elasticsearch 版本再找到對應的 Elasticsearch LTR 版本
### 如何在 Elasticsearch LTR 新增特徵
在 Elasticsearch LTR 中所有特徵存放的層級為 Feature Store > Feature Sets > Features
#### Feature Store
以 Elasticsearch 的 Index 實作, 用於儲存特徵和上傳模型的 metadata，預設 Elasticsearch LTR 的 Index 名稱為 `_ltr`。
在使用 Elasticsearch LTR 之前，首先必須建立預設 Feature Store，透過此 API 可以建立預設 Feature Store:
```
PUT _ltr
```

若需要重新建立預設的 Feature Store，可以透過下方 API 刪除後再重新建立：

```
DELETE _ltr
```

一般來說只會使用到一個 Feature Store，而不同的特徵會以 Feature Set 管理。

#### Feature Set

Feature Set 是直接參與 Elasticsearch LTR 最重要的部分，是特徵值的集合，可以依照模型不同、想要使用的特徵建立不同集合。

##### 建立 Feature Set
建立一個 Feature Set 可以透過 POST 其 payload 只需要帶入 Feature Set 名稱和所包含所有特徵的串列，範例如下：

```
POST _ltr/_featureset/more_movie_features
{
   "featureset": {
        "features": [
            {
                "name": "title_query",
                "params": [
                    "keywords"
                ],
                "template_language": "mustache",
                "template": {
                    "match": {
                        "title": "{{keywords}}"
                    }
                }
            },
            {
                "name": "title_query_boost",
                "params": [
                    "some_multiplier"
                ],
                "template_language": "derived_expressions",
                "template": "title_query * some_multiplier"
            },
            {
                "name": "custom_title_query_boost",
                "params": [
                    "some_multiplier"
                ],
                "template_language": "script_feature",
                "template": {
                    "lang": "painless",
                    "source": "params.feature_vector.get('title_query') * (long)params.some_multiplier",
                    "params": {
                        "some_multiplier": "some_multiplier"
                    }
                }
            }
        ]
   }
}
```

其中，Feature 的欄位如下：
- `name`: 特徵的名稱，用於後續 logging 使用，所有的名稱必須是唯一的不可重複的。
- `params`: 為使用者希望帶入 query 的參數名稱。
- `template`: 所有的特徵都是基於 Elasticsearch query 執行後取得的，檢索結果中的分數便是訓練資料，因此所有 Elasticsearch query 都是可以作為特徵的，如 `match`、`multi_match`、`match_all` 等都是合法的 query，特別是針對 document 屬性的 query 也可以透過 [`function_score`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html) 來獲得該計分：

    ```
    {
        "query": {
            "function_score": {
                "functions": {
                    "field": "rate_score"
                },
                "query": {
                    "match_all": {}
                }
            }
        }
    }
    ```
    也可以取得根據使用者距離計算與 query 相關性的分數：

    ```
    {
        "query": {
            "bool" : {
                "must" : {
                    "match_all" : {}
                },
                "filter" : {
                    "geo_distance" : {
                        "distance" : "200km",
                        "pin.location" : {
                            "lat" : {{users_lat}},
                            "lon" : {{users_lon}}
                        }
                    }
                }
            }
        }
    }
    ```

- `params`: 可以見到如 `{{keywords}}`、`{{users_lat}}`、`{{users_lon}}` 等語法，這些為 Elasticsearch [Search Template](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/search-template.html) 的 [Mustache](https://mustache.github.io/) 語法，透過 Search Template 可以將 `params` 中使用者加入的參數帶入至 search query 中。
- `template_language`: 為 `template` 渲染的方式，共有 `mustache`、`derived_expression`、`script_feature` 三種。

若需要加入 feature 至現有的 Feature Set 可以透過 `POST /_ltr/_featureset/<fea>/_addfeatures`：

```
POST /_ltr/_featureset/my_featureset/_addfeatures
{
    "features": [{
        "name": "user_rating",
        "params": [],
        "template_language": "mustache",
        "template" : {
            "function_score": {
                "functions": {
                    "field": "vote_average"
                },
                "query": {
                    "match_all": {}
                }
            }
        }
    }]
}
```


##### 取得 Feature Set 資訊
取得 Feature Set 的資訊可以透過此 API
```
GET _ltr/_featureset/<feature_set_name>
```
或取得所有 Feature Set 資訊
```
GET _ltr/_featureset
```
若需要針對字首過濾 Feature Set 可以透過下方 API
```
GET _ltr/_featureset?prefix=
```
刪除 Feature Set：
```
DELETE _ltr/_featureset/<feature_set_name>
```



### 如何取得 Elasticsearch LTR 特徵值
### 如何上傳排序模型至 Elasticsearch LTR
### 透過 Elasticsearch LTR 搜尋
## Reference
- [Elasticsearch Learning to Rank GitHub](https://github.com/o19s/elasticsearch-learning-to-rank) 
- [Learning to rank Weikipedia](https://en.wikipedia.org/wiki/Learning_to_rank)
- [Elasticsearch Learning to Rank Document](https://elasticsearch-learning-to-rank.readthedocs.io/en/latest/index.html)
- [We’re Bringing Learning to Rank to Elasticsearch](https://opensourceconnections.com/blog/2017/02/14/elasticsearch-learning-to-rank/)
- [(YouTube) Elasticsearch Learning to Rank: Search as a ML Problem & Search Logs + ML](https://www.youtube.com/watch?v=ZeeGskd1bjY)
- [(YouTube) Conversion Models: Building Learning to Rank Training Data - Doug Turnbull, OpenSource Connections](https://youtu.be/33QDCpZmR-E)



