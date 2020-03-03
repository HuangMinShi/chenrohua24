---
title: '以 xlsx-style 處理 Excel 文件'
date: 2019-10-23 17:54:25
tags:
---

# 目標
本文學習利用 Node.js 第三方套件 xlsx-style，處理 Excel 文件。首先釐清使用者期望輸出結果來界定目標，中間介紹套件的基本概念及用法，最後實做流程為匯入試算表、萃取資訊、處理轉換資料、計算有效範圍到匯出。

<!-- more -->

# 問題情境
某日 Nicole 即將開會，手上有空白座位表及與會人員清單，希望指定人員至特定座位，並匯出座位表張貼至會議室外牆做導引。難道得從人員清單一位一位的複製貼上至座位表？

因此指派位置欄位，依據指定位置輸出人員特定資訊，並附加基本格式為本案的實作目標。最後輸出的 Excel 形式如下：

![座位表](https://i.imgur.com/0csetYD.jpg)
![清單](https://i.imgur.com/zADEztQ.jpg)

# 認識套件 xlsx-style
## 物件
  * workbook 為活頁簿，藉由 xlsx-style 讀取 Excel 檔案獲得。
  * worksheet 為活頁簿當中的工作表。
  * cell 為儲存格，如 A12, B7。 常用屬性鍵有 v 原始值，t 類型及 s 格式。

  三者關係如下：

  ```javascript
  // workbook
  {
    SheetNames: ['sheet1', 'sheet2'],
    Sheets: {
      // worksheet
      sheet1: {
        // cell
        A1: { ... },
        // cell
        A2: { ... },
      },
      // worksheet
      sheet2: {
        // cell
        A1: { ... },
        // cell
        A2: {
          // cell type 為 String
          t: 's',
          // raw value 原始值
          v: '金剛戰士',
          // style
          s: {
            border: {}
          }
        },
      }
    }
  }
  ```

## 用法
  * `XLSX.readFile` 讀取 Excel 文件，返回 workbook
      `const workbook = XLSX.readFile('excel.xlsx', opts)`
  * `workbook.SheetNames` 取得工作表名 array
      `const sheetNames = workbook.SheetNames // 返回 ['sheet1', 'sheet2']`
  * `workbook.Sheets[Sheetname]` 取得工作表
      `const worksheet = workbook[sheetNames[1]]`
  * `XLSX.utils.sheet_to_json` 轉換 worksheet 物件至 JSON 物件
      `const objOfJson = XLSX.utils.sheet_to_json(worksheet)`
  * `worksheet[address]` 取得 cell 物件
      `const b2 = worksheet['B2'] // 返回 { v: '金剛戰士', t: 's', s: {...}}`
  * `worksheet['!ref']` 取得工作表有效範圍
      `worksheet['!ref'] // 返回 'A1:B2'`
  * `XLSX.writeFile` 匯出 Excel 文件
      `XLSX.writeFile(wb, 'newExcel.xlsx')`

# 實作流程
目前使用者已有空白座位表及與會人員清單。首先於清單新增 1 行並填入想指定的 position。其次定義每個座位須包含 4 個儲存格，儲存格由上至下將依序顯示人員編號、所屬公司、職業及名字（no, company, job, name）。

實作方面則拆解為格式及內容 2 部分。格式包含標題、框線、座位及會議室環境的配置（門及講台），內容則為人員資訊，緊接著編寫程式碼的步驟如下：

1. 載入 Excel 中的 2 張工作表，分別為
  * 空白座位 worksheet
  * 與會人員 worksheet
2. 轉換 worksheets 以方便處理資料
  * 過濾空白座位 worksheet ，返回儲存每張座位第 1 個儲存格的 array
  * 轉換與會人員 worksheet ，返回 JSON 物件
3. 處理轉換後的 worksheets
  * 依據配置寫入標題及框線，產生會議室的 template worksheet
  * 依據人員資訊及指定的座位，產生人員的 data worksheet
  * 合併 template 及 data ，返回 output worksheet
4. 計算 output worksheet 的有效範圍
5. 生成 workbook
6. 生成 Excel 檔案

# 程式碼
## 主執行 - main.js
```javascript
/*======================
  基本設定
========================*/
const XLSX = require('xlsx-style')

const { genTemplate, genData } = require('./libs/generate')
const { xlsxToJson, getRange, merge } = require('./libs/func')

// 宣告變數
const display = ['no', 'company', 'job', 'name']
const config = {
  header: {
    value: '第70屆金鐘獎頒獎典禮工作會議',
    position: 'E2:E2'
  },
  location: {
    value: '會議室座位表',
    position: 'J4:J4'
  },
  door1: {
    value: '門',
    position: 'B37:C38'
  },
  door2: {
    value: '門',
    position: 'V37:W38'
  },
  lectern: {
    value: '講台',
    position: 'I37:N38'
  }
}

/*======================
  執行
========================*/

// 讀取 Excel 匯出 worksheets
const workbook = XLSX.readFile('./excels/conference.xlsx', { cellStyles: true })
const worksheet = workbook.SheetNames
const _seats = workbook.Sheets[worksheet[0]]
const _students = workbook.Sheets[worksheet[1]]

// 處理 worksheets（過濾 _seats to array；轉換 _students to JSON）
const keys = Object.keys(_seats)
const seats = keys.filter(key => {
  return (key.charAt(0) !== '!' && _seats[key].v)
})
const { content } = xlsxToJson(_students)

// 處理使用者需求並合併為 worksheet object
const template = genTemplate(config, seats, display.length - 1)
const data = genData(content, display)
const output = merge(template, data)

// 範圍
const ref = getRange(output)

// 生成 workbook
const wb = {
  SheetNames: ['sheet_1'],
  Sheets: {
    'sheet_1': {
      '!ref': ref, // 若無範圍，則只會產出空表而已
      ...output
    }
  }
}

// 生成 Excel 文件
XLSX.writeFile(wb, './excels/output.xlsx')
```

## 功能函式 - func.js
```javascript
// 取得 cell 行列如 A1 => {A,1}
const getColAndRow = (cell) => {
  // col 位數
  let index = cell.split('').findIndex(value => {
    return !isNaN(value)
  })

  // 如 A21 中的 A
  const col = cell.substr(0, index)
  // 如 A21 中的 21
  const row = parseInt(cell.substr(index))

  return { col, row }
}
// 解析 xlsx 返回 {headers, data}
const xlsxToJson = (worksheet) => {
  const _headers = {}
  const content = []
  const keys = Object.keys(worksheet)

  keys
    // 過濾非'!'開頭的 keys
    .filter(k => k.charAt(0) !== '!')
    .forEach(k => {
      const { col, row } = getColAndRow(k)
      // 當前 cell 的值
      const value = worksheet[k].v

      // 第一列為 headers
      if (row === 1) {
        _headers[col] = value
        return
      }

      // 解析 data
      const index = row - 2

      if (!content[index]) {
        content[index] = {}
      }

      content[index][_headers[col]] = value
    })

  // 
  const headers = Object.values(_headers)

  return { headers, content }
}
// 畫外框線，如 A1:C3 的正方形外框
const drawFieldsBorder = (start, end) => {

  const startCR = getColAndRow(start) // {A, 1}
  const endCR = getColAndRow(end) // {C, 3}
  const startCol = startCR.col // A
  const startRow = startCR.row // 1
  const endCol = endCR.col // C
  const endRow = endCR.row // 3
  const startChar = start.charCodeAt() // 65
  const endChar = end.charCodeAt() //67
  const result = {}

  /* 外框選項 
  {
    A:'left', 
    C:'right', 
    '1':'top', 
    '3':'bottom'
  }
  */
  const opts = {
    [startCol]: 'left',
    [endCol]: 'right',
    [startRow]: 'top',
    [endRow]: 'bottom'
  }

  for (let c = startChar; c <= endChar; c++) {
    for (let r = startRow; r <= endRow; r++) {
      const col = String.fromCharCode(c)
      const row = r
      const cell = col + row
      const border = {}

      if (opts[col]) {
        border[opts[col]] = { style: 'medium' }
      }

      if (opts[row]) {
        border[opts[row]] = { style: 'medium' }
      }

      // 如僅有單一欄則需畫左邊框線
      if (startCol === endCol) {
        border['left'] = { style: 'medium' }
      }

      // 無邊框 cells 就不需要 s 屬性
      if (Object.keys(border).length === 0) {
        result[cell] = { v: '' }
      } else {
        result[cell] = { s: { border }, v: '' }
      }

    }
  }
  return result
}
// 取得 worksheet 範圍 返回如 'A1:G8'
const getRange = (worksheet) => {
  const cellsArr = Object.keys(worksheet)

  // 行
  const cols = cellsArr.map(cell => {
    return getColAndRow(cell).col
  }).sort()

  const startCol = cols[0]
  const endCol = cols[cols.length - 1]

  // 列
  const rows = cellsArr.map(cell => {
    return getColAndRow(cell).row
  }).sort((a, b) => a - b)

  const startRow = rows[0]
  const endRow = rows[rows.length - 1]

  return `${startCol + startRow}:${endCol + endRow}`
}
// 合併 template 及 data
const merge = (template, data) => {
  const dataCells = Object.keys(data)
  const error = []

  dataCells.forEach(cell => {

    if (!template[cell]) {
      error.push(data[cell])
    } else {
      template[cell].v = data[cell].v
    }

  })

  if (error.length) return error
  return template
}

module.exports = {
  getColAndRow,
  xlsxToJson,
  drawFieldsBorder,
  getRange,
  merge
}
```

## 生成 template 及 data 函式 - generate.js
```javascript
const { getColAndRow, drawFieldsBorder } = require('./func')

/* 解析 JSON 生成 data 如
{
  P8: { v: 1 },
  P9: { v: '喬傑立娛樂' },
  P10: { v: '歌手' },
  P11: { v: '孫協志' },
  T6: { v: 2 },
  T7: { v: '三立藝能' },
  T8: { v: '演員' },
  T9: { v: '紀言愷' },
}
*/

const genData = (json, display) => {

  const data = json
    .map(item => {
      const { col, row } = getColAndRow(item.position)

      // 返回依 display 順序顯示的欄位
      return display.map((key, index) => {
        const position = col + (row + index)

        // 返回 worksheet object
        return { [position]: { v: item[key] } }
      })
    })
    // 降維
    .reduce((prev, curr) => {
      return prev.concat(curr)
    })
    .reduce((prev, curr) => {
      return Object.assign({}, prev, curr)
    })

  return data
}

/* 生成 template 如
{
  E2: { v: '第70屆金鐘獎頒獎典禮工作會議', s: { font: [Object] } },
  J4: { v: '會議室座位表', s: { font: [Object] } },
  B37: { s: { border: [Object], font: [Object] }, v: '門' },
}
*/

const genTemplate = (config, seats, cellsPerSeat) => {

  // 畫出配置
  const c = Object.keys(config)
    .map(key => {
      const item = config[key]
      const start = item.position.split(':')[0]
      const end = item.position.split(':')[1]

      // 門及講台的儲存格畫框線
      if (key === 'door1' || key === 'door2' || key === 'lectern') {
        const fieldsHasBorder = drawFieldsBorder(start, end)
        fieldsHasBorder[start].v = item.value
        fieldsHasBorder[start].s['font'] = { sz: '16', bold: true }
        return fieldsHasBorder
      }

      return {
        [start]: {
          v: item.value,
          s: {
            font: {
              sz: '24',
              bold: true
            }
          }
        }
      }

    })
    .reduce((prev, curr) => {
      return Object.assign({}, prev, curr)
    })

  // 畫出座位
  const s = seats
    .map(start => {
      const { col, row } = getColAndRow(start)
      const end = col + (row + cellsPerSeat)

      return drawFieldsBorder(start, end)
    })
    .reduce((prev, curr) => {
      return Object.assign({}, prev, curr)
    })

  // 返回合併後的配置及座位
  return Object.assign({}, c, s)
}

module.exports = {
  genTemplate,
  genData
}
```

# 結論
實作過程接連遇到許多小插曲，例如原先以 js-xlsx (community version) 處理，不過程式碼寫到一半發現無法寫入框線格式，怎麼辦？棄用？而翻了 github 上的 issue，更加證實自己目前遇到的窘境。當然後面只能另外尋找套件，也發現了 xlsx-style 支援寫入，不過僅支援簡單格式上的操作，如字體大小、框線、儲存格填滿，不過現階段也夠用了，相當符合本案需求。

第二個問題發生在處理及轉換資料的階段，文中所述想法是分為格式及資料兩部分產生後合併，但當我寫完格式部分後，忽然有莫名的念頭閃過，其實格式可以不用自己產生阿！畢竟一開始使用者就有空白座位表的資訊，只要讀取檔案時一併匯入就好 `XLSX.readFile('Excel.xlsx', { cellStyles: true })`

WTF～但我做完了耶！哎呀～不過換個想法，假如目前情況改成使用者沒有空白座位表的資訊時，勢必也得自己從無到有，這樣子練習一下也好，總算舒服許多。

# 參考
[sheetjs](https://github.com/SheetJS/sheetjs)
[xlsx-style](https://www.npmjs.com/package/xlsx-style)
[在 Node.js 中利用 js-xlsx 处理 Excel 文件](https://scarletsky.github.io/2016/01/30/nodejs-process-excel/)