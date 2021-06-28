---
layout: post
title: 利用Google Sheet 當你的簡易表單資料庫！
image: https://img.ltn.com.tw/Upload/3c/page/2016/08/03/160803-25553-1.png
# accent_image: 
#   background: url('https://img.ltn.com.tw/Upload/3c/page/2016/08/03/160803-25553-1.png') center/cover
#   overlay: false
accent_color: '#ccc'
theme_color: '#ccc'
invert_sidebar: false
# related_posts:
#   - example/_posts/2017-11-23-example-content-ii.md
#   - /example/2012-02-07-example-content/
sitemap: false
---

這篇文章想來分享一個自己覺得非常有趣且實用的應用，就是利用 google sheet 來充當表單內容的資料庫，我想許多讀者會猜測這將會是一個麻煩的過程，可能要經過許多繁雜的設置，不過事實是…
其實非常簡單又方便
如果想一探究竟的讀者就繼續往下看吧！
## 做什麼 - Simple Demo
![](https://miro.medium.com/max/566/1*Ym4gwVh3U2NICIVrHoDmuw.gif)

## 客製化表單上傳至 google sheet
Why ?<br>
基本上大部分的事情都是先有需求，再去因應需求找出對應的解法，因此今天在說明該怎麼做到把 google sheet 當作客製化表單資料庫以前，我們得先談談這樣的方法適合應用在哪些 use cases 上。<br>
大部分企業幾乎都會有自己的形象官網，而通常官網也會與企業提供的服務分開，為單純以展示資訊為主要目的的靜態形象頁面。同樣性質的網頁還有特定活動的 campaign site，也是以展示活動資訊為主要目的，與網頁應用程式（web app）有很大的區別。不過雖然是以呈現資訊為主，這類型的網站通常會有一個例如「聯絡我們」的區塊，讓使用者填寫自己的聯絡資訊，搜集顧客資訊以挖掘更多潛在的商機，例如我最愛的國產車品牌 Luxgen 的官網：<br>

![](https://miro.medium.com/max/700/1*fj-xYxjHQZ-d9VsW6b4BHQ.png)

## Luxgen 官網
我們都知道通常 Form 的運作是在 submit 時會發一個 POST 請求到後端 API，再將資料儲存到後端資料庫裡。但是如果因為團隊人力配置因素或是開發者本身技術限制，沒辦法生出這樣一個 API 的話該怎麼辦呢？又或者即使有這樣的 API，但要運用這些資料的人反而是非工程 team 的夥伴時，請他們下 SQL query 去撈資料好像又不是那麼實際或方便（當然還是有很多能協助達成這件事的工具啦），這時候如果採用文章標題的方法似乎蠻可行的：將客製化的表單內容上傳至 google sheet 中。像這樣資料提交數量不會很大的 case，就很適合這樣的做法，google sheet 的操作介面也十分平易近人，適合所有人來使用。<br>
了解這個方法適用的 case 後，Let’s do it !<br>

## SETP 1. 建立一個 Google 表單
是的，你沒有看錯，不是 google 試算表，是 google 表單(Google Form)。在我們從客製化表單點擊 submit 後，我們會對一個 URL 發一個 POST request ，這個 URL 就是 google 表單的連結，URL 中我們會帶 query string，我們都知道 query string 會是 key-value pair 的形式，這邊的 key 會是 google 表單各欄位的 id，value 則是我們從客製化表單收集到的資訊，當我們完成這個請求後，就會自動將資料回覆到 google 表單中，接著就是 google 服務強大的地方了，它可以將 google 表單收到的回覆同步到一個 google sheet 中，如此一來 google sheet 就成為一個簡單的儲存庫了。<br>
讓我們總結一下這個方式<br>
從客製化表單按下送出按鈕後，會對 Google 表單發一個 POST 請求，以 query string 的方式將資料帶到 google 表單回覆中，再利用 google 服務相通的功能，將資料同步儲存到 Google sheet 中。<br>
不過其實除了一開始設定以外，之後的流程都可以忽略 Google 表單那層，我們可以看成從客製化表單送出後，資料就直接存到 google sheet 裡面了。<br>
![](https://miro.medium.com/max/700/1*Chuy3yR_UiRYdD3Pvdm5uw.png)

（在開始之前記得將表單權限設為所有人都可以存取喔！）
在建立 google 表單之後，就可以來建立問題欄位，注意囉，這邊的欄位要跟你網頁上客製化表單的欄位ㄧ樣喔！這邊因為 demo 方便所以隨便新增兩個欄位。<br>
接著點選回覆的 tab，有個 google sheet 的 icon，點擊創建一個與這個表單連結的試算表（其實也可以連結現有表單不需重新建立）<br>
![](https://miro.medium.com/max/700/1*Chuy3yR_UiRYdD3Pvdm5uw.png)

## google form 連結 google sheet
接下來我們需要取得兩個東西：<br>
google 表單的 ID<br>
表單中各欄位的 ID<br>
google 表單的 ID<br>
![](https://miro.medium.com/max/600/1*NmymnoIwFhup-GxUFk0Z_A.gif)

先點選右上角的「傳送」按鈕，再點選連結的傳送方式，會出現一段 URL，以 demo 裡的例子來說，URL 為<br>
<a>https://docs.google.com/forms/d/e/1FAIpQLSfniOOcWIeG4FsL14tqLoEfqz9oRYuGru8rwMvOhH2vs7UPPg/viewform?usp=sf_link</a>
我們要的 form ID 即為 /e 後的那段字串，即是<br>
1FAIpQLSfniOOcWIeG4FsL14tqLoEfqz9oRYuGru8rwMvOhH2vs7UPPg <br>
表單中各欄位的 ID<br>
我們得取得 google 表單中各欄位的 ID，才能在 POST request 時正確傳資料到該欄位，首先先進入上面拿到的用來分享表單的連結<br>

<a>https://docs.google.com/forms/d/e/1FAIpQLSfniOOcWIeG4FsL14tqLoEfqz9oRYuGru8rwMvOhH2vs7UPPg/viewform?usp=sf_link</a>

### 進入後開啟 browser 的 devtool
![](https://miro.medium.com/max/700/1*34F3KB_mQ9p4FTLuITkIZA.png)

### 在 form element 內層可以找到幾個 hidden 的 input

![](https://miro.medium.com/max/700/1*6tSElKgajvjCqNg3nA_4Rw.png)
entry.979040326 與 entry.589442071 就是我們要找的欄位 ID，注意這邊 hidden input 的順序是跟表單欄位的順序一樣的喔！<br>
## POST Request
必要的資料都拿到後就可以試著在客製化表單送出時發出 POST request 了<br>

```
import { stringify } from 'qs';
export const uploadToGoogleSheet = (formId: string, query: Record<string, unknown>): Promise<void> => {
  return new Promise((resolve, reject) => {
    fetch(`https://docs.google.com/forms/d/e/${formId}/formResponse?&${stringify(query)}&submit=SUBMIT`, {
      method: 'POST',
      mode: 'no-cors', // Google will only submit a form if 'mode' is 'no-cors'
      redirect: 'follow', 
      referrer: 'no-referrer',
    })
      .then(() => resolve())
      .catch(() => reject());
  }); 
};

// call this function when submit the form
const handleSubmit = values => {
  const FORM_ID = '1FAIpQLSfniOOcWIeG4FsL14tqLoEfqz9oRYuGru8rwMvOhH2vs7UPPg';
  const query = {
    'entry.979040326': values.data1, // 傳送使用者在表單填寫的資訊
    'entry.589442071': values.data2,
  };
  uploadToGoogleSheet(FORM_ID, query)
    .then(() => {
      // do something when submit success
    })
    .catch(() => {
      // do something when submit fail
    });
};
```

程式碼其實相當簡單，我們只需要將剛剛拿到的 form ID 帶進 URL 中，再將各欄位的 entry ID 帶入 query string 中當作 key，value 則給予使用者在客製化表單上輸入的值，POST request 成功後回到 google sheet 就會看到剛剛 submit 的值已經被記錄下來囉！<br>

## 小結
雖然說這是一個沒什麼技術含量的分享，但我認為這樣的方式十分有趣，以某些 use cases 來說也是非常適合的一種記錄表單資料的方式，google 表單雖然方便，但有時候總會因為缺乏「美感」或是獨立於我們網頁之外而使人卻步，如果你有類似的問題，不妨試試看本篇文章教的方法吧，希望這篇文章能夠幫助到螢幕前的你！