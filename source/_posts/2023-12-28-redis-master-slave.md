---
title: Redis Master Slave
date: 2023-12-28 01:54:37
tags:
  - [Redis]
categories:
  - [Redis]
---

## Redis 三部曲之首部曲-主從模式（Master-Slave）

![redis-scaling.drawio](https://i.imgur.com/Gqn4luk.png)
<!-- more -->

### 說明
主從（Master-Slave）模式提供讓 Master（主節點） 向任意數量的 Slave（附節點) 進行資料同步，以解決單點資料庫的問題，減輕主節點的負載壓力。主從複製模式最主要的目的是要實現讀寫分離和資料備份。Redis 讀寫分離的作法是 Master 可以進行讀取和寫入，其他的 Slave 只能讀取（試圖寫入會出現READONLY You can't write against a read only replica.錯誤訊息）。透過 Slave 幫忙處理讀取的工作來減輕 Master 的負擔。

### 檔案架構

![image](https://i.imgur.com/4GlZICz.png)
- docker-compose.yaml: 建立redis相關容器。
- redis-shard： 儲存各redis永久數據，不會因為誤砍container而消失。

### 實作
[完整程式碼](https://github.com/HowhowChen/redis-project/tree/main/redis-master-slave)

1. 新增volume儲存位置

```shell=
mkdir redis-shard && mkdir redis-shard/redis-master0 \
redis-shard/redis-slave0 \
redis-shard/redis-slave1
```
2. 容器規劃

```yaml=
version: '3.4'
services:
  redis-master0:
    image: redis
    container_name: redis-master0
    restart: always
    volumes:
      - ./redis-shard/redis-master0:/data
    ports:
      - 6379:6379
    networks:
      - redis-net

  redis-slave0:
    image: redis
    container_name: redis-slave0
    restart: always
    volumes:
      - ./redis-shard/redis-slave0:/data
    ports: 
      - 9001:9001
    networks:
      - redis-net
    command: redis-server --port 9001 --slaveof redis-master0 6379
    depends_on: 
      - redis-master0

  redis-slave1:
    image: redis
    container_name: redis-slave1
    restart: always
    volumes:
      - ./redis-shard/redis-slave1:/data
    ports: 
      - 9002:9002
    networks:
      - redis-net
    command: redis-server --port 9002 --slaveof redis-master0 6379
    depends_on: 
      - redis-master0
      - redis-slave0

networks:
  redis-net:
    name: redis-scaling-network
```

3. 查詢replication info
確認connected_slaves數量及slave相關資訊是否正確。

```shell=
docker exec -it redis-master0 redis-cli info replication
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.112.3,port=9001,state=online,offset=794,lag=0
slave1:ip=192.168.112.4,port=9002,state=online,offset=794,lag=1
master_failover_state:no-failover
master_replid:90ece01a700c4d5b0df6d36fb6498b437524174d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:794
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:794
```

4. 建立資料及查詢
寫入資料至redis-master0後，確認是否有同步至redis-slave0及redis-slave1。

```shell=
docker exec -it redis-master0 redis-cli set k1 v1
OK
docker exec -it redis-slave0 redis-cli -p 9001 get k1
"v1"
docker exec -it redis-slave1 redis-cli -p 9002 get k1
"v1"
```

### 建立流程
以下以redis-master0與redis-slave0之間為例子做說明

1. redis-slave0 啟動之後會主動向 redis-master0 發送 Sync 命令要求進行同步。
```log=
redis-slave0   | 1:S 27 Dec 2023 15:14:38.607 * MASTER <-> REPLICA sync started
redis-slave0   | 1:S 27 Dec 2023 15:14:38.607 * Non blocking connect for SYNC fired the event.
redis-slave0   | 1:S 27 Dec 2023 15:14:38.608 * Master replied to PING, replication can continue...
redis-slave0   | 1:S 27 Dec 2023 15:14:38.608 * Partial resynchronization not possible (no cached master)
```
2. redis-master0 收到之後會執行 Bgsave 命令建立 rdb 快照檔案儲存到硬碟。同時，Master 會把新收到的寫入和修改資料庫的命令存到緩衝區。
```log=
redis-master0  | 1:M 27 Dec 2023 15:14:38.609 * Replica 192.168.112.3:9001 asks for synchronization
redis-master0  | 1:M 27 Dec 2023 15:14:38.609 * Full resync requested by replica 192.168.112.3:9001
redis-master0  | 1:M 27 Dec 2023 15:14:38.609 * Replication backlog created, my new replication IDs are '90ece01a700c4d5b0df6d36fb6498b437524174d' and '0000000000000000000000000000000000000000'
redis-master0  | 1:M 27 Dec 2023 15:14:38.609 * Delay next BGSAVE for diskless SYNC
```

3. redis-master0 會將剛建立好的快照檔案傳給 redis-slave0。
```log=
redis-master0  | 1:M 27 Dec 2023 15:14:43.744 * Starting BGSAVE for SYNC with target: replicas sockets
redis-master0  | 1:M 27 Dec 2023 15:14:43.745 * Background RDB transfer started by pid 19
```

4. redis-slave0 收到快照檔案後會先把記憶體清空，接著載入收到的快照檔案。
```log=
redis-slave0   | 1:S 27 Dec 2023 15:14:43.747 * MASTER <-> REPLICA sync: receiving streamed RDB from master with EOF to disk
redis-slave0   | 1:S 27 Dec 2023 15:14:43.749 * MASTER <-> REPLICA sync: Flushing old data
redis-slave0   | 1:S 27 Dec 2023 15:14:43.749 * MASTER <-> REPLICA sync: Loading DB in memory
```

5. redis-master0 會再把存在緩衝區的命令傳給 redis-slave0，redis-slave0 再執行這些命令以達成和 Master 同步。
```log=
redis-master0  | 1:M 27 Dec 2023 15:14:43.754 * Background RDB transfer terminated with success
redis-master0  | 1:M 27 Dec 2023 15:14:43.755 * Streamed RDB transfer with replica 192.168.112.3:9001 succeeded (socket). Waiting for REPLCONF ACK from replica to enable streaming
redis-master0  | 1:M 27 Dec 2023 15:14:43.755 * Synchronization with replica 192.168.112.3:9001 succeeded
```

### 心得
許多大型網站在追求效能優化及低延遲性上會常用到Cache，而Redis是市場上最普片作為Cache的資料庫（in-memory），不管在效能還是操作上皆活躍於開源社群中。透由本次文章分想Redis在水平拓展的優化，減輕主節點負載壓力，提高在查詢上的效能，並做好備份工作降低系統異常容錯率。後續會分享剩下二部曲Sentinel、Cluster及Golang、Nodejs等介接Redis實例,敬請期待～


### 參考文章
[Redis (六) - 主從複製、哨兵與叢集模式](https://hackmd.io/@tienyulin/redis-master-slave-replication-sentinel-cluster)
[Redis Master Slave | How to Setup Redis Master and Slave Replication?](https://www.educba.com/redis-master-slave/)