---
title: Golang Worker Pool練習-1
date: 2023-12-21 00:58:54
tags:
  - [Golang]
categories:
  - [Golang, Worker Pool]
---

## 優化Worker Pool的奇技淫巧

![](https://miro.medium.com/v2/resize:fit:720/format:webp/1*Ya3fa36roBBhZlMl-kChXw.png)

<center>圖片來源https://itnext.io/explain-to-me-go-concurrency-worker-pool-pattern-like-im-five-e5f1be71</center>
<!-- more -->

### 題目說明
設計一個隨機生成亂數，計算亂數每一位數字總和（e.g. 69 => 6+9=15）。並請開啟一個Goroutine生成亂數，24個Goroutine計算亂數每一位數字總和。**Bonus 五秒後關閉所有Goroutine。**

詳見[題目來源](https://blog.51cto.com/u_15127617/3300755)

### 優化結果

```go=
package main

import (
    "fmt"
    "time"
    "math/rand"
    "context"
    "sync"
    "runtime"
)

type Job struct {
    Num int64
}

type ResJob struct {
    JobPtr  *Job
    Sum     int64
}

func Generate(ctx context.Context, jobChan chan<- *Job) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Genereate done")
            return
        default:
            num := rand.Int63()
            job := &Job{Num: num}
            jobChan <- job
        }
    } 
}

func Calculate(ctx context.Context, jobChan <-chan *Job, resChan chan<- *ResJob) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Calculate done")
            return
        default:
            job := <-jobChan
            n := job.Num
            sum := int64(0)
            for n > 0 {
                sum += n % 10
                n = n / 10
            }
            
            res := &ResJob{
                JobPtr: job,
                Sum:    sum,
            }
            resChan <- res
        }
    }
}

func main() {
    defer fmt.Println("NumGoroutine: ", runtime.NumGoroutine())

    duration := time.Now().Add(time.Second * 5)
    ctx, cancel := context.WithDeadline(context.Background(), duration)
    defer cancel()

    var jobChan = make(chan *Job, 100)
    var resChan = make(chan *ResJob, 100)
    var wg sync.WaitGroup

    go Generate(ctx, jobChan)

    numCalculators := 24
    wg.Add(numCalculators)
    for i := 0; i < numCalculators; i++ {
        go func() {
            defer wg.Done()
            Calculate(ctx, jobChan, resChan)
        }()
    }

    go func() {
        wg.Wait()
        close(resChan)
    }()

    for {
        select {
        case <-ctx.Done():
            fmt.Println("Context done")
            for {
                _, ok := <-resChan
                if !ok {
                    fmt.Println("ResChan closed")
                    break
                }
            }
            
            return
        case job := <-resChan:
            fmt.Printf("%v's the result is: %v\n", job.JobPtr.Num, job.Sum)
        }
    }
}
```

### 優化思路

1. 使用context上下文控制goroutine狀態
題目bonus思路規定5秒後關閉。

```go=
duration := time.Now().Add(time.Second * 5)
ctx, cancel := context.WithDeadline(context.Background(), duration)
defer cancel()
```

2. 使用sync.WaitGroup確保所有goroutine退出前完成所有工作
- 加入close(resChan)可以優化以下項目：
    - 節省資源：關閉通道可以釋放相關資源。當不再需要使用通道時，關閉它可以讓垃圾回收機制回收相關資源。
    - 通知接收者：確保不會再有resJob被送入resChan。

```go=
numCalculators := 24
wg.Add(numCalculators)

for i := 0; i < numCalculators; i++ {
    go func() {
        defer wg.Done()
        Calculate(ctx, jobChan, resChan)
    }()
}

go func() {
    wg.Wait()
    close(resChan)
}()
```

3. 接收resChan加入select機制判斷
如若使用原先for job := range resChan 會造成邊關閉goroutine同時還在透由channel獲取資料，有概率會deadlock。

```go=
for {
    select {
    case <-ctx.Done():
        fmt.Println("Context done")
        for {
            _, ok := <-resChan
            if !ok {
                fmt.Println("ResChan closed")
                break
            }
        }

        return
    case job := <-resChan:
        fmt.Printf("%v's the result is: %v\n", job.JobPtr.Num, job.Sum)
    }
}
```

4. 兩個goroutine func加入select及context
新增select判讀context是否傳遞關閉訊息並跳出迴圈。

```go=
select {
case <-ctx.Done():
    fmt.Println("Calculate done")
    return
default:
    job := <-jobChan
    n := job.num
    sum := int64(0)
    for n > 0 {
        sum += n % 10
        n = n / 10
    }

    res := &ResJob{
        JobPtr: job,
        Sum:    sum,
    }
    resChan <- res
}
```

### 心得
身為一位Gopher Goroutine及Channel是必須非常熟悉的項目，透由練習優化Worker Pool可以加深自己對Golang的認識。參考的題目是沒有設定停止機制，無限循環生成亂數及計算每位數字總和;這邊優化方向為增加停止機制並使所有Goroutine能正常關閉。可以讓剛入門GO語言的朋友參考。

