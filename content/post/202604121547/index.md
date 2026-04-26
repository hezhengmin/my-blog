---
title: "ASP.NET Core 自訂 RBAC 權限授權：PermissionAuthorizationHandler 實戰"
date: 2026-04-12T15:47:59+08:00
categories: ["後端技術", "權限管理"]
tags: ["ASP.NET Core", "RBAC", "權限", "教學"]
draft: false
---

## 前言

在 ASP.NET Core 中，最常見的授權方式是用 `[Authorize(Roles = "Admin")]` 來做角色驗證。但當系統規模變大，你會發現**角色（Role）與權限（Permission）是兩個不同層次的概念**：

| 項目 | Role-Based | Permission-Based (RBAC) |
|------|-----------|------------------------|
| 粒度 | 粗（整個角色） | 細（單一操作） |
| 彈性 | 新增角色需改程式碼 | 角色綁定權限可動態調整 |
| 維護 | 角色越多越難管理 | 權限固定，角色自由組合 |
| 範例 | `[Authorize(Roles = "Admin")]` | `[Permission("Books.Create")]` |

**RBAC（Role-Based Access Control）** 的核心思想是：

> 使用者 → 角色 → 權限

使用者不直接擁有權限，而是透過被指派的角色間接取得權限。這樣只要調整角色的權限組合，就能批量調整一群使用者的存取能力，不需要改程式碼。

本文以 **Library 圖書管理系統** 的真實實作為範例，帶你從零建立完整的 Permission-Based 授權機制。

---

## 架構總覽

### 資料模型關係

```
┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────────────┐
│   User   │────▶│   UserRole   │◀────│   Role   │────▶│ RolePermission   │
│          │     │              │     │          │     │                  │
│ Id       │     │ UserId (FK)  │     │ Id       │     │ Id               │
│ UserName │     │ RoleId (FK)  │     │ Name     │     │ RoleId (FK)      │
│ Email    │     │ AssignedAt   │     │ ...      │     │ Permission       │
│ ...      │     └──────────────┘     └──────────┘     │ (e.g."Books.View")│
└──────────┘                                           └──────────────────┘
         1 : N                              1 : N
```

- **User ↔ UserRole ↔ Role**：多對多關聯（透過 Join Table `UserRole`）
- **Role → RolePermission**：一對多關聯，一個角色可擁有多個權限字串

### 授權請求流程

```
HTTP Request
     │
     ▼
┌─────────────────┐
│  JWT 驗證中介層   │  ← 解析 Token，取出 ClaimsPrincipal
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  [Permission] Attribute │  ← Controller/Action 上標記需要的權限
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│    PermissionFilter     │  ← IAsyncAuthorizationFilter
│    (IAsyncAuthFilter)   │     呼叫 IAuthorizationService
└────────┬────────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  PermissionAuthorizationHandler  │  ← AuthorizationHandler<PermissionRequirement>
│                                  │
│  1. 從 Claims 取出 UserId        │
│  2. 呼叫 PermissionService       │
│  3. 比對是否有任一所需權限 (OR)    │
└────────┬─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│   PermissionService     │  ← 查詢 DB: UserRole → Role → RolePermission
│   (查詢資料庫)           │
└────────┬────────────────┘
         │
         ▼
┌────────┴────────┐
│                 │
▼                 ▼
✅ 200 OK      ❌ 403 Forbidden
(有權限)       (無權限 → "您沒有此操作權限")
```

---

## Step 1：定義權限常數

首先，我們用 **巢狀靜態類別** 定義所有權限字串，集中管理、避免魔術字串：

```csharp
// Library.Server/Filters/PermissionAuthorization.cs

namespace Library.Server.Authorization
{
    /// <summary>
    /// 權限常數定義
    /// </summary>
    public static class Permissions
    {
        /// <summary>
        /// 圖書管理權限
        /// </summary>
        public static class Books
        {
            public const string View = "Books.View";
            public const string Create = "Books.Create";
            public const string Edit = "Books.Edit";
            public const string Delete = "Books.Delete";
        }

        /// <summary>
        /// 使用者管理權限
        /// </summary>
        public static class Users
        {
            public const string View = "Users.View";
            public const string Create = "Users.Create";
            public const string Edit = "Users.Edit";
            public const string Delete = "Users.Delete";
            public const string ResetPassword = "Users.ResetPassword";
        }

        /// <summary>
        /// 角色管理權限
        /// </summary>
        public static class Roles
        {
            public const string View = "Roles.View";
            public const string Create = "Roles.Create";
            public const string Edit = "Roles.Edit";
            public const string Delete = "Roles.Delete";
        }

        /// <summary>
        /// 系統管理權限
        /// </summary>
        public static class System
        {
            public const string Menu = "System.Menu";
            public const string MenuCreate = "System.Menu.Create";
            public const string MenuUpdate = "System.Menu.Update";
            public const string MenuDelete = "System.Menu.Delete";
        }
    }
}
```

