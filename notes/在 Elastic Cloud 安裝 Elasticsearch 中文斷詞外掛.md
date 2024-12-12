# 在 Elastic Cloud 安裝 Elasticsearch 中文斷詞外掛

## 前言

在 Elasticsearch 中，[預設的斷詞器](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-tokenizer.html) (tokenizer) 是以空白分隔的方式進行斷詞，這對於英文來說是沒有問題的，但對於中文來說，每個字都會視為一個 token，例如「雞蛋」這個詞，如果使用預設的斷詞器，會被斷成「雞」、「蛋」兩個 token，這樣的斷詞方式對於中文來說無法讓 token 具有意義，因此需要使用中文斷詞器來解決這個問題。

## 中文斷詞

其原理是將中文文本根據內建的詞庫進行斷詞，並將斷詞後的結果作為 token，這樣就可以讓 Elasticsearch 利用這些 token 建立索引，並進行搜尋，較知名的 plugin 為 [analysis-ik](https://github.com/infinilabs/analysis-ik)。

安裝好後，可以在 Kibana Dev tools 用 `GET /_analyze` API 來測試斷詞效果，如果使用預設的斷詞器，會得到以下結果：

```json
GET /_analyze
{
  "tokenizer": "standard",
  "text": "資料庫效能優化"
}

// response
{
  "tokens": [
    {
      "token": "資",
      "start_offset": 0,
      "end_offset": 1,
      "type": "<IDEOGRAPHIC>",
      "position": 0
    },
    {
      "token": "料",
      "start_offset": 1,
      "end_offset": 2,
      "type": "<IDEOGRAPHIC>",
      "position": 1
    },
    {
      "token": "庫",
      "start_offset": 2,
      "end_offset": 3,
      "type": "<IDEOGRAPHIC>",
      "position": 2
    },
    {
      "token": "效",
      "start_offset": 3,
      "end_offset": 4,
      "type": "<IDEOGRAPHIC>",
      "position": 3
    },
    {
      "token": "能",
      "start_offset": 4,
      "end_offset": 5,
      "type": "<IDEOGRAPHIC>",
      "position": 4
    },
    {
      "token": "優",
      "start_offset": 5,
      "end_offset": 6,
      "type": "<IDEOGRAPHIC>",
      "position": 5
    },
    {
      "token": "化",
      "start_offset": 6,
      "end_offset": 7,
      "type": "<IDEOGRAPHIC>",
      "position": 6
    }
  ]
}
```

每個中文字都被斷成一個 token，無法讓 token 具有足夠意義

換成 analysis-ik 的斷詞器，anaylsis-ik 提供了 `ik_smart`、`ik_max_word` 兩種斷詞器

- `ik_smart`：粗粒度的分詞
- `ik_max_word`：細粒度的分詞

就個人理解 `ik_smart` 是以可取得最大片段&最少數量 token 的方式進行斷詞，而 `ik_max_word` 則是以可取得最多片段的方式進行斷詞，即 token 中可能還有更小的 token

再試一次 `GET /_analyze` 並使用 `ik_smart`:

```json
GET /_analyze
{
  "tokenizer": "ik_smart",
  "text": "資料庫效能優化"
}

// response
{
  "tokens": [
    {
      "token": "資",
      "start_offset": 0,
      "end_offset": 1,
      "type": "CN_CHAR",
      "position": 0
    },
    {
      "token": "料",
      "start_offset": 1,
      "end_offset": 2,
      "type": "CN_CHAR",
      "position": 1
    },
    {
      "token": "庫",
      "start_offset": 2,
      "end_offset": 3,
      "type": "CN_CHAR",
      "position": 2
    },
    {
      "token": "效能",
      "start_offset": 3,
      "end_offset": 5,
      "type": "CN_WORD",
      "position": 3
    },
    {
      "token": "優",
      "start_offset": 5,
      "end_offset": 6,
      "type": "CN_CHAR",
      "position": 4
    },
    {
      "token": "化",
      "start_offset": 6,
      "end_offset": 7,
      "type": "CN_CHAR",
      "position": 5
    }
  ]
}
```

就結果來看跟使用預設 tokenizer 相比根本一樣，這是因為 analysis-ik 的詞庫所持有的詞都是簡體中文，因此無法被正確辦識，如果輸入繁體中文，則可以正確斷詞：

```json
GET /_analyze
{
  "tokenizer": "ik_smart",
  "text": "资料库效能优化"
}

// response
{
  "tokens": [
    {
      "token": "资料库",
      "start_offset": 0,
      "end_offset": 3,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "效能",
      "start_offset": 3,
      "end_offset": 5,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "优化",
      "start_offset": 5,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 2
    }
  ]
}
```

要解決繁體無法被正確辦識的問題有 2 個方向：

1. 簡繁轉換
2. 新增自定義詞庫

利用簡繁轉換可以將繁體中文轉換成簡體中文，這樣就可以讓 analysis-ik 進行斷詞，而新增自定義詞庫則是可以讓 analysis-ik 能夠辦識更多的詞，因此也可以直接新增繁體的詞進入字庫。

# 簡繁轉換

這裡使用 [analysis-smartcn](https://github.com/infinilabs/analysis-stconvert) 在進行斷詞時將繁體中文轉換成簡體中文，這樣就可以讓 analysis-ik 進行斷詞。

預設使用會將簡體中文轉成繁體中文，但可以透過設定 `convert_type` 來指定轉換的方向，例如 `t2s` 代表繁體轉簡體，`s2t` 代表簡體轉繁體 (要在 settings 中設定)。

```json
GET /_analyze
{
  "tokenizer": "stconvert",
  "text": "资料库效能优化"
}

// response
{
  "tokens": [
    {
      "token": "資料庫效能優化",
      "start_offset": 0,
      "end_offset": 7,
      "type": "word",
      "position": 0
    }
  ]
}
```

首先建新一個 index `test` 並設定 mapping，並在 mapping 中指定使用的斷詞器：

```json
PUT test/_settings
{
  "index":{
    "analysis":{
      "analyzer": {
        "ik_smart_plus": { // 建立自訂的 analyzer 以便在這個 index 中使用
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": [
            "tsconvert"
          ]
        }
			},
      "filter": {
        "tsconvert" : {
          "type" : "stconvert",
          "convert_type" : "t2s" // 繁體轉簡體
        }
      }
    }
  }
}
```

測試 analyzer，不過 endpoint 前面要加上 index name，例如 `test/_analyze`：

```json
POST test/_analyze
{
  "analyzer": "ik_smart_plus",
  "text": "資料庫效能優化"
}

// response
{
  "tokens": [
    {
      "token": "資料庫",
      "start_offset": 0,
      "end_offset": 3,
      "type": "CN_WORD",
      "position": 0
    },
    {
      "token": "效能",
      "start_offset": 3,
      "end_offset": 5,
      "type": "CN_WORD",
      "position": 1
    },
    {
      "token": "優化",
      "start_offset": 5,
      "end_offset": 7,
      "type": "CN_WORD",
      "position": 2
    }
  ]
}
```

## 自定義詞庫

在 analysis-ik 的設定檔中，可以指定自定義詞庫的路徑，這樣就可以讓 analysis-ik 進行斷詞時使用自定義詞庫。

在 local 的話可編輯 ` /usr/share/elasticsearch/config/analysis-ik/IKAnalyzer.cfg.xml`，在 `ext_dict` 中指定自定義詞庫的路徑，例如：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">zh_words.txt</entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<!-- <entry key="remote_ext_dict">words_location</entry> -->
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>
```

若是 elastic cloud，則需先[下載](https://release.infinilabs.com/analysis-ik/stable/)對應版號的 release 再自行[打包](https://www.elastic.co/guide/en/cloud/current/ec-custom-bundles.html)後上傳，安裝後能用 `/_nodes/plugins` 取得 plugin 資訊：

```json
GET /_nodes/plugins?filter_path=nodes.*.plugins

{
  "nodes": {
    "NODE_ID": {
      "plugins": [
        {
          "name": "analysis-stconvert",
          "version": "8.14.3",
          "elasticsearch_version": "8.14.3",
          "java_version": "1.8",
          "description": "STConvert is a analysis plugin that convert Chinese characters between traditional and simplified.",
          "classname": "com.infinilabs.stconvert.elasticsearch.AnalysisSTConvertPlugin",
          "extended_plugins": [],
          "has_native_controller": false,
          "licensed": false,
          "is_official": false,
          "legacy_interfaces": [
            "AnalysisPlugin"
          ],
          "legacy_methods": [
            "getAnalyzers",
            "getCharFilters",
            "getTokenFilters",
            "getTokenizers"
          ]
        },
        {
          "name": "analysis-ik",
          "version": "8.14.3",
          "elasticsearch_version": "8.14.3",
          "java_version": "1.8",
          "description": "IK Analyzer for Elasticsearch",
          "classname": "com.infinilabs.ik.elasticsearch.AnalysisIkPlugin",
          "extended_plugins": [],
          "has_native_controller": false,
          "licensed": false,
          "is_official": false,
          "legacy_interfaces": [
            "AnalysisPlugin"
          ],
          "legacy_methods": [
            "getAnalyzers",
            "getTokenizers"
          ]
        }
      ]
    }
  }
}
```

## 踩坑

elastic cloud 出於安全性與穩定性考量，預設只提供一些內建 plugin，要使用 custom plugin 的話得向 support 提出 issue 讓他們協助開通

## Future work

將要搜尋的資料使用台灣 CKIP 斷詞器分析以擴充中文字庫

## Reference

https://blog.philip-huang.tech/?page=elasticsearch-chinese-search-optimize
https://godleon.github.io/blog/Elasticsearch/Elasticsearch-advanced-search/
