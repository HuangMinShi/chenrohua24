---
title: '理解 Event Loop'
date: 2020-10-27 17:54:25
tags:
---
# Target
理解並敘述 Event Loop 是什麼？
<!-- more -->

# Contents
* JavaScript in browser 
* JavaScript Engine 
* Event Loop 
* Zero delays
* Conclusion
* References

# JavaScript in browser
網路應用程式包含了很多技術，並且共同組成 JS Runtime Environment 。除了由 JavaScript Engine 負責解析程式碼之外，瀏覽器也提供了許多 Web API 像是 DOM, setTimeout 來與使用者進行互動，以及等等會提到的 Event Loop, Event Queue and Event Table。而透過 Event Loop 搭配非同步的方式進而達成 JavaScript 能同時處理多個工作的錯覺。

# JavaScript Engine
>it’s job is to go through all the lines of JavaScript in an application and process them one at a time

JavaScript Engine 所做的事情，是去遍歷應用程式當中每一行程式碼，而且一次執行一段。這同時也注定了 JavaScript 是單線程（single threaded）的程式語言。

那麼 JavaScript Engine 如何做到一次只執行一段？它利用了 Call Stack 去記錄目前程式執行的部分。

Call Stack 是一種具有先進後出（FILO, First-in-Last-out）性質的有序串列，就像搭電梯一樣，第一個進電梯的人會最後出電梯，最後進電梯的則最先出來。因此程式執行時會層層往上疊，而執行結束後層層跳離（pop off）。舉個例子：

```javascript
/* main.js */

const one = () => {
  console.log('1')
}
const two = () => {
  one()
  console.log('2')
}

two()

/* Results:
* => 1
* => 2
*/
```

流程：
1. 執行全域環境的程式 main.js
2. 調用 two
3. 引起 one 被調用
4. 執行 one ，console 輸出 '1'，此時 one 無其他程式碼，執行完畢從 Call Stack 跳離（pop off）
5. 執行 two ，console 輸出 '2'，此時 two 無其他程式碼，執行完畢從 Call Stack 跳離（pop off）
6. 最後 main.js 也沒有其他程式碼可執行，執行完畢從 Call Stack 跳離（pop off）

[Javascript Call Stack demo](https://www.youtube.com/watch?v=aqI8Aovw4Po)

上述其中一個程式碼片段若需要執行非常久的時間，則後面的程式碼就會等待執行，好像卡住一樣，這樣的情況稱作阻塞 （blocking）。

例如請求資料的 AJAX request，如果是以同步方式執行的話，因為必須等待伺服器回應，才能從 Call Stack 跳離（pop off）。還沒接收回應前整個瀏覽器頁面停滯，即使點了其他 button，瀏覽器就只是掛在那裡，不會有任何反應。

# Event Loop
如何避免阻塞？

>JavaScript provides a mechanism and it's via asynchronous callback functions.

非同步搭配 Callback Function，常見於將函式當成參數傳進另外一個函式之中，而這些被傳進的函式並沒有馬上被執行，而是等待條件觸發後才執行。

底下以 setTimeout 為例。setTimeout 函式是瀏覽器提供的 API ，不是 JavaScript Engine 本身的功能，當調用 setTimeout 函式時，瀏覽器會提供一個計時器用來計時秒數。
```javascript
/* Within main.js */

const one = () => {
  console.log('1')
}
const two = () => {
  setTimeout(one, 5000)
  console.log('2')
}

two();

/* Results:
 * => 2
 * (And 5 seconds later)
 * => 1
 */

```
* ...（如上個例子所述）
* 執行 two，並調用 setTimeout，setTimeout 進入 Call Stack
![](https://i.imgur.com/UEtUTKQ.jpg)

* 執行 setTimeout，Call Stack 向 Event Table 註冊 Callback Function （one），並同時記錄觸發事件，也就是等待 5 秒。
![](https://i.imgur.com/n3ryqFK.jpg)

  Event Table 就是個註冊站，Call Stack 讓 Event Table 註冊一個特定函式（在這裡為 one）以及觸發的事件（wait 5000 ms）。當事件發生，Event Table將對應的函式移到 Event Queue 等待被調用。而 Event Queue 就像是個暫存區，依序存放著等待被調用至 Call Stack 的函式。並且具有先進先出（FIFO, First-in-First-out）的性質，就如同排隊，先排到的先買到。

  那麼何時會被調用至 Call Stack ？

  JavaScript Engine 利用 Event Loop 機制來解決何時會被調用的問題。這個處理程序將不斷地檢查 Call Stack 當中是否已清空，若清空，則檢查 Event Queue 是否有等待被調用的函式，如果有，則將第一個等待被調用的函式移至 Call Stack 執行。如果沒有，仍不斷地反覆檢查 Call Stack 及 Event Queue。這個監控的過程就是 Event Loop，就像 While Loop 一樣不停的運作。接續上述例子：

* setTimeout 執行完畢跳離（pop off）Call Stack，此時 Call Stack 將不再有阻塞情形，所以 two 繼續執行下一行， console 輸出 '2'
![](https://i.imgur.com/zt6zG4x.jpg)

* two 執行完畢，跳離（pop off）Call Stack，而後 main.js 也是一樣。此時 Event Table 仍於背景持續監控是否有任何已註冊事件發生
![](https://i.imgur.com/UCRDzxX.jpg)

* 5 秒鐘過去，等待 5秒的事件完成，Event Table 將剛剛註冊的 Callback Function （one）移至 Event Queue
![](https://i.imgur.com/MiLqppS.jpg)

* Event Loop 也是不斷地在監控 Call Stack 及 Event Queue。一旦確認 Call Stack 為空後，且 Event Queue 也有一個等待被調用的函式，於是將這個函式 one 轉移至 Call Stack 執行。one 執行，console 輸出 ‘2’，並且執行完畢，跳離 Call Stack。
![](https://i.imgur.com/JiDO1bR.jpg)

[JavaScript Event Loop with setTimeout demo](https://www.youtube.com/watch?v=lo3YA_-8MDM)

# Zero delays
如果上述例子使用 setTimeout 並且希望在 0 秒後馬上執行，會是一樣的結果嗎？

```javascript
/* Within main.js */

const one = () => {
  console.log('1')
}
const two = () => {
  setTimeout(one, 0)
  console.log('2')
}

two();

/* Results:
 * => 2
 * (And 0 seconds later)
 * => 1
 */

```
是的，即使 0 秒，它一樣會先將 Callback Function 註冊到 Event Table 中，並計時 0 秒，時間到，再把 Callback Function 放到 Event Queue，藉由 Event Loop 的機制等到所有 Call Stack 的內容清空後，而且 Event Queue 當中前面無其他的待處理事件時，才會立即執行這個 Callback Function。

因此 setTimeout 的時間表示的是執行的最少等待時間，並非保證會被執行的時間。

# Conclusion
Event Loop 是一種機制，不斷地監控 Call Stack 及 Event Queue，如同一個無限迴圈。一旦 Call Stack 清空，則把 Event Queue 裡，第一個待執行函式移到 Call Stack 執行。而透過這個機制，搭配非同步方法進而快速切換並處理多項工作，而不再只是卡著不動了。

# References
[並行模型和事件循環 - MDN](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/EventLoop)
[What is the JavaScript event loop?](http://altitudelabs.com/blog/what-is-the-javascript-event-loop/)
[Philip Roberts: What the heck is the event loop anyway?](https://2014.jsconf.eu/speakers/philip-roberts-what-the-heck-is-the-event-loop-anyway.html)