**設計重點：**
- 用 `模組.操作` 的命名慣例（如 `Books.View`），語意清晰
- 每個模組一個巢狀 class，方便 IntelliSense 自動完成
- 常數字串不會打錯，編譯期就會檢查

---

## Step 2：實作 PermissionRequirement

`IAuthorizationRequirement` 是 ASP.NET Core 授權機制的**需求介面**。我們建立一個 `PermissionRequirement` 來存放「這次操作需要哪些權限」：

```csharp
// Library.Server/Filters/PermissionRequirement.cs

using Microsoft.AspNetCore.Authorization;

namespace Library.Server.Filters
{
    public class PermissionRequirement : IAuthorizationRequirement
    {
        public string[] Permissions { get; }

        public PermissionRequirement(params string[] permissions)
        {
            Permissions = permissions;
        }
    }
}
```

**說明：**
- `Permissions` 是一個字串陣列，代表「此操作需要的權限清單」
- 使用 `params` 讓呼叫端可以傳入多個權限：`new PermissionRequirement("Books.View", "Books.Create")`
- 這個類別只負責**攜帶資料**，不包含邏輯

---

## Step 3：實作 PermissionAuthorizationHandler

這是整個機制的**核心**——繼承 `AuthorizationHandler<PermissionRequirement>`，負責判斷「使用者是否擁有所需權限」：

```csharp
// Library.Server/Filters/PermissionAuthorizationHandler.cs

using Microsoft.AspNetCore.Authorization;
using System.Security.Claims;
using Zheng.Bll.Interfaces;

namespace Library.Server.Filters
{
    public class PermissionAuthorizationHandler : AuthorizationHandler<PermissionRequirement>
    {
        private readonly IPermissionService _permissionService;

        public PermissionAuthorizationHandler(IPermissionService permissionService)
        {
            _permissionService = permissionService;
        }

        protected override async Task HandleRequirementAsync(
            AuthorizationHandlerContext context,
            PermissionRequirement requirement)
        {
            // ① 如果沒有指定權限，則直接通過（允許存取）
            if (requirement.Permissions == null || requirement.Permissions.Length == 0)
            {
                context.Succeed(requirement);
                return;
            }

            // ② 取得使用者 ID（從 JWT Claims 中擷取）
            var userIdStr = context.User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userIdStr) || !Guid.TryParse(userIdStr, out var userId))
            {
                return; // 無法辨識使用者 → 授權失敗（不呼叫 Succeed）
            }

            // ③ 透過 PermissionService 從 DB 取得使用者所有權限
            var userPermissions = await _permissionService.GetUserPermissionsAsync(userId);

            // ④ 檢查是否有「任一」所需的權限（OR 邏輯）
            foreach (var requiredPermission in requirement.Permissions)
            {
                if (userPermissions.Contains(requiredPermission))
                {
                    context.Succeed(requirement);
                    return;
                }
            }

            // ⑤ 都沒有符合的權限 → 不呼叫 Succeed → 授權失敗
        }
    }
}
```

**關鍵邏輯解析：**

| 步驟 | 說明 |
|------|------|
| ① | 如果 `[Permission]` 沒帶參數，等同不檢查權限，直接放行 |
| ② | 從 `ClaimsPrincipal` 取出 `NameIdentifier`（即 UserId），這是 JWT Token 解析後自動帶入的 |
| ③ | 呼叫 `PermissionService` 查詢 DB（UserRole → RolePermission），取得使用者的所有權限字串 |
| ④ | **OR 邏輯**：只要使用者擁有其中一個權限就通過。例如 `[Permission("Books.View", "Books.Edit")]` 只要有其一就放行 |
| ⑤ | ASP.NET Core 的授權機制是「沒有呼叫 `context.Succeed()` 就視為失敗」，不需要顯式 Fail |

---

## Step 4：實作 PermissionFilter + PermissionAttribute

