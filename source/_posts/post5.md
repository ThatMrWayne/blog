---
title: Docker storage
date: 2025-04-03 16:02:02
tags: [Docker storage]
categories: [Technical]
---

{% asset_img volume.png rl %}

How cute is this picture 🤩

<!-- more -->

## Why

從第一次接觸 docker 這項技術時，感覺一直都是一知半解的在用，用是用了，
東西也會動，但過程中總有很多地方有點疙瘩感，一種雖然會用又好像不會用的感覺，
前陣子找工作時因為 side project 的緣故又重新摸了一遍，結果是也會動，
但那熟悉的疙瘩感又浮了上來🥲。

最近去上了講授有關 docker 底層知識的課，之前在腦中模糊的那幾塊區域，
似乎明朗了一些，趁著印象深刻記錄起來。

---

## TL;DR
記錄 `bind mounts` 和 `volume` 的實驗筆記。

---

## Docker storage

container 在跑起來後，在 container 中所做的任何改變，
都存在於 container 的可寫層，如果將 container 停掉並移除，
可寫層的資料也都會被移除。

如果想保留資料，docker 提供了幾種不同方式，
其中較常聽到的 `bind mounts` 和 `volume` 可以達到很相似的效果。

### Bind mounts

`Bind mounts` 可以將 host 上的指定的資料夾或檔案掛載進 container 內

=> `-v host_path:container_path`


```shell
docker container run -it -v ./host_dir:/container_dir alpine sh
```

原則上可以想像這樣的方式會以 host 的檔案或資料夾為主，覆蓋過在 container 內所見，
以下會分幾種不同情況實際實驗看看，當我們把 host 的檔案或資料夾掛載到 container 內會發生什麼事。

---

### ☕ 暫停一下 ☕
說明一下等等實驗用到的 container，這邊使用 `alpine` 為 base image ，
在 image 內的 root directory 內新增了 app 資料夾，其中新增了一個檔案 `a.txt`，
新增後 commit 成新的 image (name: `my-alpine:v1`)，
所以 `/app/a.txt` 是屬於 image 的 read-only layer。

```shell
~$ docker container run -it --rm my-alpine:v1 sh
/ # ls
app    dev    home   media  opt    root   sbin   sys    usr
bin    etc    lib    mnt    proc   run    srv    tmp    var
/ # cd app
/app # ls
a.txt
/app # cat a.txt
 I am a.txt
```

---

### 繼續講 Bind mounts ～

1. 第一種情況
  - 在 host 內已存在一個 app 資料夾，在 app 資料夾內有三個檔案

    || host | image |
    |---|---|---|
    | directory | app/ | app/ |
    | files in directory | `b.txt`, `c.txt`, `k.txt` | `a.txt` |

  - 試著把 host 機器的 app 資料夾掛到 container 裡
    ```shell
      docker container run -it -v ./app:/app my-alpine:v1 sh
    ```

  - host 的 app 資料夾會蓋過 container 的 app 資料夾，進到 container 內會看到 host app 資料夾的內容(三份檔案)

    ```shell
      ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app:/app my-alpine:v1 sh
      /# cd app
      /app# ls
      b.txt  c.txt  k.txt
    ```

  - 另外如果在 host 去查看 container 的 merged dir ，會看到原本的 `a.txt`，因為原先 app 資料夾就已經存在 image 裡，docker 會在 overlay filesystem 疊加之後才 apply volume 上去，所以 host 的 app 資料夾不會出現在 container writable layer 裡

    ```shell
      root@ip-172-31-85-204:/var/lib/docker/overlay2/1be9453e56824abfa1b810ad15968afc60d1c3a8b56a7b1f315e2475f013fc18/merged# ls
      app  bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
      root@ip-172-31-85-204:/var/lib/docker/overlay2/1be9453e56824abfa1b810ad15968afc60d1c3a8b56a7b1f315e2475f013fc18/merged# cd app
      root@ip-172-31-85-204:/var/lib/docker/overlay2/1be9453e56824abfa1b810ad15968afc60d1c3a8b56a7b1f315e2475f013fc18/merged/app# ls
      a.txt #原先 readable layer 的 a.txt
    ```

  - 在 container 內對 app 資料夾內所做的變動都不會出現在 writable layer(都是在 volume 層)

    ```shell
      # insdie container
      /app# touch e.txt

      # 在 host 去看 container 的 writable layer 的 diff 層，看不到新增的 e.txt 檔案
      root@ip-172-31-85-204:/var/lib/docker/overlay2/1be9453e56824abfa1b810ad15968afc60d1c3a8b56a7b1f315e2475f013fc18/diff# ls
    ```


