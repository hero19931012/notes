# Seam Model 與程式可測試性技術筆記

## 1. 引言: 為什麼要了解 Seam Model？

在軟體開發中，可測試性決定了應用程式的維護性與可靠性。Seam Model 提供了一個有效的架構設計視角，透過識別與設計 “Seams” （可替換的行為點），我們可以降低耦合度，增加測試的靈活性。

---

## 2. Seam Model 概念解釋

### 什麼是 Seam？

Seam 是一個可替換的行為點，允許我們在不更改原始碼的情況下改變程式的行為。這個概念經常出現在測試環境中，藉由替換或模擬元件行為，讓程式碼更容易被測試。

### Seam 的特性：

- 行為可替換性: 可更換具體實作，支援多種測試策略。
- 測試雙關係: 常見於依賴注入 (Dependency Injection, DI)、設計模式等。

---

## 3. Seam 的分類與實作方式

### 靜態 Seam (Static Seam)

- 在編譯時決定，常見於介面繼承與多態設計。

範例:

```go
package main

import "fmt"

// 定義介面
type Greeter interface {
    Greet() string
}

// 真實實作
type RealGreeter struct{}

func (r RealGreeter) Greet() string {
    return "Hello, World!"
}

// 測試用實作
type MockGreeter struct{}

func (m MockGreeter) Greet() string {
    return "Mocked Greeting"
}

// 使用者函數
func PrintGreeting(g Greeter) {
    fmt.Println(g.Greet())
}

func main() {
    realGreeter := RealGreeter{}
    PrintGreeting(realGreeter) // Hello, World!
}
```

### 動態 Seam (Dynamic Seam)

- 在執行時改變行為，常見於策略模式等設計模式。

範例:

```go
package main

import (
	"fmt"
	"time"
)

type TimeProvider interface {
	CurrentTime() string
}

// 真實的時間提供者
type RealTimeProvider struct{}

func (r RealTimeProvider) CurrentTime() string {
	return time.Now().Format("2006-01-02 15:04:05")
}

// 測試用時間提供者
type MockTimeProvider struct{}

func (m MockTimeProvider) CurrentTime() string {
	return "2024-12-09 12:00:00"
}

func PrintTime(tp TimeProvider) {
	fmt.Println("Current Time:", tp.CurrentTime())
}

func main() {
	realTime := RealTimeProvider{}
	PrintTime(realTime) // 顯示真實時間
}
```

---

## 4. 如何設計可測試的程式架構

1. 拆解責任: 模組化程式結構，避免巨石式架構。
2. 控制依賴: 使用介面與依賴注入實作控制反轉 (IoC)。
3. 使用設計模式: 如策略模式、觀察者模式來實現行為可替換性。

---

## 5. 常見錯誤與反模式

### 過度設計:

引入過多的 Seam 會增加開發與維護成本，應視需求設計。

### 未控制依賴:

若類別內部直接建立實例，將導致測試時無法替換行為。

### 忽略 Mock 與 Stub:

未妥善使用 Mock、Stub，導致測試嚴重依賴真實資源。

---

## 6. 測試實戰建議

### 單元測試中的 Seam 應用

- Mock（模擬行為）：用於替代真實實作，控制測試環境。
- Stub（固定回應）：用於返回預期資料，模擬簡單行為。

測試範例：模擬資料庫回應

```go
package main

import (
	"fmt"
	"testing"
)

// 資料庫存取介面
type UserRepository interface {
	GetUser(id int) string
}

// 真實實作
type RealUserRepository struct{}

func (r RealUserRepository) GetUser(id int) string {
	return "Real User"
}

// 測試用 repository
type MockUserRepository struct{}

func (m MockUserRepository) GetUser(id int) string {
	return "Mock User"
}

func GetUserName(repo UserRepository, id int) string {
	return repo.GetUser(id)
}

// 待測函式
func TestGetUserName(t *testing.T) {
	mockRepo := MockUserRepository{}
	user := GetUserName(mockRepo, 1)
	if user != "Mock User" {
		t.Errorf("Expected Mock User, got %s", user)
	}
}
```

---

## 7. 結論

Seam Model 提供了一個清晰的架構設計視角，透過設計可替換的行為點來提升程式的可測試性。藉由應用介面、依賴注入與設計模式，我們能夠有效地降低程式耦合，增強單元測試的靈活性與維護性。
