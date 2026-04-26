---
title: "C# volatile 詳解：理解多執行緒可見性 (Memory Visibility)"
date: 2026-03-21T01:53:00+08:00
categories: ["C#", "多執行緒"]
tags: ["C#", "volatile", "多執行緒", "記憶體"]
draft: true
---

當我們編寫多執行緒程式時，最容易被忽視的概念之一就是**記憶體可見性 (Memory Visibility)**。這個範例專案演示了在沒有適當同步機制的情況下，一個執行緒對變數的修改如何可能被另一個執行緒「視而不見」，以及 `volatile` 關鍵字如何解決這個問題。

## 核心問題：緩存與優化

現代 CPU 和編譯器為了效能，會做很多優化：
1.  **CPU 快取 (L1/L2 Cache)**：CPU 核心可能會將變數的值暫存在自己的快取中，而不是每次都去讀寫較慢的主記憶體 (RAM)。
2.  **指令重排 (Instruction Reordering)**：CPU 或編譯器可能會改變指令的執行順序以優化管線 (Pipeline)。
3.  **暫存器分配 (Register Allocation)**：編譯器 (JIT) 可能將變數直接存在 CPU 暫存器中，完全不寫回記憶體。

### 情境 A：沒有使用 `volatile`

在 `WithoutVolatileDemo` 類別中：

```csharp
private bool _running = true; // 普通欄位

// Worker Thread
while (_running) { /* ... */ }

// Main Thread
_running = false;
```

**發生了什麼事？**
-   **Release 模式下**，JIT 編譯器看到 `while (_running)` 迴圈內部沒有任何程式碼修改 `_running`。
-   它可能會將 `_running` 的值讀入 CPU 暫存器，並認為「這個值永遠不會變」。
-   於是迴圈變成了 `while (true)` 的死路。
-   即使主執行緒將記憶體中的 `_running` 改為 `false`，Worker 執行緒仍然在看它暫存器裡的舊值 (`true`)。

這就是**可見性問題**：主執行緒的寫入對 Worker 執行緒不可見。

### 情境 B：使用 `volatile`

在 `WithVolatileDemo` 類別中：

```csharp
private volatile bool _running = true; // 加上 volatile
```

**發生了什麼事？**
-   `volatile` 關鍵字告訴編譯器和 CPU：「這個變數可能會被其他執行緒隨時修改，不要對它做過度的讀取優化」。
-   **讀取時**：強制從主記憶體讀取，確保拿到最新值。
-   **寫入時**：強制立即寫回主記憶體，確保其他執行緒能看到。
-   它還會在讀寫操作前後插入**記憶體屏障 (Memory Barrier)**，防止特定類型的指令重排。

因此，當主執行緒將 `_running` 設為 `false`，Worker 執行緒在下一次迴圈判斷時，會被迫去主記憶體查值，從而正確地跳出迴圈。

## 何時使用 `volatile`？

`volatile` 適用於以下**非常特定**的情況：

1.  **狀態標誌 (State Flags)**：如本例中的 `_running` 或 `_cancelled` 布林值，用於控制執行緒的啟動/停止。
2.  **雙重檢查鎖定 (Double-Checked Locking)**：在 Singleton 模式的延遲初始化中，`volatile` 確保實例初始化的寫入順序正確。
3.  **簡單的計數器更新**：當只有一個執行緒寫入，多個執行緒讀取時。

## `volatile` 的限制

`volatile` **不保證原子性 (Atomicity)**！

如果你的操作是 `count++` (讀取 -> 加一 -> 寫回)，即使 `count` 是 `volatile`，在多執行緒同時寫入時仍然會發生 Race Condition（競爭條件），導致資料遺失。

-   如果要保證原子性（例如計數器），請使用 `Interlocked.Increment()`。
-   如果要保證一段程式碼的完整性，請使用 `lock` 陳述式。

## 原始碼參考

```csharp
// 位於 Program.cs
class WithVolatileDemo
{
    // volatile 保證：
    // 1. 每次讀取都從主記憶體取值
    // 2. 每次寫入都立即刷新到主記憶體
    private volatile bool _running = true;

    public void Run()
    {
        var worker = new Thread(() =>
        {
            // 每次迴圈都會從主記憶體重新讀取，一定能看到主執行緒的寫入。
            while (_running)
            {
                // ...
            }
        });
        
        // ...
        _running = false; // 寫入後立即對 worker 可見
        // ...
    }
}
```

## 參考

- 實作專案：https://github.com/hezhengmin/Project/tree/master/VolatileExample
