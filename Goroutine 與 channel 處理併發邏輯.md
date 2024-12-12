Goroutine 是 Go 語言中的輕量級執行單元，相當於「執行緒 (Thread)」，但由 Go 執行環境 (runtime) 管理，不同於 Java 或 Python 需要由開發者自行管理執行緒。

大綱

- [基本操作](#基本操作)
- [sync package](#sync-package)
- [Channel](#channel)
- [錯誤處理與 goroutine leak](#-錯誤處理與-goroutine-leak)
- [避免 Goroutine leak 的最佳實踐](#避免-goroutine-leak-的最佳實踐)

## 基本操作

使用 `go` 關鍵字開始一個 goroutine：

```go
func main() {
    go func(){
	    fmt.Println("Hello, goroutine!")
    }

    fmt.Println("Main function is running")

    // 暫停一下，以確保 goroutine 有時間完成
    time.Sleep(3 * time.Second)
}
```

除了 `go func(){}` 也可以用已定義好的 func:

```go
func hello() {
    fmt.Println("Hello, goroutine!")
}

func main() {
    go hello()
    fmt.Println("Main function is running")
    time.Sleep(3 * time.Second)
}
```

## sync package

`sync` package 提供了一些同步 goroutine 的方法，例如 `WaitGroup` 用來等待 goroutine 完成。

步驟：

- 建立一個 `WaitGroup`
- `Add` 一個 goroutine (背後持有一個計數器，每次 `Add` 一個 goroutine，計數器加一)
- 在 goroutine 結束時呼叫 `Done` (計數器減一)
- 使用 `Wait` 等待所有 goroutine 完成 (計數器為 0)

```go
func hello(wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Println("Hello, goroutine!")
}

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go hello(&wg)
    wg.Add(1)
    go hello(&wg)
    fmt.Println("Main function is running")
    wg.Wait() // 等待所有 goroutine 完成，不需要 time.Sleep
}
```

註：由於 goroutine 併發執行，所以可以用來模擬一些並行的操作，以測試 api 同時處理多個請求的情況，例如：購買、儲值等。

## Channel

Channel 是 goroutine 之間交換訊息的橋樑

```go
func hello(ch chan string) {
    ch <- "Hello, goroutine!" // 寫入資料到 channel
}

func main() {
    ch := make(chan string) // 建立一個 string channel
    go hello(ch)
    fmt.Println(<-ch) // 從 channel 讀取資料
}
```

Channel 有兩種：`unbuffered` 和 `buffered`，預設為 `unbuffered`，當 channel 是 `unbuffered` 時，寫入資料到 channel 會 block，直到有其他 goroutine 從 channel 讀取資料，也就是說有進就要有出，否則會 deadlock。

```go
func hello(ch chan string) {
    ch <- "Hello, goroutine!"
}

func main() {
    ch := make(chan string)
    ch <- "Hello, main!"
    fmt.Println(<-ch)
}

// Output:
// fatal error: all goroutines are asleep - deadlock!
```

buffered channel 可以在建立時指定 channel 的容量，當 channel 滿了才會阻塞，最常用的情境是用來控制 goroutine 的數量，例如：限制最大 goroutine 數量，避免 goroutine 過多導致系統資源耗盡。

```go
func worker(id int, ch chan int) {
    for {
        data, ok := <-ch // 從 channel 讀取資料，channel 尚未關閉時若沒有資料會 block，直到有資料可以讀取
        if !ok { // 如果 channel 已經關閉，ok 為 false
            break
        }
        fmt.Printf("Worker %d got %d\n", id, data)
    }
}

func main() {
    ch := make(chan int, 100)

    // 啟動 3 個 goroutine 不斷從 channel 讀取資料
    for i := 0; i < 3; i++ {
        go worker(i, ch)
    }
    for i := 0; i < 100; i++ {
        ch <- i
    }
    close(ch)
}
```

## goroutine leak 與錯誤處理

goroutine leak 是指 goroutine 啟動後沒有正確結束，導致 goroutine 永遠不會退出導致 goroutine 耗盡系統資源，常見狀況有：

- 未關閉的 Channel (缺少 close())
- 死鎖 (Deadlock)
- 未取消的任務 (缺少超時或取消控制)
- 阻塞的 Select 語句 (未處理所有通道情況)
- Race Conditions

### 未關閉的 Channel

```go
func producer(ch chan int) {
	for i := 0; i < 5; i++ {
		ch <- i
		fmt.Println("發送資料:", i)
		time.Sleep(time.Second)
	}
	// 忘記關閉 Channel，導致消費者阻塞
}

func consumer(ch chan int) {
	for {
		data, ok := <-ch
		if !ok { // 沒有資料且 Channel 已關閉
			break
		}
		fmt.Println("接收資料:", data)
	}
	fmt.Println("消費者結束")
}

func main() {
	ch := make(chan int)

	go producer(ch)
	go consumer(ch)

	time.Sleep(10 * time.Second)
	fmt.Println("主程序結束")
}
```

當 producer 執行完成後，沒有呼叫 close，導致 consumer 永遠阻塞。

解法：使用 close() 正確關閉 Channel, **只有生產者應該關閉 Channel**。

### 死鎖 (Deadlock)

所有 Goroutine 都處於等待狀態，無法解除阻塞，導致死鎖錯誤。

```go
func main() {
ch := make(chan int)

    // 發送資料但無人接收
    ch <- 1 // 無消費者，導致死鎖

    fmt.Println("完成")

}
```

ch <- 1 嘗試寫入資料，但沒有消費者讀取

解法：啟動消費者 Goroutine

### 未取消的任務 (缺少超時或取消控制)

若 Goroutine 在等待資料時，資料永遠不會到來，會導致阻塞且無法結束。

```go
func worker() {
  for {
    select {
    case <-time.After(time.Second):
      fmt.Println("工作中...")
    }
  }
}

func main() {
  go worker()

  // 主程序等待 3 秒後退出
  time.Sleep(3 * time.Second)
  fmt.Println("主程序結束")
}
```

worker() 永遠等待 time.After() 的資料，但資料永遠不會到來，導致 Goroutine 永遠阻塞。

解法：使用 Context 控制 Goroutine 的取消

```go
func worker(ctx context.Context) {
  for {
    select {
    case <-time.After(time.Second):
      fmt.Println("工作中...")
    case <-ctx.Done():
      fmt.Println("取消工作")
      return
    }
  }
}

func main() {
  ctx, cancel := context.WithCancel(context.Background())
  go worker(ctx)

  // 主程序等待 3 秒後取消 Goroutine
  time.Sleep(3 * time.Second)
  cancel()
  fmt.Println("主程序結束")
}
```

### 阻塞的 Select 語句 (未處理所有通道情況)

```go
func worker(ch chan int) {
  for {
    select {
    case data := <-ch:
      fmt.Println("接收資料:", data)
    }
  }
}

func main() {
  ch := make(chan int)

  go worker(ch)

  // 主程序發送資料但無人接收
  ch <- 1

  fmt.Println("主程序結束")
}
```

worker() 只處理 ch 的資料，但主程序發送資料到 ch，導致 worker() 阻塞。

解法：使用 default 處理所有通道情況

```go
func worker(ch chan int) {
  for {
    select {
    case data := <-ch:
      fmt.Println("接收資料:", data)
    default:
      fmt.Println("無資料")
    }
  }
}

func main() {
  ch := make(chan int)

  go worker(ch)

  // 主程序發送資料但無人接收
  ch <- 1

  fmt.Println("主程序結束")
}
```

### Race Conditions

```go
func main() {
  var wg sync.WaitGroup
  var data int

  for i := 0; i < 10000; i++ {
    wg.Add(1)
    go func() {
      data++
      wg.Done()
    }()
  }

  wg.Wait()
  fmt.Println("data:", data)
}
```

data++ 可能在多核 CPU 上同時執行，導致在不同 goroutine 中同時 read 得到一樣的值並且 write，原本預期 data++ 10000 次，實際上可能少於 10000。

解法：使用 sync.Mutex 保護共享變數

```go
func main() {
  var wg sync.WaitGroup
  var data int
  var mu sync.Mutex

  for i := 0; i < 10000; i++ {
    wg.Add(1)
    go func() {
      mu.Lock()
      data++
      mu.Unlock()
      wg.Done()
    }()
  }

  wg.Wait()
  fmt.Println("data:", data)
}
```

## 避免 Goroutine leak 的最佳實踐

1. 使用 WaitGroup 等待 Goroutine 完成
2. 使用 Channel 通知 Goroutine 退出
3. 使用 Context 控制 Goroutine 的取消
4. 使用 defer 關閉 Channel