有了 Handler，我們需要一個方便的方式在 Controller 上使用。這裡用 **TypeFilterAttribute + IAsyncAuthorizationFilter** 的組合：

### PermissionFilter（授權過濾器）

```csharp
// Library.Server/Filters/PermissionFilter.cs

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using System.Net;

namespace Library.Server.Filters
{
    public class PermissionFilter : Attribute, IAsyncAuthorizationFilter
    {
        private readonly IAuthorizationService _authorizationService;
        private readonly PermissionRequirement _requirement;

        public PermissionFilter(
            IAuthorizationService authorizationService,
            PermissionRequirement requirement)
        {
            _authorizationService = authorizationService;
            _requirement = requirement;
        }

        public async Task OnAuthorizationAsync(AuthorizationFilterContext context)
        {
            // 呼叫 ASP.NET Core 內建的 AuthorizationService
            // 它會找到對應的 Handler (PermissionAuthorizationHandler) 來處理
            var result = await _authorizationService.AuthorizeAsync(
                context.HttpContext.User, null, _requirement);

            if (!result.Succeeded)
            {
                // 授權失敗 → 回傳 403 Forbidden
                context.Result = new JsonResult("您沒有此操作權限")
                {
                    StatusCode = (int)HttpStatusCode.Forbidden
                };
            }
        }
    }
}
```

### PermissionAttribute（便利用 Attribute）

```csharp
// Library.Server/Filters/PermissionAttribute.cs

using Microsoft.AspNetCore.Mvc;

namespace Library.Server.Filters
{
    public class PermissionAttribute : TypeFilterAttribute
    {
        public PermissionAttribute(params string[] codes)
            : base(typeof(PermissionFilter))
        {
            // 將權限碼包成 PermissionRequirement，傳給 PermissionFilter
            Arguments = new[] { new PermissionRequirement(codes) };
        }
    }
}
```

**為什麼使用 `TypeFilterAttribute`？**

`TypeFilterAttribute` 是 ASP.NET Core 提供的特殊基底類別，它能讓你的 Attribute **支援依賴注入**。流程如下：

```
[Permission("Books.View")]
         │
         ▼
PermissionAttribute : TypeFilterAttribute
    ├── 指定 Filter 類型 = typeof(PermissionFilter)
    └── 傳入參數 = new PermissionRequirement("Books.View")
         │
         ▼
ASP.NET Core 的 DI 容器自動建立 PermissionFilter
    ├── 注入 IAuthorizationService  ← DI 自動解析
    └── 注入 PermissionRequirement  ← 從 Arguments 取得
```

---

## Step 5：實作 PermissionService

`PermissionService` 負責查詢資料庫，取得使用者的所有權限：

### 介面定義

```csharp
// Zheng.Bll/Interfaces/IPermissionService.cs

namespace Zheng.Bll.Interfaces
{
    public interface IPermissionService
    {
        Task<IList<string>> GetUserPermissionsAsync(Guid userId);
    }
}
```

### 實作

```csharp
// Zheng.Bll/Services/PermissionService.cs

using Microsoft.EntityFrameworkCore;
using Zheng.Bll.Interfaces;
using Zheng.Infra.Data.Models;

namespace Zheng.Bll.Services
{
    public class PermissionService : IPermissionService
    {
        private readonly LibraryContext _context;

        public PermissionService(LibraryContext context)
        {
            _context = context;
        }

        public async Task<IList<string>> GetUserPermissionsAsync(Guid userId)
        {
            return await _context.UserRoles
                .Where(ur => ur.UserId == userId)       // 1. 找到使用者的所有角色
                .SelectMany(ur => ur.Role.RolePermissions) // 2. 展開每個角色的權限
                .Select(rp => rp.Permission)             // 3. 取出權限字串
                .Distinct()                              // 4. 去重複
                .ToListAsync();
        }
    }
}
```

**LINQ 查詢邏輯拆解：**

```
UserRole 表                    RolePermission 表
┌────────┬────────┐           ┌────┬────────┬────────────┐
│ UserId │ RoleId │           │ Id │ RoleId │ Permission │
├────────┼────────┤           ├────┼────────┼────────────┤
│ A001   │ R01    │ ──JOIN──▶ │ 1  │ R01    │ Books.View │
│ A001   │ R02    │           │ 2  │ R01    │ Books.Edit │
└────────┴────────┘           │ 3  │ R02    │ Users.View │
                              └────┴────────┴────────────┘

結果：["Books.View", "Books.Edit", "Users.View"]
```

