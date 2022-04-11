---
title: '寫於低空飛過之後：AWS SAA-C02 考後心得'
date: 2022-04-11 17:08:50
tags:
 - AWS
 - Cloud
 - Certification
---
作為網路上眾多分享文章的受益者，小弟就不在此額外獻醜，何況自己這次的準備方法也沒有學生時代時的認真與穩健，更不適合作為後繼者的借鑑。這次的分享重點就稍微調整一下，談談透過今年改版的資訊來一窺官方想引導的雲端方向，以及小弟個人對於證照的想法。<!--more-->

## SAA-C02 & SAA-C03
當初報名時不確定是消息未完全釋出或是自己沒注意到，根本就不知道今年 SAA 進行改版，直到考前一天重新閱讀官方指南時，才發現已經有 SAA-C03 的簡章出來，並且公告相關生效時程。當初抱持稍微看看的心情，快速地掠過內容，大致上抓了兩個改變方向。一是配題比例，另一個則是 Service Component 的涵蓋範圍。

### 配題比例
C02 跟 C03 的差別當然不及 C01 換 C02 時的變化來的大，原則上仍是維持四大章節，且四大章節的項目不變，變化的是各項目中的配題比例。

|項目名稱|SAA-CO2|SAA-C03|
|:--|:--:|:--:|
|Design Resilient Architectures|30%|26%|
|Design High-Performing Architectures|28%|24%|
|Design Secure Applications and Architectures|24%|30%|
|Design Cost-Optimized Architectures|18%|20%|

若以「考試引導教學」或「考試引導學習」這種前提來看，AWS 確實想透過認證，讓使用者更能對安全性有進一步認識，這點就如同官方再三強調的一樣，基礎建設方面的安全性就放心地交給 AWS，而在其平台上各式設定就有賴使用者的規劃安排。當然不安全的服務牽扯的因素很多，但當設定皆以最低限度作為考量時，就能把可能受害範圍及機率降至一定比例。

再來之所以成本優化會被微幅提升的原因，我的理解是雖然還是有許多新 Workload 在 AWS 上建立，但是走到今天，維運現有服務架構並配合商業需求持續成長，如何在基礎建設上所花的每一分錢都能夠花在刀口上，發揮最大效益，無非是所有架構師每天面臨的課題。如果成本控管是實務上客戶經常面對的問題，那麼將其視為基本需求與期待也是理所當然。

### Service Component
近一兩年來，AWS 官方應對新興市場需求推出各種代管服務，同時也有部分服務相較小眾，對題庫而言也是需要更新及調整，並聚焦核心服務，避免認證與實際業務脫節。我覺得這點也是對於已經投身於相關職務、尚未但有意考取 SAA 的人來說一個重要的考量，假設日常所面對的業務範圍與目前 SAA-C02 的重疊程度比 SAA-C03 高，那就適合先在趕在 SAA-C02 退場前趕緊考取，如此一來，準備的成本也會小很多，反之亦然。有興趣的人可以再多看看兩份指南的附錄，細細品味兩者差異。至於那些在 SAA-C03 中標註「Out of scope」的服務，會不會在未來的幾年中被官方淘汰，值得持續觀察官方作為。

## 證照的用途及定位
<div data-iframe-width="150" data-iframe-height="270" data-share-badge-id="36c109a6-43ce-4721-8452-657d783b1982" data-share-badge-host="https://www.credly.com"></div><script type="text/javascript" async src="//cdn.credly.com/assets/utilities/embed.js"></script>

在談論自身想法前，先曬一下人權照。過去總有許多人爭論著證照到底有沒有用，又或者能證明什麼，始終沒有正確答案，同時也要視哪張證照而定。對我而言，SAA 這張對我的意義主要是從 QA 轉換到 SRE 後的一個審視，站在官方的角度，是不是具備一定基礎。在考試的前後，自身技術能力其實並沒有什麼差異，但過程中早應有所收穫，多了一張證照，不過是在這個速食社會中，多了一個明顯的標籤，給予素未謀面的人一個清晰的想像。另一個好處是重新修正自身對於技術的表達，以求未來在外面溝通時能有更一致的用詞與標準。

## 後記
不確定是否最近 AWS 官方流程有所改變，在送出考試問卷的那刻，沒有如同過去網路上所說的馬上結果，讓我悲觀地以為只能下次再來了。直到上了 Reddit，查到最近考取的人都是如此，才稍微放心點。不過也在考完 5 小時後，就得到最終結果，也讓我總算放鬆且回神。

## Reference
1. [AWS Certified Solutions Architect – Associate (SAA-C02) Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide.pdf)
2. [AWS Certified Solutions Architect - Associate (SAA-C03) Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide_C03.pdf)