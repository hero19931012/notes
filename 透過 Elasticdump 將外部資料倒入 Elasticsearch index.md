## Input

要 dump 的資料來源檔案以 jsonl 為佳，若原資料是 json 可寫個小工具轉

如：
```JavaScript
const input = require('./input.json')
const fs = require('fs')
const path = require('path')

output = ""

for (const item of input) {
	output += JSON.stringify(item) + "\n"
}
fs.writeFileSync(path.join(__dirname, 'output.jsonl'), output)
```

使用：
```bash
node ./transfer-json-to-jsonl.js
```

## Dump

使用 docker image 加上 --rm 就不需在 local 安裝 elasticdump
由於是 local 測試因此 elasticsearch 並沒有開啟 security
如要倒入如 elastic cloud 等服務的 instance 則 --output 要改成 https
`--output=https://{USERNAME}:{PASSWORD}@{INSTANCE_ENDPOINT}/{INDEX_NAME}`

cmd:
```bash
#!/bin/bash

docker run --rm -ti --net=elastic \
-v {PATH}:/tmp \
elasticdump/elasticsearch-dump \
--input=/tmp/input.jsonl \
--output=http://host.docker.internal:9200/{INDEX_NAME} \
--type=data
```

但是只有這樣會遇到下面錯誤
```JSON
{
  _index: 'title',
  _id: 'kHfYvZIBm3QVcD1t0M81',
  status: 500,
  error: {
    type: 'not_x_content_exception',
    reason: 'Compressor detection can only be called on some xcontent bytes or compressed xcontent bytes'
  }
}
```

根據[這篇 stackoverflow 的回覆](https://stackoverflow.com/a/55728041) 指出 json 檔案應指派給 doc.\_source，故要使用 elasticdump 的 --transform 參數並傳入 JavaScript code
```bash
docker run ...
--transform="doc._source=Object.assign({},doc)" # assign json into doc._source
```

也可用 module export 的方式指定 .js file
```bash
docker run ...
--transform="@/tmp/transforms/transform.js"
```

內容就是 JavaScript 模組化的寫法
```JavaScript
module.exports = function(doc,options) {
  doc._source=Object.assign({},doc)
}
```

再跑一次 script
```bash
Thu, 24 Oct 2024 09:35:33 GMT | starting dump
Thu, 24 Oct 2024 09:35:33 GMT | Will modify documents using these scripts: @/tmp/transforms/transform.js
Thu, 24 Oct 2024 09:35:33 GMT | got 90 objects from source file (offset: 0)
Thu, 24 Oct 2024 09:35:33 GMT | sent 90 objects to destination elasticsearch, wrote 90
Thu, 24 Oct 2024 09:35:33 GMT | got 0 objects from source file (offset: 90)
Thu, 24 Oct 2024 09:35:33 GMT | Total Writes: 90
Thu, 24 Oct 2024 09:35:33 GMT | dump complete
```
倒入成功
## Transform
然而 jsonl 倒入 elasticsearch 後，進入 kibana dev tool 讀出來的資料發現巢狀結構會扁平化

```json
  "author": "[56]",
  "genre": "[2]",
  "tag": """["tag_1", "tag_2"]""",
```
這些巢狀內容原本都是預期以 array 的方式存入，現在變成了 json 字串

為了解析並以正確的方式存入，在 /transform/transform.js 加入
```javascript
  for (let key in doc._source) {
    if (typeof doc._source[key] === 'string') {
      try {
        doc._source[key] = JSON.parse(doc._source[key])
      } catch (e) {}
    }
  }
```
確認資料正確
```json
"author": [102],
"genre": [1,2,12],
"tag": ["異世界"],
```

