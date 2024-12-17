---
title: "用 Pub/Sub Pattern 解耦 Use Case 之間不必要的依賴"
date: 2024-12-17T00:00:00+08:00
tags:
- Software Development
- Design Pattern
---

在專案上的新功能開發，團隊都盡量以 Clean Architecture 為指導原則，從 Domain、Use Case 到 Handler 層層分離。這種分層確實帶來了更清楚的職責界線，但也不是每次都這麼順利，有時還是會遇到實作上的「卡點」。

最近實際遇到的問題，就是 **某一 Use Case 依賴了另一個 Use Case**，這其實違反了 Use Case 間應各自獨立的精神。 這篇就來記錄當時主要遇到的問題，以及最後如何用 Pub/Sub Pattern 重構解決。

## 問題：Use Case 之間的依賴長什麼樣子？
以某個 domain 下的 Notification Use Case 為例：
它會根據傳入的 orderID、userID 呼叫外部 API 發送推播通知。

假設有兩個 Use Case - 初始化參加活動資料，B - 根據訂單狀態更新活動資料

這兩個 Use Case 都需要在處理後通知用戶，所以他們各自都依賴 **Notification Use Case 介面**，像這樣：
```go
// 用戶參加活動，A Use Case
type ActivityUsecase struct {
	notificationUsecase NotificationUsecase
	// ...其他依賴...
}

func (uc *ActivityUsecase) InitActivity(ctx context.Context, req InitActivityRequest) error {
	// ...初始化活動資料...
	// 業務完成後直接呼叫 NotificationUsecase 發通知
	return uc.notificationUsecase.SendActivityCreated(ctx, req.UserID, req.OrderID)
}

// B Use Case 同理
// ...
```
雖然 A, B Use Case 已經把「通知」這件事委託出去了，依賴的是抽象的 Notification Use Case 介面，但仍然需要知道 Notification Use Case 的存在。這違反了 Use Case 各自獨立、單一職責的原則。
> Uncle Bob 在《The Clean Architecture》中強調，依賴應該指向內層，Use Case 應保持獨立，避免相互依賴，以維持系統的可維護性和靈活性。

## 重構思路
既然目標是讓 Use Case A, B 完全不知道 Notification Use Case 的存在，Pub/Sub Pattern 就是很適合的選擇。核心做法是：

- 用 Event 物件作為中介，用來描述「某件事發生了」。
- 讓負責處理通知的 handler 註冊為訂閱者，由 Event Bus 廣播事件給它們。
- Use Case 只需專心發送事件，不需知道有沒有人會「收到」或怎麼處理。

## 重構實作

1. 定義事件與 EventBus
```go
// event.go
type Event interface {
	Name() string
}

type Handler interface {
	Handle(ctx context.Context, event Event) error
}

// eventbus.go
type InMemoryEventBus struct {
	handlers map[string][]Handler
	mu       sync.RWMutex
}

func NewInMemoryEventBus() *InMemoryEventBus {
	return &InMemoryEventBus{
		handlers: make(map[string][]Handler),
	}
}

func (bus *InMemoryEventBus) Publish(ctx context.Context, event Event) {
	bus.mu.RLock()
	defer bus.mu.RUnlock()
	handlers, ok := bus.handlers[event.Name()]
	if !ok {
		return
	}
	for _, handler := range handlers {
		_ = handler.Handle(ctx, event)
	}
}

func (bus *InMemoryEventBus) Subscribe(eventName string, handler Handler) {
	bus.mu.Lock()
	defer bus.mu.Unlock()
	bus.handlers[eventName] = append(bus.handlers[eventName], handler)
}
```

2. 定義活動相關事件
```go
// event/activity_event.go
type ActivityCreated struct {
	UserID  string
	OrderID string
}

func (e ActivityCreated) Name() string { return "ActivityCreated" }
```

3. 實作 Notification Handler
```go
// handler/notification_handler.go
type NotifyUserOnActivityCreated struct {
	notificationUsecase NotificationUsecase
}

func (h NotifyUserOnActivityCreated) Handle(ctx context.Context, event Event) error {
	e, ok := event.(ActivityCreated)
	if !ok {
		return fmt.Errorf("unexpected event type: %T", event)
	}
	return h.notificationUsecase.SendActivityCreated(ctx, e.UserID, e.OrderID)
}
```

4. Use Case 只需發送事件，不需關注通知邏輯
```go
// usecase/activity.go
type ActivityUsecase struct {
	eventBus EventBus
	// ...其他依賴...
}

func (uc *ActivityUsecase) InitActivity(ctx context.Context, req InitActivityRequest) error {
	// ...業務邏輯...
	event := ActivityCreated{UserID: req.UserID, OrderID: req.OrderID}
	uc.eventBus.Publish(ctx, event)
	return nil
}
```

5. 在啟動時註冊訂閱者
```go
// main.go 或 wire.go
eventBus := NewInMemoryEventBus()
notificationUsecase := NewNotificationUsecase(...)

eventBus.Subscribe("ActivityCreated", NotifyUserOnActivityCreated{notificationUsecase})
```

### 重構後的好處

- 單一職責更明確：Use Case 不需知道 Notification 的存在，後續要新增/異動通知行為只需註冊或退訂事件即可。
- 更易於測試：測試 Use Case 不用 Mock Notification，只驗證有無發事件。
- 維護彈性大：日後想加其他副作用處理，只需新加 handler，完全不動原本業務邏輯。


### 小結
這次遇到的問題，看似只是 Use Case 依賴多一層 Notification Use Case，實際上卻讓系統多了一層耦合、降低了彈性。即使依賴的是介面，只要這個介面本質上還是代表另一個 Use Case 的職責，耦合依然存在。

透過引入 Pub/Sub Pattern，用事件傳遞，讓業務流程之間的協作改為事件觸發，而不是互相呼叫，這樣每個 Use Case 都能專注在自己的職責上。

回頭看，其實還是回到分層設計和職責單純這件事。當然還有許多細節可以再優化，像是事件訂閱與錯誤處理怎麼做、事件流程是否需要同步/非同步等等。

如果剛好看到這篇的你對這個議題有經驗或其他看法，也歡迎留言一起討論！