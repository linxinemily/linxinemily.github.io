---
title: Message Broker? Message Queue? 一次搞懂 SQS + Lambda 運作方式
date: 2025-09-20
tags:
  - AWS
  - Serverless
---

## TL;DR

在設計大型資料密集型系統時，經常需要處理大量資料。由於這些處理工作通常既大量又耗時，因此多採用非同步方式處理。此時常需要引入 Message Broker 等相關組件，來解耦「資料傳遞與（緩）儲存」和「接收訊息處理工作」之間的關係。

在 AWS 架構中，常聽到的 SQS、Kinesis、MQ 等服務，都與 Message 的傳遞和處理相關。此外，一些相關概念和名詞也容易混淆，如 Message Broker、Message Queue、Message Stream 等。

本文從 Message Broker 的定位談起，再比較 Message Queue 與 Message Stream，最後聚焦於 SQS + Lambda 的整合，包含運作原理與最佳實踐，一次弄懂這些概念與背後運作方式；內容多參考官方文件並加以歸納整理。

## 什麼是 Message Broker

### 定義

Message Broker 是一種中介服務，負責在不同系統間傳遞與緩衝訊息。常見的訊息傳遞模型有 Queue 和 Stream，不同的 broker 系統（SQS、Kinesis、MQ）會支援不同模型。

### Message Queue vs Message Stream

#### 相似之處

1. 它們都有一個或多個生產者（producers）創建訊息，訊息被存儲在 queue 或 stream 中，然後由一個或多個消費者（consumers）透過輪詢來處理這些訊息。
2. 訊息都不會永久保存 - 都有一個保留期限來決定每條訊息的最大存在時間。

#### 區別

|                | Message Queue                          | Message Stream                       |
| -------------- | -------------------------------------- | ------------------------------------ |
| 訊息處理方式   | 競爭模式：只有一個消費者能處理特定訊息 | 共享模式：所有消費者都能處理所有訊息 |
| 訊息存留       | 讀取後刪除                             | 讀取後保留                           |
| 代表服務       | SQS                                    | Kinesis                              |
| 適合場景       | 任務分配和工作負載分散                 | 事件廣播、即時分析和資料複製         |
| 重複處理容忍度 | 低                                     | 高                                   |

因此，這兩種模型都可以作為 Lambda 的 producer，而 Lambda 可以作為它們的 consumer，去消費這些訊息池裡面的 message。

## Lambda 與事件來源（Event Source）整合模型

Lambda 可以和多種事件來源（Event Source）整合，這些事件來源可以分為 Push-based vs Pull-based 模型。

### Push-based

Lambda 扮演**被動**的角色，由其他服務透過 Trigger 主動調用 Lambda function。根據呼叫端服務的特性，調用方式可分為同步或非同步兩種。

|                | 同步                        | 非同步                                 |
| -------------- | --------------------------- | -------------------------------------- |
| 是否會等待結果 | 是                          | 否                                     |
| 重試行為       | client 端負責重試           | Lambda 服務會負責重試（最多重試 2 次） |
| 常見呼叫端服務 | API Gateway, Step Functions | S3, SNS                                |

### Pull-based

Lambda 作為 Message streams/queues 的 consumer，透過 **Event Source Mapping** 機制**主動**去輪詢（poll）message streams/queues，從中拉取（pull）事件/訊息來處理。

> Event Source Mapping 是 Lambda 服務內建的功能，會負責執行 poller 程式去拉取這些 message streams/queues 裡的訊息、追蹤處理訊息的狀態並管理批次處理和重試邏輯。

Lambda 支援的 pull-based producers：

- message streams: Kinesis, DynamoDB, DocumentDB, Self-managed Kafka, MSK
- message queues: SQS（包含 FIFO）, MQ

## SQS + Lambda 運作原理和官方最佳實踐

當使用 SQS 作為 Lambda 的事件來源時，Lambda 會透過 Event Source Mapping 機制輪詢 SQS 佇列並**批次**接收訊息，然後同步調用函數一次處理整個批次。函數成功處理後，Lambda 會從佇列中刪除該訊息。

### 批次處理設定

幾個比較重要的參數包括：

- `Batch size`
  - 每次從 SQS 拉取的最大訊息數
  - 可以在新增/編輯 Lambda 的 SQS Trigger 時自定義（雖然背後是建立 Event Source Mapping，但還是用 Trigger 來設置），預設為 10 筆。若為 FIFO 最多 10 筆，若為標準佇列最多可以為 10,000 筆
- `Batch window`：
  - 簡單說就是「願意等多久來湊滿一批」
  - 設為
    - `0`：不等待，來一個處理一個（或同時來多個就一起處理）
    - `> 0`：會在這段時間內收集訊息，再一次處理

### 訊息處理流程

當消費者接收到 SQS 裡的訊息開始處理時，該訊息會對其他消費者隱藏。當處理成功時，訊息會從佇列裡被刪除；當處理失敗或處理時間超過訊息的 Visibility Timeout 時，訊息會被標記為重新可見，可以再次被其他消費者處理。

> Visibility Timeout 可以在 SQS 的設定中配置，預設為 30 秒（針對佇列裡的每則訊息）

