---
title: 小記 - Redis memory 分析
date: 2022-10-26T00:58:37+08:00
tags:
 - SRE
 - redis 
---

近期工作上因頻頻發出 redis 記憶體用量告警，故不得不去嘗試分析目前的 Redis 使用分佈，並進行盤點，進而方便後續服務的調整。這邊稍微紀錄幾種方式並寫下配合 RDB 的分析過程，以備不時之需。<!--more-->

## 環境現況
目前線上正式環境是採用 AWS ElastiCache for Redis，Regis Engine 4.0.10，1 Primary 1 Replica，同時有開啟 CloudWatch 監測，並設定記憶體用量超過 80% 噴出警告。由於 Redis 的資料都是 in-memory，所以 memory 用量可以視作在這個 Node 上資料存放的大小，所以就朝向分析內部存放資料的種類跟容量占比。

CloudWatch 雖然有提供基本的監測，如 CPU、Memory，以及各種類型的指令次數、網路吞吐量大小，但就是沒辦法針對內容進行分析，這時只好尋求其他工具協助。

## 分析方式

### Redis SCAN
因為想要查每個 Key 在 Redis 裡佔了多少容量，同時確認 TTL 的狀況，所以若自己寫 Script 去兜，第一時間就是先用 `SCAN` 把 Key 都列出來，在逐 Key 用 `MEMORY USAGE key` 將每個 Key 實際佔用的記憶體大小回傳，並用 `TTL key` 確認還有多久過期。為避免影響線上環境，我這邊是將備份的 RDB 匯出，在本機環境架設 Redis Server 恢復資料。

後來主要有幾點考量，故未繼續執行下去。

1. 執行效率：必須用 `SCAN` 將整個 Redis 都翻遍，Redis 越大，整體耗時也需越長。
2. 一致性：即使是使用同一個 RDB 檔，也會有一個問題，隨著時間推進，除了沒設定 TTL 還是一樣以外，其他 Key 都會逐步過期淘汰，造成驗證上的失真失準。

### Redis RDB Tool + NUMPY + PANDAS
最後在搜尋時發現直接針對 RDB 進行分析的離線工具，故最後改採這方向進行分析。

首先先安裝相關套件
```sh
> pip install rdbtools python-lzf # redis 分析
> pip install pandas numpy # 分析 rdbtools 產生的 CSV
```

再來則執行 memory mode 的分析
```sh
> rdb -c memory /path/to/your/dump.rdb -f result.csv
```

取得 CSV 檔後，就可以透過 pandas 及 numpy 進行相關分析
```py
import pandas as pd
result = pd.read_from_csv("./result.csv")

# Total memory size
result["size_in_bytes"].sum()

# Filter no TTL
result[result["expiry"].isna()]

# Filter specific prefix
result[result["key"].str.startswith("prefix-", na=False)]
```

## Reference
1. [Redis Command](https://redis.io/commands/)
2. [Redis RDB Tools](https://github.com/sripathikrishnan/redis-rdb-tools)
