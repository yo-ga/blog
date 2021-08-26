---
title: '淺談 AWS ElastiCache Redis 資料淘汰機制及相關 Metric'
date: 2021-08-25 16:25:39
tags:
 - AWS
 - redis
 - elasticache
---
這篇主要會談談 Redis 淘汰資料機制會如何觸發對應動作，及 AWS 上相關觀察的 Metric、Parameter Group Value。

## 前提
之所以會需要去了解相關內容的機制，是因為在觀察到目前使用 ElastiCache 的記憶體用量頻頻升高，已有多次峰值超過 80%，所以試圖配合 CloudWatch 其他指標進行評估，確認是否需要更改大小以保環境穩定。然而這些指標之間代表的關係為何，跟設定檔的參數間是否相等或是有間接相關，對一個來說初入 AWS 叢林的小白兔來說，一時之間會有點霧煞煞。既然都花時間了解測試，那就整理成一篇筆記吧！<!--more-->

另外先把本篇使用及測試的環境記於文前，避免未來參考時混淆：
- Redis: 4.0.10
- AWS ElastiCache: Cluster mode, cache.t2.micro, 2021.08 建立
- Redis Client: redis-cli 6.2.5, redis-py 3.5.3 (Python 3.8.11)

## Redis 如何淘汰資料
在 Redis 淘汰資料這塊上，其實主要可以依觸發條件分成在兩種，一個是存活時間，另一個是資料使用量達到資料使用記憶體的最大允許量。

### 存活時間 (TTL)
以存活時間來說比較簡單，就是依照存活時間設定，一旦到時限之前都尚未因各種原因刪除淘汰，就淘汰該 key。以下是用 Python 進行 TTL 設定。
```python
from datetime import timedelta, datetime
from redis import Redis

client = Redis(host="redis.amazonaws.com", port="6379")
client.set("key1", "data", ex=timedelta(minutes=15))  # 15 分鐘後過期
client.set("key1", "data", ex=15)  # 15 秒後過期

client.set("key2", "data2")  # 先設定資料不設定 ttl
client.expireat("key2", int(datetime(2022, 11, 27, 21, 57, 42, 100).timestamp()))  # 設定特定過期時間 timestamp
```

針對過期處理，又可以分被動式 (passive) 跟主動式 (active) 淘汰兩種。被動是指當 client 有去接觸到 key 時，確認是否是否過期，若發現則 time out 掉。然而若照此做法，則可能會有許多需要淘汰掉但未曾使用過的 key 囤積，所以另外得靠 redis 進行處理，處理流程如下，而此流程則維持每秒 10 次的頻率進行。

1. 從有被標記 TTL 的 key 中抓出 20 個進行測試
2. 將過期的進行淘汰
3. 當步驟二淘汰的 key 有多於 5 個則回到步驟一，若無則結束

### 達到最大記憶體 (maxmemory) 使用量
講完過期淘汰 (expire) 的機制，則輪到資料達到最大記憶體與許使用量時去除 (evict) 的機制。在 redis 原生的設定檔中，有個 `maxmemory` 可以去設定 Redis 最多能佔多少記憶體。假設今天不斷塞資料讓 Redis 所佔記憶體容量達到該設定數值，則觸發 eviction 機制，此時會依據 config 中 `maxmemory-policy` 選用的 policy 進行淘汰，若清出可用的空間後，則將 data 繼續寫入。若盡可能淘汰後仍無法清除足夠空間，則會回傳 OOM (Out Of Memory) 錯誤，不接受存入資料，直到清出空間為止。本文不會特別針對每一種 policy 進行解釋，僅針對預設行為解釋。

在 Redis 4 下，若為原生手動安裝，default policy 為 `noeviction`，也就是沒有 eviction 機制，若資料又不巧沒設定過期時限，到填滿那一刻就注定發生 OOM，只能手動刪除資料。至於 AWS ElastiCache Redis 下，設定檔是靠 Parameter Group 進行處理，其中 default group 的 `maxmemory-policy` 是 `volatile-lru`。而此 policy 簡單來說就是從有設定 TTL 的 Key 按照 LRU (Less Recently Used) 演算法挑出可能近期不用的 Key 進行淘汰，所以 TTL 的長短反倒不是重點，僅僅作為區分是否需要長期保存的依據。在這個狀況下，必須注意無 TTL 的資料是否會長期成長而無刪除，若有這種狀況，即使有 eviction 機制，也會隨著 Redis 內部沒帶 TTL 資料逐步排擠帶 TTL 資料，最後沒有任何可淘汰的資料進而 OOM。