2. 第二種情況
  - 在 host 內原先沒有 app2 資料夾

    || host | image |
    |---|---|---|
    | directory | app2/ | app/ |
    | files in directory | | `a.txt` |

  - 試著把 host 機器的 app2 資料夾掛到 container 裡，
  因為 host 原先沒有 `app2` 資料夾，所以 docker 會先 create 一個，
  然後因為 app2 裡面是空的，所以 container 裡看到的也是空的

    ```shell
      ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app2:/app my-alpine:v1 sh
      /# cd app
      /app# ls
      /app#
    ```

  - 在 container 內對這個 app 資料夾內所做的變動一樣都不會出現在 writable layer

    ```shell
      # 加 g.txt
      /app# touch g.txt


      # 在 host 去看 container 的 writable layer 的 merged 層，也看不到新增的 g.txt 檔案，一樣看到原先 image 的 a.txt
      root@ip-172-31-85-204:/var/lib/docker/overlay2/feb140ae6bff34bde098a2c3f54e3b01d041324471928561aa20d7f892ff496b/merged# ls
      app  bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
      root@ip-172-31-85-204:/var/lib/docker/overlay2/feb140ae6bff34bde098a2c3f54e3b01d041324471928561aa20d7f892ff496b/merged# cd app
      root@ip-172-31-85-204:/var/lib/docker/overlay2/feb140ae6bff34bde098a2c3f54e3b01d041324471928561aa20d7f892ff496b/merged/app# ls
      a.txt
    ```

3. 第三種情況
  - 在 host 機器內原先有 app 資料夾，image 內沒有資料夾

    || host | image |
    |---|---|---|
    | directory | /app | /app2 |
    | files in directory | `b.txt`, `c.txt`, `k.txt`, `e.txt` |  |

  - 試著把 host 機器的 app 資料夾掛到 container 裡
    ```shell
      ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app:/app2 my-alpine:v1 sh
      /# cd app2
      /app2# ls
      b.txt  c.txt  e.txt  k.txt
    ```

  - 因為 image 內原先沒有 `app2` 資料夾，所以 docker 會先 create 一個`app2` 資料夾，而且會加在 container 的 writable layer ，所以可以在 upper layer(diff) 層看到 ，但仔細觀察，往裡面看會發現沒有 host app 資料夾裡的檔案們，是空的

    ```shell
      root@ip-172-31-85-204:/home/ubuntu# cd /var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff
      root@ip-172-31-85-204:/var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff# ls
      app2  root
      root@ip-172-31-85-204:/var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff# cd app2
      root@ip-172-31-85-204:/var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff/app2# ls
      root@ip-172-31-85-204:/var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff/app2#
    ```

  - 在 container 內對這個 app2 資料夾內所做的變動一樣都不會出現在 writable layer

    ```shell
      # 在 container 加 d.txt
      ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app:/app2 my-alpine:v1 sh
      /# cd app2
      /app2# ls
      b.txt  c.txt  e.txt  k.txt
      /app2# touch d.txt

      # 在 host 去看 container 的 writable layer 的 diff 層，看不到新增的 d.txt 檔案，一樣是空的
      7e24c94fa11d/diff/app2# ls
      root@ip-172-31-85-204:/var/lib/docker/overlay2/ae7fa21216bff4fb1725b33362a60bdf23ab0a06a726c650ea7a7e24c94fa11d/diff/app2#
    ```
---
### 小結-1

  1. 如果 container 裡原先沒有要掛載的資料夾，**docker 會先 create 資料夾結構 在 container 的 writable layer**，其餘裡面的檔案或是資料夾都還是會在 volume 層，不會出現在 writable layer。

  2. 如果 container 裡原先要掛載的資料夾已經存在，裡面被掛載的的檔案或是資料夾都會存在volume 層，不會出現在 writable layer。
---

4. 第四種情況
  - bind mounts 檔案 , 且 image 裡原先已存在同名檔

    || host | image |
    |---|---|---|
    | file | app/b.txt | /app/a.txt |

  - 試著把 host 上的 `app/b.txt` 掛到 container 裡原先已存在的 `/app/a.txt`

    ```shell
    ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app/b.txt:/app/a.txt my-alpine:v1 sh
    /# cd app
    /app# ls
    a.txt
    /app# cat a.txt
    I am b.txt.
    ```

  - 因為原先 image 裡已經存在 `/app/a.txt`，所以 docker 會用 host 的 `app/b.txt` 蓋過 container 的 `/app/a.txt`，這個檔案一樣不會出現在 container writable layer

    ```shell
      root@ip-172-31-85-204:/home/ubuntu# cd /var/lib/docker/overlay2/a3df90f6a6caffe686bc8cdc749856947130927d667e1ac8b936c44fe65836e7/diff
      root@ip-172-31-85-204:/var/lib/docker/overlay2/a3df90f6a6caffe686bc8cdc749856947130927d667e1ac8b936c44fe65836e7/diff# ls
      root
      root@ip-172-31-85-204:/var/lib/docker/overlay2/a3df90f6a6caffe686bc8cdc749856947130927d667e1ac8b936c44fe65836e7/diff#
    ```