由於 Lambda 讀取 SQS 訊息時是以批次方式讀取，因此如果批次設定為一次 10 筆資料，Visibility Timeout 為 30 秒，表示這 10 筆會同時被觸發 Visibility Timeout，因此 Lambda 必須要在 30 秒內處理完這 10 筆資料才算成功，否則所有訊息就會在超時後重新回到佇列中可見。

處理失敗的情況包含 **function code 的錯誤**或是 **Lambda 服務限流（throttling）**：

- function code 錯誤：當處理一批訊息時，其中一則回傳失敗，Lambda 就會停止處理後面的訊息，等 Visibility Timeout 時間到，訊息就會重新在佇列裡可見
- throttling 導致失敗：訊息一樣會等 Visibility Timeout 時間到，就會重新在佇列裡可見

訊息在佇列裡重新可見後，會在下次被 poller（aka Event Source Mapping）輪詢到（AWS 通常使用包含抖動的指數退避模式來重試），並再次把整批訊息丟給 Lambda 函數處理。此過程會一直持續，直到訊息本身過期被 SQS 自動刪除（預設的訊息保留期間為 4 天，最長 14 天）。

因此，若沒有特別處理這些失敗訊息，它們就會不斷堆積在佇列裡一直被重試直到過期。同時，如果當整批訊息還沒全部處理完時就超時，也會導致整批訊息重新回到佇列當中可見，又被其他消費者撈去處理，這時就會有同個訊息被重複處理的狀況。那要如何才能避免這些情況呢？

### 官方最佳實踐

針對不同情況，對應的官方建議最佳實踐方式如下：

#### 設計階段的預防策略

**確保函數冪等性**：設計函數時確保重複執行不會產生副作用，這樣即使訊息被重複處理也不會有問題。

#### 訊息處理失敗時

當批次處理中部分訊息失敗時，可採用以下策略：

1. **實作 Batch item failure 機制**：在批次處理中只回傳部分失敗的項目，Lambda 會自動只刪除成功處理的訊息，失敗的會保留在佇列中

   > - `batchItemFailures` 可以是一個包含多個 record id 的 list
   >
   > - 在 SQS FIFO 隊列的批次處理中，list 必須包含所有失敗和未處理的訊息。當批次中某訊息處理失敗時，應立即停止處理後續訊息，並將失敗訊息及所有未處理訊息都包含在  `batchItemFailures`  中。例如，若訊息 B 失敗，必須報告 B 及後續的 C、D、E，否則會破壞 FIFO 順序。
   >
   > - 但 `batchItemFailures`  中訊息 ID 的排列順序不影響處理。無論以何種順序返回 ID，SQS 都會按原始順序重新處理這些訊息。

2. **手動控制訊息刪除**：使用 SQS DeleteMessage API，成功處理後立即刪除訊息，確保只有失敗的訊息會回到佇列中

#### 訊息處理超時時

將 SQS visibility timeout 設定為至少 6 倍的 Lambda function timeout，讓所有訊息有足夠的時間被處理完畢。

此外，當 Lambda 因為達到並發限制而被限流時，設定較長的 visibility timeout 可以確保訊息有足夠的時間等待限流解除後被處理，避免訊息過早重新變為可見狀態而被重複處理。

#### 存在無法處理的訊息時

如果一直持續有 Lambda 無法處理的訊息回到 SQS 裡面重新可見，會形成所謂的「snowball 反模式」。

因此官方建議可以透過配置 DLQ（Dead Letter Queue），設定最大重試次數（maxReceiveCount），讓這些多次無法成功處理的訊息轉到 DLQ，以便後續人工或自動處理／分析。

## 小結

透過以上介紹，可以理解到 SQS 是一種採用 Queue 模型傳遞訊息的 Message Broker，同時也整理出 Message Broker 常見的兩種模型：Queue 與 Stream，並釐清它們的異同與適用情境。

在此基礎上，進一步說明了 Lambda 與其他服務的整合模式（Push-based 與 Pull-based），以及 Lambda 如何透過 Event Source Mapping 機制處理 SQS 訊息，包括錯誤處理與逾時機制，以及官方提供的最佳實務建議。

雖然過度糾結於名詞定義並非必要，但對齊這些概念能建立統一的溝通語言，讓討論更有效率；同時，透過理解名詞背後的語義與設計考量，也能更清楚掌握各元件之間的關聯與運作方式。

## 參考資料

- [Lambda 非同步調用的錯誤和重試](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-error-handling.html)
- [非同步調用 Lambda 函數](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async.html)
- [Event Source Mapping](https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html)
- [搭配 Amazon SQS 使用 Lambda](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html#sqs-polling-behavior)
- [Visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Lambda SQS 事件來源錯誤處理](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html)
- [Amazon SQS 開發人員指南](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [建立和設定 Amazon SQS 事件來源映射](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)
- [使用退避模式重試](https://docs.aws.amazon.com/zh_cn/prescriptive-guidance/latest/cloud-design-patterns/retry-backoff.html)
- [實作部分批次回應的最佳實務](https://docs.aws.amazon.com/prescriptive-guidance/latest/lambda-event-filtering-partial-batch-responses-for-sqs/best-practices-partial-batch-responses.html#snowball-anti-patterns)
- [Lambda SQS 批次項目失敗報告](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-errorhandling.html#services-sqs-batchfailurereporting)

<small>如果有有任何問題或指教，歡迎下面留言 ʕ•ᴥ•ʔ</small>
