# 使用 Gomock 進行測試的完整指南

## 前言

Gomock 是一個用於 Golang 的 Mock 框架。其核心功能是自動生成模擬物件（Mock Objects），開發者能夠模擬介面中定義的方法，設定模擬的回應，並檢查行為是否符合預期。

適用場景：

- 資料庫查詢的模擬
- 外部 API 呼叫的虛擬回應
- 網路與文件系統操作的測試
- 系統間的整合與行為驗證

---

## Gomock 的核心概念

### 1. Mock 物件

以下是建立一個簡單的 Mock 物件範例，模擬 `UserService` 介面的行為。

```go
package example

type UserService interface {
    GetUser(id int) (string, error)
}
```

mockgen 是 Gomock 提供的命令列工具，用於自動生成 Mock 物件。

```bash
mockgen -source=example/user_service.go -destination=example/mock_user_service.go
```

生成的 Mock 物件會依照介面提供對應的方法。在測試環境中，實際呼叫的是 Mock 物件，但在待測程式碼中看到的仍然是相同的介面。

---

### 2. 控制器 (Controller)

使用 `gomock.NewController` 建立控制器，並初始化 Mock 物件。

```go
package example_test

import (
	"testing"
	"example"

	"github.com/golang/mock/gomock"
)

func TestGetUser(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish() // 釋放資源，驗證所有 Mock 行為

	// 使用自動生成的 Mock 檔案
	mockService := example.NewMockUserService(ctrl)

	// 設定期望與行為
	mockService.EXPECT().GetUser(1).Return("Alice", nil)
}
```

---

### 3. 期望設定 (Expectations)

在測試過程中設定方法呼叫的預期行為，定義回傳值與錯誤。

```go
func TestGetUserWithExpectations(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)

	// 設定期望行為與回傳值
	mockService.EXPECT().GetUser(1).Return("Alice", nil)
	mockService.EXPECT().GetUser(2).Return("", fmt.Errorf("user not found"))

	// 呼叫方法進行測試
	user, err := mockService.GetUser(1)
	if err != nil || user != "Alice" {
		t.Errorf("expected Alice, got %v, err %v", user, err)
	}

	// 測試錯誤回傳情境
	_, err = mockService.GetUser(2)
	if err == nil {
		t.Errorf("expected an error for unknown user")
	}
}
```

---

### 4. 行為驗證 (Assertions)

在測試完成後，控制器會自動驗證 Mock 物件是否符合預期。如果呼叫行為與期望不符，Gomock 會回傳詳細的錯誤訊息。

範例：未達到預期行為的錯誤訊息

```shell
Expected call at least once: GetUser(1)
```

此訊息表示 `GetUser(1)` 方法未被預期次數呼叫，表示測試邏輯未達成預期的業務行為。

---

## Gomock 的進階功能與應用

### 1. 參數匹配器 (Argument Matchers)

Gomock 支援參數匹配器，允許開發者對傳入的參數進行條件匹配，而不僅限於固定值。

```go
package example_test

import (
	"testing"
	"example"

	"github.com/golang/mock/gomock"
)

func TestArgumentMatchers(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)

	// 精確匹配傳入值
	mockService.EXPECT().GetUser(gomock.Eq(1)).Return("Alice", nil)

	// 匹配任意整數
	mockService.EXPECT().GetUser(gomock.Any()).Return("Unknown", nil)

	_, _ = mockService.GetUser(1)
	_, _ = mockService.GetUser(999)
}
```

---

### 2. 呼叫次數控制 (Call Times Control)

設定待測程式期望的呼叫次數

```go
func TestCallTimes(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)

	// 設定必須被呼叫兩次
	mockService.EXPECT().GetUser(gomock.Any()).Times(2).Return("Test User", nil)

  // 必須呼叫兩次，否則 test fail
	_, _ = mockService.GetUser(100)
	_, _ = mockService.GetUser(200)
}
```

---

### 3. 順序控制 (Call Order Verification)

驗證方法是否按照預期順序被呼叫。

```go
func TestCallOrder(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)
	callOrder := gomock.InOrder(
		mockService.EXPECT().GetUser(1).Return("Alice", nil),
		mockService.EXPECT().GetUser(2).Return("Bob", nil),
	)

	_, _ = mockService.GetUser(1) // Alice
	_, _ = mockService.GetUser(2) // Bob
	if !callOrder {
		t.Error("Method calls did not match expected order")
	}
}
```

---

### 4. 回應模擬 (Simulating Responses)

根據不同的條件模擬 Mock Object 的回應，適用於仿造來自外部依賴的不同情境，例如外部 api 或資料庫查詢。

```go
func TestSimulateResponses(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)

	// 靜態回應設定
	mockService.EXPECT().GetUser(1).Return("Alice", nil)

	// 動態回應設定
	mockService.EXPECT().GetUser(gomock.Any()).DoAndReturn(func(id int) (string, error) {
		if id == 2 {
			return "Bob", nil
		}
		return "Unknown", nil
	})

	_, _ = mockService.GetUser(1)  // 回傳 Alice
	_, _ = mockService.GetUser(2)  // 回傳 Bob
	_, _ = mockService.GetUser(99) // 回傳 Unknown
}
```

---

### 5. 錯誤模擬 (Error Simulation)

模擬方法回傳錯誤，測試錯誤處理邏輯。

```go
func TestErrorSimulation(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockService := example.NewMockUserService(ctrl)

	// 模擬成功與錯誤回應
	mockService.EXPECT().GetUser(1).Return("Alice", nil)
	mockService.EXPECT().GetUser(2).Return("", fmt.Errorf("user not found"))

	// 測試正常情境
	user, err := mockService.GetUser(1)
	if err != nil || user != "Alice" {
		t.Errorf("expected Alice, got %v, err %v", user, err)
	}

	// 測試錯誤回應
	_, err = mockService.GetUser(2)
	if err == nil {
		t.Errorf("expected an error but got none")
	}
}
```
