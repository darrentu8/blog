---
layout: post
title: web工程師應該要知道瀏覽器架構演變及渲染機制
image: https://img.technews.tw/wp-content/uploads/2017/12/29164436/browsers-1265309_1280.png
accent_image: 
  background: url('https://www.pbs.org/wgbh/nova/media/images/bfcppfu.width-800.png') center/cover
  overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
invert_sidebar: false
# related_posts:
#   - example/_posts/2017-11-23-example-content-ii.md
#   - /example/2012-02-07-example-content/
sitemap: false
---

> 「在未來，瀏覽器會變得越來越強，以後我們可以在瀏覽器做越來越多事。」

身為常與瀏覽器共舞的 Web 工程師，尤其是 Frontend Engineer，如果瀏覽器突然消失了，應該也等同於要失去飯碗了 😂 而上面這句話我想大家應該或多或少都有聽過，不過你知道所謂變得越來越強是指什麼嗎？瀏覽器又是為什麼會變得更強呢？透過這篇文章，我想淺淺得說明瀏覽器的架構演進史，從過去，到現在，再看向未來。身為 Web 工程師，瀏覽器的關係與我們密不可分，但除了學會使用它以外，如果能去理解背後的運作模式，我認為是百利而無一害的，除了學會根據背後運作模式去建構更好的 web 應用以外，也可以提早洞察到未來可能的發展，領先其他人一步去探索更多可能性。<br>
這篇文章會透過簡易系統架構的角度去看瀏覽器的演進，process 程序（對岸用語為進程），thread 執行緒（對岸用語為線程）是兩個必備的知識點，如果還不太了解兩者概念的讀者可以參考我之前的文章。<br>

## 舊石器時代：Single Process 瀏覽器時期
首先這邊假設各位讀者都已經了解 Process 與 Thread 的基本概念。<br>
其實在 2007 年以前，市面上的瀏覽器基本上都是 Single Process 單一程序的架構的。<br>

