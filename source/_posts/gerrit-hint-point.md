---
title: "Gerrit 及多人協作的好姿勢"
date: 2019-09-12 12:05:14
tags:
 - git
 - gerrit
 - develop
---
因為工作關係，面對及使用 Gerrit 將近一年。然而最近因部門內大家都開始集中開發，並提交到同一個 Repository，開始面對到一些因不良習慣產生的問題，並奔波於同事之間解決問題。今天這邊就簡單介紹一下 Gerrit 是什麼，以及在這種環境底下對初學者來說該如何妥善操作。<!--more-->

## Git
在講 Gerrit 之前，勢必得先解釋 Git 是什麼。簡單來說，就是一款版本控制軟體，能夠留下更改歷史並方便的選取特定版本，如果稍有經驗的讀者肯定也聽過 SVN，基本上是同類型軟體，只是已逐漸成為歷史眼淚。本篇就不花篇幅教學，僅就會遇到的指令提出說明，畢竟光是講完基本指令及整個操作概念，基本上就能產生出極大篇幅去解釋，而網路資源已經夠多，我暫時不多花心力去談論。

基於 Git 的使用及搭配，版本倉儲代管服務就順勢產生，各家也都有不同的服務項目及特點，但基本上除了基本將 Git 紀錄及內容提交以外，還會搭配權限控制及審查機制。當然近年來有多款服務或軟體不斷與 CICD 進行整合，讓 SRE、RD、QAE 能更進一步在同意地方就處理完成，而不用在串連多項服務組成完整 Workflow。而現在大家最為熟知的諸如 GitHub、GitLab、Bitbucket，各有各的好，除了工程師自己的 Side Project 外，多半也是看上層的臉色，所以也就不評斷。

## Gerrit
而 Gerrit 就是這樣一套版本倉儲代管服務軟體，只是他與多數其他軟體有許多奇妙的機制，底下一一說明。

嚴格來說，他提供的功能就只有程式碼代管、權限控制、程式碼審查。使用者能從上頭將程式碼下載下來，修改並提交審查，待審查人員審核完畢後將這次修改合進目標分支。如果熟知 GitHub 或 GitLab 的讀者肯定會想說這不就是直接將這次的修改開在不同分支並將分支上傳，並開 PR (Pull Request) 讓對方知道這次改動即可？重點就在這邊。

首先，Gerrit 除非伺服器端存在該分支，開發者不能直接在本地端建立其他分支並直接上傳。另外 Gerrit 在提交變更時不能直接推到目標分支，而是要推到 `refs/for/<Branch Name>` 上面等待審查。最後一點 Gerrit 將每次變更叫做 Change，當推上去伺服器時就是提交一次 Change，服務也會自動幫你開好，等同其他服務的 PR。這邊的 Change 也是單止一個 commit，當多個 commit 時視為多個 Change，而非其他服務將 PR 是做一串 commits。既然不能將整個分支上傳上去，那 Gerrit 就要找到其他方式辨識每次不同的 Change，那就是 Change ID。

那 Change ID 是怎麼產生的呢？Gerrit 在你要 clone project 下來時有個選項，「Clone with commit-msg hook」，從中可以瞧見一點端倪。

```sh
git clone http://<Gerrit IP>/<Project Name> && (cd <Project Name> && curl -kLo `git rev-parse --git-dir`/hooks/commit-msg http://<Gerrit IP>/tools/hooks/commit-msg; chmod +x `git rev-parse --git-dir`/hooks/commit-msg)
```

也就是 Gerrit 有一段預先準備好的 script，將會替換掉原先 git commit-msg hook 的 script。然後當你建立新的 commit 時，便會自動在你的 commit massage 底下加上類似 `Change-Id: I02b2255dc2342342342342342323424340989233` 這樣的內容。當每次上傳時，就用 change ID 確認這是哪次的 Change，而利用該次 change 的 parent commit ID 辨識你要跟在哪個 commit 後面。所以就算你這次的 Commit ID 變了，只要是同一個 Change ID，你就會更改到同一個 Change。因為如此，當你的變更受到退回需要更改時，只要將檔案更改完後，將檔案 git add 加進紀錄，並執行 `git commit --amend` 並 push 到 `refs/for/<Branch Name>` 就成功了。

## 眉角
多人密集協作底下，基本上都是並行開發，也就是說一開始你是跟在 commit 1 後面，在你開發完後，可能早就有其他人提交 commit，伺服器上分支都已經到 commit 4 或 5，在這種情況下很容易出現 Conflict 狀況。對初學者而言，會因 Gerrit 不用上傳 Branch，而在本機上就直接在 master branch 上進行處理。一旦發生 Conflict，本機上又得折騰一番才能處理衝突。下面講一下我的習慣，當然沒有所謂的好壞，但能有效避免不必要的麻煩。另外也適用於其他 Service，只是將 master 代換成各自開發源頭分支，及部分步驟參考著用。

首先假定遠端分支名稱是 master，而在本機端的 master 分支我就保持乾淨狀況，不在上面進行修改，當有需要開發時就按底下步驟操作。
1. `git checkout master`: 先回到 master branch
2. `git pull origin master`: 更新至伺服器端上最新的 code，因沒有任何編輯，故分支路線就順順成長下去，不可能有衝突。
3. `git checkout -b new_branch_name`: 這時你會從本機上複製出一個分支，並將跳到該分支上，之後新 commit 也會在這支上面。
4. 進行開發並將新檔案 `git add`
5. `git commit -m "Commit 1"`: 加上 commit 並建立新 Change ID。
6. `git checkout master`: 回到 master 分支。
7. `git pull origin master`: 拉新的程式碼回來。
8. `git checkout new_branch_name`: 回到要提交的分支。
9. `git rebase master`: 最為關鍵的一步，將要提交的 commit 剪下並貼到最新的 commit 後面。
10. 當有衝突時，先將衝突整理完後 `git add` 檔案，再 `git rebase --continue`
11. `git push orgin new_branch_name:refs/for/master`: 最後提交 change

雖然步驟很多，但相比於直接在 master 上改而發生紀錄有落差，進而發生衝突，版本回復處理又擔心原本改動不見還要好。另外的好處就是當審查者被眾多待審查的更動塞住後，你也可以先開新分支繼續做下一個改動，如果最後要取消，就直接將不要的內容加進去，並把分支刪除就乾乾淨淨。

## 碎語
如果如果是你進去時大家都在用 Gerrit，並且暫時不會更改服務軟體，那就乖乖繼續使用吧。如果有考慮其他種軟體時，最好還是選擇其他如 GitLab 或 BitBucket 之類的軟體。一來最新手使用習慣會有比較好的建立，二來對開發者也能在本機端將一次的項目拆成多個 Commit，當每個小功能做壞時，也能在本機擁有版本控制的優勢，直接 reset 或 revert。以上就是目前使用的想法，就給大家參考了。