一個使用者可能有多個角色，每個角色有多個權限，`SelectMany` 把所有權限攤平，`Distinct` 去除重複。

---

## Step 6：DI 註冊

在 `Program.cs` 中註冊所有必要的服務：

```csharp
// Library.Server/Program.cs

// 1. 註冊 PermissionAuthorizationHandler
builder.Services.AddScoped<IAuthorizationHandler, PermissionAuthorizationHandler>();

// 2. 註冊 PermissionService
builder.Services.AddScoped<IPermissionService, PermissionService>();

// 3. 註冊 HttpContextAccessor（CurrentUser 需要）
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUser, CurrentUser>();
```

**為什麼用 `AddScoped`？**
- `Scoped` = 每個 HTTP Request 建立一個實例
- 權限查詢依賴當前使用者的 Request Context，不能用 `Singleton`
- 每次 Request 查一次權限即可，不需要 `Transient`（每次注入都建立新的）

---

## Step 7：Controller 套用範例

現在可以直接在 Controller 上使用 `[Permission]` Attribute：

```csharp
// Library.Server/Controllers/RoleController.cs

[Route("api/[controller]")]
[ApiController]
[Authorize]  // ← 先確認已登入（有合法 JWT Token）
public class RoleController : ControllerBase
{
    private readonly IRoleService _roleService;

    public RoleController(IRoleService roleService)
    {
        _roleService = roleService;
    }

    [HttpGet]
    [Permission(Permissions.Roles.View)]  // ← 需要 "Roles.View" 權限
    public async Task<IActionResult> GetRoles([FromQuery] RoleQueryParameters query)
    {
        var result = await _roleService.GetRolesAsync(query);
        return Ok(result);
    }

    [HttpPost]
    [Permission(Permissions.Roles.Create)]  // ← 需要 "Roles.Create" 權限
    public async Task<IActionResult> CreateRole([FromBody] CreateRoleDto dto)
    {
        var result = await _roleService.CreateRoleAsync(dto);
        return Ok(result);
    }

    [HttpPut("{id}")]
    [Permission(Permissions.Roles.Edit)]  // ← 需要 "Roles.Edit" 權限
    public async Task<IActionResult> UpdateRole(Guid id, [FromBody] UpdateRoleDto dto)
    {
        var result = await _roleService.UpdateRoleAsync(id, dto);
        return Ok(result);
    }

    [HttpDelete("{id}")]
    [Permission(Permissions.Roles.Delete)]  // ← 需要 "Roles.Delete" 權限
    public async Task<IActionResult> DeleteRole(Guid id)
    {
        var result = await _roleService.DeleteRoleAsync(id);
        return Ok(result);
    }
}
```

也可以一次要求多個權限（**OR 邏輯**，有其一即可）：

```csharp
[HttpGet("export")]
[Permission(Permissions.Books.View, Permissions.Books.Edit)]  // 有 View 或 Edit 都可以
public async Task<IActionResult> ExportBooks() { ... }
```

---

## Step 8：CurrentUser 服務

`CurrentUser` 從 `HttpContext` 的 JWT Claims 中擷取當前使用者的資訊，在整個 Request 生命週期中隨時可用：

### 介面

```csharp
// Library.Server/Services/CurrentUser.cs

public interface ICurrentUser
{
    Guid UserId { get; }
    string Account { get; }
    string Name { get; }
    string Email { get; }
    IEnumerable<string> RoleIds { get; }
    bool IsAuthenticated { get; }
    bool HasRole(string roleId);
    bool HasAnyRole(params string[] roleIds);
    bool HasAllRoles(params string[] roleIds);
}
```

### 實作

