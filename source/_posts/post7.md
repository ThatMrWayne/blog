---
title: Hexo 其他功能設定
date: 2025-04-27 10:49:50
tags: [hexo]
categories: [Technical]
---

{% asset_img post7_title.png rl %}

有關 Hexo 設定分享...


<!-- more -->

部落格弄好後也過了一陣子，
最近受到前輩建議可以加些基本的功能，
至少看起來不那麼陽春...

---

## TL;DR
分享一些有關 Hexo 的設定。

---

## 留言功能

Hexo 可以串接留言的功能方式有多種，網路上已經有很多分享可以參考，  
這邊選用可以直接串接 Github issue 當作留言的方式。

### utterances

1. 首先要先到 Github 上安裝 [utterances](https://github.com/apps/utterances) app，進到連結後選自己的 blog repo 做安裝

    {% asset_img post7_1.png pst7_1 %}

2. 安裝後到 https://utteranc.es/ 完成相關設定

  - 填入剛剛安裝的 repo

    {% asset_img post7_2.png pst7_2 %}

  - 輸入想用的 label (要先確定在自己的repo有該label) 會被留言時用來開 issue 用
  - 選出現在 blog 內留言的樣式主題

    {% asset_img post7_3.png pst7_3 %}

    剛剛提到的 label 如果有留言後在 github issue 會像這樣被上 label
    {% asset_img post7_4.png pst7_4 %}

  - 複製頁面下方自動產生的程式碼到 `themes/next/layout/_partials/comments.njk`
    ```html
        <script src="https://utteranc.es/client.js"
            repo="ThatMrWayne/blog"
            issue-term="pathname"
            label="✨💬✨comment"
            theme="photon-dark"
            crossorigin="anonymous"
            async>
        </script>
    ```
    貼在 `{%- if page.comments %}` 下方

3. 到專案的 `_config.yml` 設定

    ```
      utterances:
        enable: true
    ```

4. 準備 commit

- 上面有改到 `themes/next` 的內容，  
  因為 nextT 是 submodule 指向官方 repo，這邊要先 fork repo 後，再去改 remote repo 指向，隨後才能繼續 commit。

    ```shell
        #cd 到 /themes/next 底下
        git remote set-url origin https://github.com/ThatMrWayne/hexo-theme-next.git

        # 改完後先重新拉
        git pull

        # 再 commit
        git add .
        git commit -m "Update comments.njk"
        git push
    ```

- 要記得再多改 `.gitmodule` 內 url 指向 fork 的 repo

- 回到專案 root 再 commit 一次 (此時有兩個 change ， `_config.yml`& `.gitmodule`)

最後重新部署後就完成啦～

{% asset_img post7_5.png pst7_5 %}


---

## 文章瀏覽次數功能

都加了留言，也研究一下如何加入瀏覽次數功能。

### Firestore

1. 到 Firebase 加入新的專案

2. 記住自己專案的 WEB API key 和 專案 ID

    {% asset_img post7_6.png pst7_6 %}

3. 建立 Firestore database

4. 到專案的 `_config.yml` 設定

    ```
    firestore:
      enable: true
      collection: articles # Required, a string collection name to access firestore database
      apiKey: FIRESTORE_API_KEY # Required
      projectId: FIRESTORE_PROJECT_ID # Required
    ```

    因為 `FIRESTORE_API_KEY` 和 `FIRESTORE_PROJECT_ID` 算是 sensitive info，  
    這邊利用 github actions 部署時，使用 github secret 動態帶入資訊。

5. 部署

- 部署後確實可以看到文章的瀏覽次數變成 1，但和我想像的不太一樣，  
  我希望的是能記錄每次進入文章的計數，研究後發現在 nextT  
  (`themes/next/source/js/third-party/statistics/firestore.js`)   
  底下是使用 localStorage 記錄是否有瀏覽過，  
  如果瀏覽器有進入過某篇文章，之後再進入瀏覽次數就不會再加1。

- 後來決定直接把 `firestore.js` 內這段 comment 掉，以達到我的目的，   
  改掉後也需要先 commit submodule。
    ```javascript
        // if (localStorage.getItem(title)) {
        //   increaseCount = false;
        // } else {
        // Mark as visited
        //  localStorage.setItem(title, true);
        // }
    ```

6. 更改 Firestore 的安全性規則如下
    ```
    rules_version = '2';

    service cloud.firestore {
    match /databases/{database}/documents {
        match /articles/{document=**} {
          allow read, create: if true;
          allow write: if true;
        }
        match /{document=**} {
          allow read, write: if false;
        }
      }
    }
    ```

最後就可以看到瀏覽次數啦～

  {% asset_img post7_7.png pst7_7 %}


----

That's all I wanna share in this post ~

See ya ~ 👋