另外在手動安裝原生 Redis 狀況下，基本上若需要就會將系統能用的記憶體吃好吃滿，這樣的狀況下，就算 Redis 沒有 OOM，也可能排擠到系統其他程式。公有雲各廠為了必免 Redis 自己完全吃滿，讓其他 snapshot 等執行程式失敗，故多半有類似「保留記憶體」的參數設定。在 AWS 中參數為 `reserved-memory-percent` 或 `reserved-memory`，使用哪個視建立的 ElastiCache Redis 時間而定，若在 2017.03.16 之前則是 `reserved-memory`，反之則是 `reserved-memory-percent`。在 ElastiCache Redis 4.0 的 default parameter 中，`reserved-memory-percent` 是 25，所以就是會預留機器 25% 記憶體，以 75% 記憶體容量的作為 Redis 服務的 `maxmemory`。這點可以建立完 Redis 後，進 Redis 下 `INFO memory` 拿 `maxmemory` 的數值跟 default parameter group `maxmemory` 相除得證。

## 服務監測數值
基本上，講完設定方式後，後續就剩下作為維運團隊需要觀察哪些參數，以了解各項數據的成長趨勢及發生特定事件的時間點。在這邊就主要以 AWS CloudWatch 及 Redis `INFO` 指令有的參數作為說明對照。

### 記憶體相關
主要會確認用了多少記憶體，還剩下多少可以使用，而 CloudWatch 有 `FreeableMemory` 可以知道系統還有多少可以使用，也有 `DatabaseMemoryUsagePercentage` 可以知道所用記憶體的百分比，很多人一開始會以為這兩項是可以直接互通的，然而實際上雖然同樣是記憶體相關，但兩個卻有些不同。`FreeableMemory` 是指以 Instance 角度來看，有多少空閒記億體可以使用，所以這個參數會包含前面的保留記憶體。`DatabaseMemoryUsagePercentage` 是在 Redis 中下 `INFO` 拿 `used_memory` 除以 `maxmemory` 所得，而前面有提到這邊的 `maxmemory` 是扣除保留記憶體的。所以在預設狀況下會出現 `DatabaseMemoryUsagePercentage` 已經 100%，但 `FreeableMemory` 仍有 instance 約四分之一的記憶體未使用。

Redis `INFO` 中 `used_memory` 指的是 Redis allocator 所佔用的容量，然而這些 allocator 有一小部分是 Redis 服務本身自帶的，這個容量則被記於 Redis `INFO` 中的 `used_memory_overhead` 中。至於 Redis 服務剛起來不屬於 allocator 的則被獨立在 `used_memory` 外，記於 `used_memory_startup`。

### Expired & Evicted
在成功觸發 Eviction policy 淘汰資料時，Redis `INFO` 中 `evicted_keys` 會累計，直到重啟 service，然而 CloudWatch 中的 `Evictions` 則是記錄每個時間點因觸發 Eviction Policy 淘汰的數量。在過期資料下也有同樣狀況，只是 Redis 中換成 `expired_keys`，CloudWatch 中變成 `Reclaimed`。

## 後記
這篇算是換工作後重拾 blog，就當作未來工作上的 survey、筆記或備忘錄吧！同時提供給有同樣狀況、需求的<del>未來的自己</del>工程師們一個參考或借鏡。

## 參考連結
[1] [Redis-py Document](https://redis-py.readthedocs.io/en/stable/)
[2] [How Redis expires keys](https://redis.io/commands/expire#how-redis-expires-keys)
[3] [Using Redis as an LRU cache](https://redis.io/topics/lru-cache)
[4] [Redis INFO Command](https://redis.io/commands/INFO#notes)
[5] [Monitoring best practices with Amazon ElastiCache for Redis using Amazon CloudWatch](https://aws.amazon.com/blogs/database/monitoring-best-practices-with-amazon-elasticache-for-redis-using-amazon-cloudwatch/)
[6] [AWS ElastiCache Host-Level Metrics](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheMetrics.HostLevel.html)
[7] [AWS ElastiCache Metrics For Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/CacheMetrics.Redis.html)