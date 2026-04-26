---
title: "C# Static Field 陷阱：在建構子中指派 this"
date: 2026-03-21T02:22:00+08:00
categories: ["C#", "程式設計"]
tags: ["C#", "static", "建構子", "陷阱"]
draft: false
---

這個專案示範了一個常見但危險的模式：在實例建構子（Instance Constructor）中將 `this` 指派給靜態欄位（Static Field）。這通常是為了方便全域存取「當前」物件，但往往會導致意想不到的錯誤。

## 核心問題

在 `Person` 類別中，我們看到這樣的程式碼：

```csharp
public class Person
{
    // 靜態欄位，所有實例共享同一個變數
    public static Person CurrentPerson;
    
    public string Name;

    public Person(string name, int age)
    {
        Name = name;
        // ...
        CurrentPerson = this; // 每次 new 一個新物件，都會覆蓋這個靜態欄位！
    }
}
```

### 發生了什麼事？

1.  當你建立 `Person A` 時，`CurrentPerson` 指向 A。
2.  當你建立 `Person B` 時，建構子再次執行，`CurrentPerson` 被更新指向 B。A 的全域參考就此遺失。
3.  當你建立 `Person C` 時，`CurrentPerson` 再次被更新指向 C。

這意味著 `Person.CurrentPerson` **永遠只指向最後被建立的那個物件**。如果你的系統依賴這個靜態屬性來存取「主要使用者」或「設定檔」，一旦程式的其他部分無意間建立了一個新的 `Person` 物件（例如暫時性的變數），原本的全域狀態就會被悄悄替換掉。

## 程式碼行為分析

執行 `Program.cs` 的輸出結果清楚地展示了這個現象：

```text
=== 建立 Person A ===
personA          : Name=Alice, Age=30
CurrentPerson    : Name=Alice, Age=30
是同一個實例嗎？ : True

=== 建立 Person B ===
personB          : Name=Bob, Age=25
CurrentPerson    : Name=Bob, Age=25   <-- CurrentPerson 變成了 Bob
是同一個實例嗎？ : True

...

=== 問題：personA 的引用已遺失 ===
CurrentPerson 還是 Alice 嗎？ : False
CurrentPerson 現在是誰？      : Ming
```

## 為什麼這很危險？

1.  **不可預測的狀態**：全域狀態依賴於物件建立的順序。這在大型應用程式中極難除錯。
2.  **記憶體洩漏風險**：靜態欄位是 GC Root。如果 `CurrentPerson` 意外地持有一個應該被釋放的大型物件（例如包含大量圖片資料的物件），那麼該物件將永遠不會被垃圾回收，因為靜態欄位一直參考著它（直到下一次被覆蓋）。
3.  **非執行緒安全**：如果在多執行緒環境下同時建立多個 `Person`，`CurrentPerson` 的最終值將取決於 Race Condition，無法確定最後是哪一個。

## 正確的做法

如果你需要一個全域唯一的實例（例如設定檔管理器），請使用 **Singleton Pattern（單例模式）**，並明確禁止外部隨意建立新實例：

```csharp
public class PersonManager
{
    // 1. 私有靜態實例
    private static PersonManager _instance;
    private static readonly object _lock = new object();

    // 2. 私有建構子，防止外部 new
    private PersonManager() { }

    // 3. 公開靜態存取點
    public static PersonManager Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                    {
                        _instance = new PersonManager();
                    }
                }
            }
            return _instance;
        }
    }
}
```

或者，在現代 .NET Core / ASP.NET Core 應用中，使用 **Dependency Injection (DI)** 並將服務註冊為 `Singleton` 是最佳實務。

## 原始碼參考

```csharp
// 位於 Program.cs
public class Person
{
    public static Person CurrentPerson;
    public string Name;
    public int Age;

    public Person() : this("Unknown", 0) { }

    public Person(string name, int age)
    {
        Name = name;
        Age = age;
        CurrentPerson = this; // 陷阱：每次 new 都會覆蓋
    }

    public override string ToString() => $"Name={Name}, Age={Age}";
}
```

## 參考

- 實作專案：https://github.com/hezhengmin/Project/tree/master/StaticThisDemo