5. 第五種情況
  - bind mounts 檔案 , 且 image 裡不存在檔案

    || host | image |
    |---|---|---|
    | file | app/b.txt | /app/a.txt |

  - 試著把 host 上的 `app/b.txt` 掛到 container 裡原先不存在的 `/app/g.txt`，原先只有 `/app/a.txt`

    ```shell
    ubuntu@ip-172-31-85-204:~$ docker container run -it -v ./app/b.txt:/app/g.txt my-alpine:v1 sh
    /# cd app
    /app# ls
    a.txt  g.txt
    ```

  - 這種情況下，docker 會 create `/app/g.txt` 在 writable layer，其實很像是上面[第三種情況](#3-第三種情況)

    ```shell
      # 在 host 的 upper(diff) 層 看得到 /app/g.txt
      root@ip-172-31-85-204:/home/ubuntu# cd /var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/diff
      root@ip-172-31-85-204:/var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/diff# ls
      app  root
      root@ip-172-31-85-204:/var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/diff# cd app
      root@ip-172-31-85-204:/var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/diff/app# ls
      g.txt

      # 在 host 的 merge view 看得到 image 原先就有的 a.txt 和多寫入的 g.txt
      root@ip-172-31-85-204:/var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/merged# cd app
      root@ip-172-31-85-204:/var/lib/docker/overlay2/d7d54020c2ead028732671d8dd052449cf147d6ea8ea59c7253ae1c5b5308279/merged/app# ls
      a.txt  g.txt
    ```
---

### Volume

和 bind mounts 非常相似，主要差異是 volume 所用的 host 資料夾由 docker 管理。

使用前需要先 create volume。

1. 第一種情況
  - 全新 volume，image 內原先有資料夾

    ```shell
      docker volume create my-vol
      docker container run -it -v my-vol:/app my-alpine:v1 sh
    ```

  - 因為是全新的 volume， docker 會先**複製 container 內 app 資料夾裡的內容到 volume 資料夾內**，此時在 container writable layer 一樣看不到

    ```shell
      # 在 container 裡
      /app# touch b.txt
      /app# ls
      a.txt  b.txt

      # 在 host 檢查 upper layer，看不到任何變動
      root@ip-172-31-85-204:/var/lib/docker/overlay2/6dbac8d93dea09eb1dab007ea5339189447234da0bb6349da6097111de03b1f2/diff# ls
      root
    ```

2. 第二種情況
  - volume 已經有資料 , image 沒有資料夾

    ```shell
      docker container run -it -v my-vol:/app2 my-alpine:v1 sh
    ```

  - 因為 image 原先沒有 `app2` 資料夾 ，所以會先 create `app2` 資料夾在 writable layer，而裡面的檔案都存在 volume層，在 container writable layer 看不到

    ```shell
      # 在 host 檢查 diff layer 會發現 app2
      root@ip-172-31-85-204:/var/lib/docker/overlay2/c4769d5369a060d76bd8391b5da109a2e7b14f73c2bbdab2e3d168df880f1934/diff# ls
      app2
    ```

  - 在 app2 裡做的變動也都不會出現在 writable layer

    ```shell
      # 在 container app2 裡加檔案
      /app2# touch c.txt

      # 在 host 檢查 merged 資料夾，看不到東西
      root@ip-172-31-85-204:/var/lib/docker/overlay2/c4769d5369a060d76bd8391b5da109a2e7b14f73c2bbdab2e3d168df880f1934/merged/app2# ls
      root@ip-172-31-85-204:/var/lib/docker/overlay2/c4769d5369a060d76bd8391b5da109a2e7b14f73c2bbdab2e3d168df880f1934/merged/app2#
    ```

3. 第三種情況
  - volume 已經有資料 , image 也有資料夾

    ```shell
      docker container run -it --rm -v my-vol:/app my-alpine:v1 sh
    ```

  - 因為 image 原本資料夾就存在，所以 docker 會直接蓋過去原先的 app 資料夾，volume 裡的檔案都不會出現在 writable layer

    ```shell
      # container 裡
      /# cd app
      /app# ls
      a.txt  b.txt  c.txt
      /app# touch d.txt

      # 在 host 檢查 diff 層是空的
      root@ip-172-31-85-204:/var/lib/docker/overlay2/6c177d88eed2de5cbefa015fdce3d9a275f910c2ba566add84927713be2214c2/diff# ls
      root
    ```
---

### 小結-2

  - 其實 `volume` 和 `bind mounts` 幾乎一樣，只差在 `volume` 初始化時會複製 image 裡的內容(如果 image 裡有東西)，其餘情況都是以 host 裡的檔案為主，會覆蓋 container 裡的內容。

---

That's all I wanna share in this post ~

See ya ~ 👋