![](https://miro.medium.com/max/3160/1*aCbpvQk_djR_W8sxzF8ODw.png)

這代表著瀏覽器的所有功能模組都是運行在同一個 Process 裡的，所謂的功能模組指的是 JavaScript 執行環境、網路、瀏覽器擴充功能（插件）、渲染引擎…等，由上面的架構圖也可以看出不同的功能模組可能運行在不同的 thread 中，然而這種架構衍伸出了幾個明顯的問題：<br>
不穩定性<br>
不流暢性<br>
安全性問題<br>
不穩定性<br>

簡單來說就是，單一程序的瀏覽器中，其中一個功能模組如果出問題壞掉了，會導致整個瀏覽器的崩潰。早期的瀏覽器有很多功能並沒有原生支援，例如影片播放、遊戲引擎…等，需要透過插件 (Plugin) 來協助實現，但偏偏 Plugin 又很容易出問題，當插件運行崩壞時，也會導致瀏覽器的崩壞。JavaScript 執行環境也是一樣的道理，如果程式碼過於複雜或是效能炸裂，也會導致渲染引擎的崩壞，進而導致整個瀏覽器的崩壞。<br>
不流暢性<br>
從上面的 Single Process Browser 架構圖可以看到，有一個叫 Page Thread 的執行緒，它的工作包山包海，從頁面的渲染、頁面的顯示、JavaScript 的執行環境還有插件的運行都是它的職責。不過由於是單一執行緒，意味著同一時刻只有一個功能可以執行，所以如果遇到一段跑很慢或是無限循環的 JS 程式碼，例如：<br>

```
function stupidFunc() {
  while(true) {
   console.log('HI');
  }
};
stupidFunc();
```

在 Single Process 的瀏覽器架構裡，上面這段無限循環的程式碼會獨佔整個 thread 的資源，導致其他模組永遠沒有機會被執行到，因為瀏覽器所有的頁面都是靠這個執行緒運行的，所以這樣的狀況會讓整個瀏覽器失去反應或是變得卡頓。<br>
另外相信大家也知道，記憶體的空間也是影響瀏覽器效能的重大因素，而 Single Process 架構的瀏覽器有一個明顯的缺點就是往往不能完全地回收記憶體，導致使用時間越久記憶體佔用就越來越高，導致頁面慢慢變得不流暢與卡頓。<br>

## 安全性問題
因為我本身對資安並不是那麼的了解，所以這邊只能簡單說明可能發生的狀況。<br>
以往 Single Process 架構的瀏覽器沒有實作合理的安全環境（例如等等會提到的 sandbox），因此透過 Plugins 或是 Script 是有可能可以獲取系統資源與權限的，而這個狀況不需多說，自然會衍伸出許多安全性的問題，注入病毒、盜竊帳號密碼都是可能發生的資安攻擊。<br>
以上就是三個 Single Process Browser 最大的缺陷，如果隨便一個頁面壞掉會使整個瀏覽器崩潰的話，確實蠻恐怖的，萬一其中一個頁面是你花了一個星期撰寫，要給老闆看的企業報告，你應該也會崩潰吧 😂 所幸現在的我們已經不需要受到這種瀏覽器架構的荼毒與威脅，讓我們接著看看下個階段瀏覽器的架構做出了什麼樣的改變吧！<br>

## 新石器時代：Multi Processes 瀏覽器時期
接下來的架構我都會以 Chrome 這個瀏覽器的架構來說明，並不是單純因為它是我最喜歡的瀏覽器（雖然是事實沒錯 😂），更是因為它是第一個提出 Multi Processes 多程序架構的瀏覽器。Chrome 在 2008 年發表了 Multi Processes 的架構，當時的瀏覽器架構圖大致上如下:<br>
![](https://miro.medium.com/max/700/1*3Tx4AkFEIG700D4--KBBpQ.png)

這個版本的架構與上一個 Single Process 的架構差別在於獨立出了 Renderer Process 與 Plugins Process，也就是渲染程序與插件程序，不同 Processes 間需要通過 IPC 來溝通（也就是上圖的虛線），Plugins Process 顧名思義就是專門運行瀏覽器插件、外掛的程序，Renderer Process 又被稱作渲染引擎，主要負責頁面的渲染，包含 Parse、Render、JS 的執行等工作，關於現今瀏覽器渲染引擎的運作機制，稍後會有更加詳細的介紹。<br>

### 接著來看看改成這樣的架構後，是否有效地解決了 Single Process 瀏覽器的三大問題：<br>

-**不穩定性的問題：** 因為各個 Process 是互相隔離的，也就是說如果 Plugin 崩壞，或是頁面失去響應了，只會影響到它們當前所運行的 Process，並不會像以前一樣牽一髮而動全身，導致整個瀏覽器的癱瘓，因此有效解決了不穩定性的問題。<br>

-**不流暢性的問題：** 這個架構下拆分出 Renderer Process 的特點是，瀏覽器中每一個 Tab 都會運行在獨立的 Renderer Process 上。雖然遇到上面無限循環的 Script，ㄧ樣會造成頁面失去反應，不過現在會影響的就只有當前的頁面（Tab）而已，其他的頁面因為是運行在不同的 Process，因此仍能正常運作。再來是前面提到記憶體有可能無法完全回收或發生 memory leak 的問題，儘管現在頁面失去反應了，當你關閉它時，整個 Renderer Process 也會被關閉，這個 Process 所佔用的記憶體會被系統完整的收回，解決了過往 memory leak 的問題。因此這樣看下來，不流暢性的問題也得到了大大的改善。<br>

-**不安全性的問題：** 在 Multi Processes 架構下的瀏覽器，不僅獨立出了不同的程序，還引用了 「Sandbox 沙盒」的機制，使 Plugin Process 與 Renderer Process 運行在沙盒中。（可以把 Sandbox 想像成一個安全的隔離環境，在裡面運行的程式無法獲取外部的數據，放到瀏覽器中，就是指無法獲得系統的資源，想當然，連讀取都禁止了，寫入當然也是禁止的），即使有惡意的程式碼，也只會運行在 Sandbox 中，無法突破 Sandbox 影響外部瀏覽器的系統，這也就解決了之前提到可能會發生的一些資安問題。<br>

## 現代：更加豐富的 Multi Processes 瀏覽器架構

（可能有讀者會疑惑，怎麼從新石器時代馬上就跳到現代來了 😂，這點就放過我不要計較了吧 🤣）<br>
大家都知道 Chrome 是 Google 開發的，進化的速度也是非常快的，現在的 Chrome 瀏覽器仍然以多程序架構為基礎，不過獨立出了更多的 Processes，大致上如以下的架構圖：<br>
![](https://miro.medium.com/max/700/1*bmUbnAhxH9C7raz-bwEOxw.png)

從上圖可以看出，原本運行於 Browser Process 中的網路資源操作與 GPU 操作變成 Network Process（主要負責網路資源載入） 與 GPU Process（負責一些頁面的繪製與運算） 被獨立了出來，而獨立出 Process 的好處也在稍早提過了，除了解決不穩定性、不安全性與不流暢性以外，也可以擁抱 process parallel 運行帶來的性能提升。<br>
那麼方便的話，我把所有操作都獨立出一個 Process 不就好了？<br>
這個思考方向並沒有錯，不過想擁有收穫總是得付出一些成本或代價，獨立出更多 Processes 的缺點主要有：<br>
更高的記憶體佔用<br>
系統架構會變得更加複雜（要考慮不同 process 間的溝通）<br>
因此這其實是個非常複雜的問題，也是 Chrome 團隊一直在優化的方向。<br>
未來世界：SOA 服務導向架構瀏覽器<br>
面對上述高記憶體佔用與架構複雜的問題，Chrome 團隊非常努力想找到一個彈性的解決方案，在 2016 年 chrome 提出了以 SOA (Services Oriented Architecture) 服務導向架構為基礎的新架構：<br>

![](https://miro.medium.com/max/700/1*r05OaXt93q9ZcWo2mV1dmw.png)

也就是希望各個在 browser 中運行的 program 可以以服務 (service) 的角度被拆分或聚合，並運行在獨立的 Process 中，Processes 間透過 IPC 來溝通，讓系統架構實現高內聚、低耦合、易擴展與易維護的特性。<br>
而關於 SOA，相信很多讀者會聯想到 Microservices，如果想了解兩者區別的讀者可以參考這篇文章。<br>
另外 Chrome 還提供一個我認為非常厲害又彈性的架構，我們都知道設備的性能差異是很大的，如果在低階的設備上，例如老舊的手機，這樣的瀏覽器架構似乎不是低階設備承受得起的。在遇到性能較高的設備時，Chrome 會採用上面所說拆成多個 Processes 的架構去增強穩定性與效能，但是如果是在較低階老舊的設備上運行時，Chrome 則會自動採用多個服務合併成單一 Process 的方式來節省記憶體耗費，來達到更彈性的架構。<br>

> General idea is that when Chrome is running on powerful hardware, it may split each service into different processes giving more stability, but if it is on a resource-constraint device, Chrome consolidates services into one process saving memory footprint. — developer.google.com

其實 Chrome 現在就已經在朝著這個方向前進了，只不過這必定是一個緩慢的過程，因此我將它放在「未來」這個時間線裡。不過值得一提的是，Chrome 的更新是漸進式的，也就是說未來服務會慢慢的改進與更新，我們將會慢慢享受到更多更新的服務，未來不管是 AR/VR、遊戲引擎，甚至是 AI，都可以在瀏覽器身上看到無限可能，身為 Web 開發者，我想這是一件會讓我們都感到期待與熱血沸騰的一件事。<br>


## 現今瀏覽器渲染引擎的運作機制

原本這篇文章應該在上面簡單介紹完瀏覽器架構演進後就該告一段落了，不過剛好在這篇文章中一直提到了 Renderer Process，它也就是你常常會聽到的「渲染引擎」，身為 Web 工程師，又甚至像我一樣是更偏向前端開發的工程師，渲染引擎想必是你最常聽到，也最在乎的一個程序，因為它負責處理頁面的渲染流程，負責將 HTML、CSS、JavaScript 三劍客變成我們看到的頁面，然而這中間發生的過程你是否都了解了呢？如果你還不是很了解也沒關係，我想藉著文章的最後一個段落（雖然應該會是最長的一個段落），簡單複習一下渲染引擎的運作機制，讓各位開發者們可以更了解網頁究竟是怎麼被顯示出來的。<br>
（底下ㄧ樣會以 Chrome 瀏覽器作為示範）<br>
每個 Tab 都會產生一個獨立的 Renderer Process<br>
這點在看完上面的架構演進史後，讀者應該都能了解了，這也就是為什麼其中一個 Tab 的網頁掛掉之後，你仍然可以繼續正常使用其他 Tab 的網頁，因為不同的 Renderer Process 是不會互相影響的。讀者可以點擊 Chrome 瀏覽器的右上角，點擊「更多工具」->「工作管理員」<br>

### 會出現類似作業系統工作管理員的介面

在了解瀏覽器架構演進史後，看到瀏覽器運行著這麼多 Process 應該就不會被嚇到了，除了 Browser Process 以外， GPU Process、Network Process 等不同程序都可以在工作管理員看到，再來就是各個不同的 Tab 也會運行在獨立的 Renderer Process 裡。<br>
Per-frame Renderer Processes — Site Isolation<br>
這是 Chrome 在 2018 年左右引進的新特色，Same Origin Policy 同源政策是 web 裡一個很普遍的安全模型，理論上不同源的網站在未經授權下是要不能存取到彼此的資源的，不然會產生許多安全性問題。而要做到把兩個不同來源的網站徹底分開，獨立 Process 成為最有效率也最根本的一個方式，因此在 Chrome 中，實現了每個 Tab 都獨立一個程序的機制，甚至在網頁中嵌入不同來源的 iframe，該 iframe 也會運行在不同的 Renderer Process 上：<br>

![](https://miro.medium.com/max/700/1*vfFQIhgox9px6rPUoj8_fA.png)


不過讀者要知道採用這種方式不僅僅是獨立出不同 Renderer Processes 這麼單純而已，它也徹底改變了 iframe 與網頁間的溝通方式，對於 Chrome 團隊來說絕對是一個很大的里程碑。
不過眼尖的讀者可能會發現，有些頁面顯示為「子頁框」，並且沒有獨立的 Process ID，這是為什麼呢？<br>
Process Per Site Instance<br>
雖然預設狀況下，每個 Tab 都會是獨立一個 Renderer Process，不過在某些「Same Site」的狀況下，Chrome 預設會將同源的網頁運行在同一個 Process 中。這裡的 Same Site 指的是 Protocol ㄧ樣、root domain 一樣就符合了（跟一般 Same Site Cookie 或同源政策的標準都不太一樣，讀者別搞混囉，請把它們當作完全不同的觀念！），也就是說<br>

```
https://kylemo.com
https://www.kylemo.com
https://www.kylemo.com:3000
```

都會被視為 Same Site，另外還有一個條件是「必須從一個頁面打開另一個 Same Site 頁面」 ，例如透過 <a> tag 或是 window.open 等方法，瀏覽器就會將新開啟的 Same Site 頁面與原本的頁面運行在同一個 Process 中。其實仔細想想這樣的特性是合理的，畢竟有些 Same Site 的網頁，是有共享 JavaScript 執行環境的需求的，另外節省記憶體也是這個特性的優勢之一。<br>

## 渲染流程
Renderer Process 主要負責的就是頁面的渲染流程，這邊還是簡單說明下，在瀏覽器輸入 URL 並按下 Enter 後，搜尋列會先對輸入做解析，判斷使用者輸入的是 URL 還是搜尋關鍵字，並透過 Network Thread 或是 Network Process 去做資源請求，並根據回傳的 Content-Type 來決定下一步要交給誰做，如果是回傳的是 HTML，就會準備交由 Renderer Process 進行渲染流程。<br>

![](https://miro.medium.com/max/700/1*57F8N58JxoDtgnaGn0neKA.png)

讀者可以透過 CURL 來看看 Response Header 中的 content-type<br>
基本上如果你上網查詢瀏覽器渲染流程，應該都會看到跟下面這張差不多的圖片：<br>

![](https://miro.medium.com/max/700/1*ex4bOPRpLwwNMmLm0v18Yg.png)

如果有看過我之前關於前端效能優化方式統整的文章「今晚，我想來點 Web 前端效能優化大補帖」的讀者應該會對以上流程有點印象，大致上網頁的渲染流程為：<br>

- 讀取 HTML 後生成 DOM Tree
- 讀取 HTML 中的 CSS Link Tag 生成 CSSOM Tree
- DOM Tree 與 CSSOM Tree 共同生成 Render Tree
- 根據 Render Tree 生成 Layout Tree，負責各元素大小與位置的計算
- 最後 Paint 畫面

但是，其實瀏覽器在 Layout 之前與 Paint 之後的過程還做了一些事。<br>
而這些事是一般在網路查詢渲染流程時不太會被介紹到的，往往被開發者所忽略的部分，雖然主要是因為瀏覽器幫我們做好了，不知道這些事也不會影響到開發者的開發流程，不過如果對整個渲染流程有更完整的理解，一定是利大於弊的。<br>

### Layer 分層
為了方便實現一些瀏覽器上的複雜效果例如頁面滾動或是三維空間的排序，瀏覽器會根據 Layout Tree 產生 Layer Tree<br>

![](https://miro.medium.com/max/700/1*cwAlMho6lZ6Mycj-iFap8A.png)

如果使用過 Adobe Photoshop 的讀者應該都知道，我們最後產出的圖，實際上就是由許多圖層疊加在一起的，而瀏覽器上的頁面其實也是ㄧ樣的，這邊讀者還不需要去理解瀏覽器是怎麼安排哪些節點應該要變成一個獨立的圖層的，這是一個非常複雜的技術，各位讀者目前只需要知道「瀏覽器上的頁面也是由許多圖層疊加在一起的」就足夠了，如果真的有興趣了解背後機制的讀者可以再自行研究。<br>

![](https://miro.medium.com/max/700/1*0M-n6IF38nNvztBa97yOEw.png)

在將頁面分層以後，就要對每個圖層進行繪製了，不過我想大部分的人都會以為真正的繪製行為就是在 Renderer Process 中的 Main Thread 執行的，不過其實這個階段做的只是「產生繪製指令」而已，所謂的繪製指令就是告訴瀏覽器在哪個座標要繪製線或是繪製幾何圖形等簡單指令的集合，後續的操操作則是會在生成繪製指令後轉交給 Renderer Process 中的另一個執行緒 — Compositor Thread 來接棒。<br>

## Compositing
現在瀏覽器已經獲得了渲染頁面所需要的資訊了，例如 DOM Tree 的結構、每個節點的 Style、節點在頁面中的幾何位置，還有剛剛提到的各個圖層的繪製指令與疊加順序…等，因此已經準備好可以繪製到頁面上了。<br>
首先讀者得先了解一個專有名詞 — rasterize 柵格化，也就是把上述頁面資訊轉變成 pixels 顯示在螢幕上。如果不考慮分層的話，最符合常理的狀況就是一次 rasterize 出現在 viewport 裡的部分，如果使用者滑動了頁面，再去 rasterize 新出現在 viewport 的部分。在舊的 Chrome 架構中的確是採用這種方式，不過隨著瀏覽器進步，現在採用的是更複雜，但是整體效能更好的流程，也就是標題的 compositing，中文可以稱作合成。<br>
先前已經提到瀏覽器根據 Layout Tree 生成了 Layer Tree，compositing 的概念就是各個 Layer 分別做 rasterize，並在 Compositor Thread 把各個經過柵格化的圖層組合起來，此時如果有頁面滾動事件產生，因為每個 Layer 已經經過柵格化，所以要做的事就是合成一個新的 frame 就好。<br>
不過每個 Layer 的大小不ㄧ樣，有些 layer 可能幾乎包含整個頁面的大小，會導致效能受到影響，因此 Compositor Thread 會再將一個 Layer 切分成更小的單位 — tile(柵格化的最小單位)，並把這些 tiles 送到真正負責柵格化的 raster threads，raster threads 將 tiles rasterize 後會存放到瀏覽器的 GPU 儲存空間裡。Tiles 被柵格化之後，<br>Compositor Thread 會匯集被稱作 draw quads 的資訊來產生 Compositor Frame。<br>

![](https://miro.medium.com/max/700/1*6dleKIcII0jVrm80RouMMg.png)
Compositor Frame 接著會藉由 IPC 被送到 Browser Process，最後送到 GPU 去顯示到畫面上。<br>

![](https://miro.medium.com/max/700/1*vpRqOs1dCtxluNu8aCv_Nw.png)

所以看到這我們應該可以把原本的渲染流程圖修改成下面這樣：<br>

![](https://miro.medium.com/max/700/1*ZwAgbaV5mi3DmVRhiefY7A.png)

## Reflow & Repaint & Compositing

最後來談談頁面更新造成的 Reflow 回流、Repaint 重繪與 Compositing 合成，這是三個與頁面效能高度相關的概念。<br>
### Reflow 回流
指的是瀏覽器為了重新渲染部分或全部的 document 而重新計算 Render Tree 中元素的物理屬性，如位置、大小的過程。<br>
觸發條件為改變一些元素的幾何樣式，例如 height、width、margin 或是排列的方式等等。<br>
### Repaint 重繪
將計算結果轉為實際的像素，畫到畫面上。<br>
如果只改動元素的顏色、背景圖等不需要重新計算頁面元素 layout 的樣式，就只會從 Repaint 開始觸發，跳過 Reflow 的步驟，最後再到合成階段。<br>
### Composition 合成
也就是剛剛提到的合成。<br>

![](https://miro.medium.com/max/700/1*AFOOcrE0IOgkdRAOjiow8g.png)
頁面更新簡易流程<br>

這邊需要了解的是，如果觸發了渲染流程的某個階段，那麼其之後的階段就也會被觸發。透過文章前面的渲染流程內容，我們大致可以推斷出，Reflow 跟 Repaint 分別會觸發渲染流程中的哪些步驟<br>

(圖片的部分因為是自己畫的，很醜我先道歉QQ)<br>
了解這些以後，應該可以發現，不同的改變樣式的方式，是會觸發不同渲染流程的，因此也是效能優化的一個方向：<br>
### 1. 改變一些 Layout 相關屬性<br>
![](https://miro.medium.com/max/700/1*WvuD9L5xYQo84gLFBAW0WQ.png)

例如 width、height、position 的 left 或 top 等，瀏覽器會需要重新計算頁面元素的 layout，因此渲染流程沒有步驟可以被省略，Reflow、Repaint、Compositing 都會被觸發。<br>
### 2. 只改變一些「paint only」的屬性<br>
![](https://miro.medium.com/max/700/1*MV1SQJw7WqhXT6fta590zw.png)

例如背景圖片、字體顏色等不需要重新計算 layout 的屬性，Reflow 就不會被觸發，Layout 會被跳過，只會觸發 Repaint 之後的流程。<br>
### 3. Compositing Only
也就是更改一些不需要 Reflow 與不需要 Repaint 的屬性，例如 CSS 的<br>

```
transform: translate(xxx, yyy);
```

這種改變方式是效率最好的，除了因為它不需要經過 Reflow 與 Repaint ，只需要做合成以外，還有一個重點是「合成的運作不是在 Main Thread 進行的，而是在 Compositor Thread 與 Raster Thread，因此不會佔用 Main Thread 資源」，這也是為什麼要做 CSS 動畫會建議使用 transform 的原因。<br>

![](https://miro.medium.com/max/700/1*EDEOzsebT3qXabY6i4KKTw.png)
很醜，再次道歉QQ<br>

如何避免多餘的 Reflow 與 Repaint？<br>
避免用多個 statement 修改 style，建議使用新增或移除 class 的方式。<br>
先一次讀取完，再一起修改<br>
![](https://miro.medium.com/max/700/1*-qq29BNULK1pljvETfXv7Q.png)


從上圖範例可以看到 read 一次就馬上 write 一次的寫法會造成 6 次 Reflow，然而<br>
![](https://miro.medium.com/max/700/1*9lIC33O9tb5ha6Gq9ukPIQ.png)

這種一次 read 完全部再 write 全部的寫法卻只會造成 1 次 Reflow，在複雜的應用中也許兩種方式會造成頁面效能產生很大的差異，而背後原因則跟瀏覽器更深入的運作機制有關，這邊就不多加探討。<br>

其實優化 Reflow 與 Repaint 的方式還有許多，有興趣的讀者可以參考這裡，也可以在這裡看到各種 CSS 屬性改變會觸發的渲染階段。<br>
Reflow、Repaint 與Compositing 的簡單介紹就到這裡。其實上面說的這些也只是基礎中的基礎，可見寫 CSS 也是水很深的一門學問啊，如果想知道怎麼好好掌握 Reflow、Repaint、Compositing，來避免寫出消耗渲染性能的程式碼，建議讀者參考這篇文章。<br>

## 結語
李開復曾說過：「瀏覽器就是未來的作業系統。」這句話曾引起了廣泛的討論，持正反意見的人都有，雖然現在看下來，要取代作業系統有一定的難度，不過可以肯定的是，瀏覽器只會越變越強，能在瀏覽器上做的事只會越來越多，並且執行效能只會愈來越好。身為一個需要與瀏覽器共舞 Web 開發者，除了要知道瀏覽器越變越厲害以外，也該知道它進步的原因，背後的架構是怎麼演進的，還有它在短時間的未來可能會成長成什麼樣子，才能好好的擁抱這波技術大躍進，發展更多的可能性。希望這篇文章可以使讀者對於瀏覽器有更深的理解，也激發出對它的興趣。<br>

### References
Inside look at modern web browser (Part 1, 2, 3, 4)<br>
浏览器工作原理与实践<br>
Chrome Site Isolation<br>
Rendering Performance<br>
Reflow 和 Repaint 引發的性能問題<br>
DOM Performance<br>