```csharp
public class CurrentUser : ICurrentUser
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUser(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    private ClaimsPrincipal? User => _httpContextAccessor.HttpContext?.User;

    public Guid UserId
    {
        get
        {
            var subClaim = User?.FindFirst(ClaimTypes.NameIdentifier)?.Value
                ?? User?.FindFirst("sub")?.Value;
            return string.IsNullOrEmpty(subClaim) ? Guid.Empty : Guid.Parse(subClaim);
        }
    }

    public string Account =>
        User?.FindFirst(ClaimTypes.Name)?.Value
        ?? User?.FindFirst("preferred_username")?.Value
        ?? string.Empty;

    public string Name =>
        User?.FindFirst(ClaimTypes.GivenName)?.Value
        ?? User?.FindFirst("name")?.Value
        ?? string.Empty;

    public string Email =>
        User?.FindFirst(ClaimTypes.Email)?.Value
        ?? User?.FindFirst("email")?.Value
        ?? string.Empty;

    public IEnumerable<string> RoleIds =>
        User?.FindAll(ClaimTypes.Role).Select(c => c.Value) ?? Enumerable.Empty<string>();

    public bool IsAuthenticated => User?.Identity?.IsAuthenticated ?? false;

    public bool HasRole(string roleId) => RoleIds.Contains(roleId);

    public bool HasAnyRole(params string[] roleIds) => roleIds.Any(HasRole);

    public bool HasAllRoles(params string[] roleIds) => roleIds.All(HasRole);
}
```

**使用情境：**
- Controller 中注入 `ICurrentUser` 即可取得當前登入者的 Id、帳號、Email 等
- 不需要每次都手動解析 `HttpContext.User.Claims`

---

## 完整元件關係圖

最後，整理所有元件之間的依賴關係：

```
┌─────────────────────────────────────────────────────────────┐
│                        Controller                           │
│                                                             │
│  [Authorize]                   ← JWT 登入驗證               │
│  [Permission("Books.View")]    ← 權限檢查                   │
│                                                             │
│  注入 ICurrentUser             ← 取得當前使用者資訊          │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┼──────────────┐
         ▼             ▼              ▼
┌────────────┐ ┌──────────────┐ ┌─────────────┐
│ Permission │ │  Permission  │ │  CurrentUser│
│ Attribute  │ │  Filter      │ │  Service    │
│            │ │              │ │             │
│ 包裝成     │ │ 呼叫         │ │ 解析 JWT   │
│ TypeFilter │ │ IAuthService │ │ Claims     │
└─────┬──────┘ └──────┬───────┘ └─────────────┘
      │               │
      │               ▼
      │        ┌──────────────────────────────┐
      │        │ PermissionAuthorizationHandler│
      │        │                              │
      │        │ 取得 UserId (from Claims)     │
      │        │ 呼叫 PermissionService        │
      │        │ 比對權限 (OR 邏輯)            │
      │        └──────────────┬───────────────┘
      │                       │
      │                       ▼
      │        ┌──────────────────────────────┐
      │        │     PermissionService        │
      │        │                              │
      │        │ DB 查詢：                     │
      │        │ UserRole → Role →            │
      │        │ RolePermission               │
      │        └──────────────┬───────────────┘
      │                       │
      ▼                       ▼
┌─────────────────────────────────────────────┐
│  PermissionRequirement                      │
│                                             │
│  攜帶需要的權限字串陣列                       │
│  e.g. ["Books.View", "Books.Create"]        │
└─────────────────────────────────────────────┘
```

---

## 總結

| 元件 | 職責 | 檔案 |
|------|------|------|
| `Permissions` | 權限常數集中定義 | `PermissionAuthorization.cs` |
| `PermissionRequirement` | 攜帶所需權限（IAuthorizationRequirement） | `PermissionRequirement.cs` |
| `PermissionAuthorizationHandler` | 核心判斷邏輯（查 DB、比對權限） | `PermissionAuthorizationHandler.cs` |
| `PermissionFilter` | 攔截 Request、呼叫 AuthorizationService | `PermissionFilter.cs` |
| `PermissionAttribute` | 語法糖，讓 Controller 可用 `[Permission(...)]` | `PermissionAttribute.cs` |
| `PermissionService` | 查詢 DB 取得使用者所有權限 | `PermissionService.cs` |
| `CurrentUser` | 從 JWT Claims 擷取當前使用者資訊 | `CurrentUser.cs` |

這套架構的好處是：
1. **零耦合**：Controller 只需要加一個 Attribute，不需要知道底層怎麼驗證
2. **動態權限**：角色的權限可以在 DB 中動態調整，不需要重新部署
3. **OR 邏輯**：可以指定多個權限，使用者有其一就放行
4. **可擴展**：如果未來需要 AND 邏輯，只要修改 Handler 中的 `foreach` 邏輯即可

---

## 參考

- [LibraryDevelop - GitHub](https://github.com/hezhengmin/LibraryDevelop)
- [ASP.NET Core 中的動態授權 - 雨夜朦朧](https://www.cnblogs.com/RainingNight/p/dynamic-authorization-in-asp-net-core.html)


