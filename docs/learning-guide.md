# eShopOnWeb 学习指南

> 一份系统化的学习路径，帮助你从零掌握这个基于 ASP.NET Core 10 Clean Architecture 的电商参考项目。

---

## 目录

1. [项目概览与学习前提](#1-项目概览与学习前提)
2. [推荐学习路线](#2-推荐学习路线)
3. [第一阶段：理解 Clean Architecture 分层](#3-第一阶段理解-clean-architecture-分层)
4. [第二阶段：领域模型与聚合根](#4-第二阶段领域模型与聚合根)
5. [第三阶段：Specification 查询模式](#5-第三阶段specification-查询模式)
6. [第四阶段：Repository 仓储模式](#6-第四阶段repository-仓储模式)
7. [第五阶段：MediatR 命令/查询模式](#7-第五阶段mediatr-命令查询模式)
8. [第六阶段：FastEndpoints API 模式](#8-第六阶段fastendpoints-api-模式)
9. [第七阶段：EF Core 数据库配置](#9-第七阶段ef-core-数据库配置)
10. [第八阶段：认证与授权](#10-第八阶段认证与授权)
11. [第九阶段：Blazor WebAssembly 管理后台](#11-第九阶段blazor-webassembly-管理后台)
12. [第十阶段：测试策略](#12-第十阶段测试策略)
13. [第十一阶段：中间件与请求管道](#13-第十一阶段中间件与请求管道)
14. [第十二阶段：依赖注入配置](#14-第十二阶段依赖注入配置)
15. [关键设计模式速查表](#15-关键设计模式速查表)
16. [动手练习建议](#16-动手练习建议)
17. [延伸学习资源](#17-延伸学习资源)

---

## 1. 项目概览与学习前提

### 项目简介

eShopOnWeb 是微软官方维护的 ASP.NET Core 参考应用，展示了如何在单体架构中正确使用 Clean Architecture（整洁架构）。它模拟了一个简单的电商网站，包含商品浏览、购物车、下单、管理后台等功能。

### 技术栈

| 层级 | 技术 |
|------|------|
| 运行时 | .NET 10 |
| Web 框架 | ASP.NET Core MVC + Razor Pages |
| API | FastEndpoints |
| 前端管理 | Blazor WebAssembly |
| ORM | Entity Framework Core |
| 认证 | ASP.NET Core Identity + JWT |
| 测试 | xUnit, NSubstitute, MSTest |
| 可观测性 | OpenTelemetry, Seq |
| 编排 | .NET Aspire |

### 学习前提

- 基本 C# 语法（泛型、异步、接口）
- 了解 HTTP 协议基础
- 了解关系数据库基本概念
- （可选）了解依赖注入基本原理

---

## 2. 推荐学习路线

```
阶段 1: Clean Architecture 分层    ← 先理解全局架构
    │
阶段 2: 领域模型与聚合根           ← 核心业务逻辑
    │
阶段 3: Specification 查询模式     ← 理解查询封装
    │
阶段 4: Repository 仓储模式        ← 数据访问抽象
    │
阶段 5: MediatR CQRS 模式         ← Web 层请求处理
    │
阶段 6: FastEndpoints API          ← REST API 设计
    │
阶段 7: EF Core 配置               ← 数据库映射细节
    │
阶段 8: 认证与授权                 ← 安全机制
    │
阶段 9: Blazor 管理后台            ← 前端 SPA
    │
阶段 10: 测试策略                  ← 质量保障
    │
阶段 11: 中间件与管道              ← 请求生命周期
    │
阶段 12: 依赖注入配置              ← 组合全局
```

**建议：** 每个阶段先阅读对应的源文件，再对照本文档的讲解，最后做动手练习。

---

## 3. 第一阶段：理解 Clean Architecture 分层

### 核心原则

依赖方向严格向内：外层依赖内层，内层不知道外层的存在。

```
┌──────────────────────────────────────────┐
│  Presentation (Web / PublicApi / Blazor)  │  ← 最外层
│  ┌────────────────────────────────────┐  │
│  │       Infrastructure              │  │  ← 数据访问实现
│  │  ┌──────────────────────────────┐  │  │
│  │  │     ApplicationCore          │  │  │  ← 最内层，纯业务逻辑
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/ApplicationCore/ApplicationCore.csproj` | 注意：没有任何对 Infrastructure 或 Web 的引用 |
| `src/Infrastructure/Infrastructure.csproj` | 仅引用 ApplicationCore |
| `src/Web/Web.csproj` | 引用 Infrastructure 和 ApplicationCore |
| `src/PublicApi/PublicApi.csproj` | 引用 Infrastructure 和 ApplicationCore |

### 学习要点

- **ApplicationCore**（核心层）：包含实体、接口、规约、领域服务。零框架依赖，仅使用 Ardalis 的轻量库。
- **Infrastructure**（基础设施层）：包含 EF Core 的 DbContext、Repository 实现、Identity 配置。
- **Web/PublicApi**（表示层）：包含控制器、页面、API 端点、视图模型。

### 关键认知

> ApplicationCore 项目中定义的是接口（如 `IRepository<T>`），而具体实现（如 `EfRepository<T>`）在 Infrastructure 中。这就是依赖倒置原则（DIP）的体现。

---

## 4. 第二阶段：领域模型与聚合根

### 核心概念

**聚合根（Aggregate Root）** 是 DDD（领域驱动设计）中的核心概念：一组相关对象的入口点，外部只能通过聚合根来修改其内部状态。

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/ApplicationCore/Entities/BaseEntity.cs` | 所有实体的基类，提供 `Id` 属性 |
| `src/ApplicationCore/Interfaces/IAggregateRoot.cs` | 标记接口，标识哪些实体是聚合根 |
| `src/ApplicationCore/Entities/BasketAggregate/Basket.cs` | **重点**：最典型的聚合根实现 |
| `src/ApplicationCore/Entities/BasketAggregate/BasketItem.cs` | Basket 的子实体 |
| `src/ApplicationCore/Entities/OrderAggregate/Order.cs` | Order 聚合根 |
| `src/ApplicationCore/Entities/OrderAggregate/OrderItem.cs` | Order 的子实体 |
| `src/ApplicationCore/Entities/OrderAggregate/Address.cs` | 值对象（Value Object） |
| `src/ApplicationCore/Entities/CatalogItem.cs` | 简单聚合根 |

### 学习要点：以 Basket 为例

```csharp
// Basket.cs — 关键设计模式标注
public class Basket : BaseEntity, IAggregateRoot  // ① 实现聚合根标记
{
    public string BuyerId { get; private set; }   // ② setter 为 private

    private readonly List<BasketItem> _items = new();              // ③ 私有字段
    public IReadOnlyCollection<BasketItem> Items => _items.AsReadOnly(); // ④ 只读暴露

    public void AddItem(int catalogItemId, decimal unitPrice, int quantity = 1)
    {
        // ⑤ 业务规则在聚合根内部强制执行
        if (!Items.Any(i => i.CatalogItemId == catalogItemId))
            _items.Add(new BasketItem(catalogItemId, quantity, unitPrice));
        else
            Items.First(i => i.CatalogItemId == catalogItemId).AddQuantity(quantity);
    }
}
```

**关键设计决策：**

1. `IAggregateRoot` 是空接口，仅用于类型约束（只有聚合根才能作为 Repository 的泛型参数）
2. 集合使用 `private readonly List` + `IReadOnlyCollection` 暴露 —— 外部无法直接增删元素
3. 状态修改必须通过行为方法（`AddItem`、`RemoveEmptyItems`）—— 确保业务规则被遵守
4. `Address` 是值对象，没有自己的 `Id`，通过 EF Core Owned Type 映射到 Order 表中

### 领域事件

```csharp
// src/ApplicationCore/Entities/OrderAggregate/Events/OrderCreatedEvent.cs
public class OrderCreatedEvent(Order order) : DomainEventBase
{
    public Order Order { get; init; } = order;
}

// src/ApplicationCore/Entities/OrderAggregate/Handlers/OrderCreatedHandler.cs
public class OrderCreatedHandler : INotificationHandler<OrderCreatedEvent>
{
    public async Task Handle(OrderCreatedEvent domainEvent, CancellationToken ct)
    {
        // 发送邮件通知 —— 解耦了订单创建和邮件发送
    }
}
```

---

## 5. 第三阶段：Specification 查询模式

### 核心概念

Specification（规约）模式将查询逻辑封装成可复用的对象，避免在 Repository 或 Service 中散布 LINQ 查询。

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/ApplicationCore/Specifications/BasketWithItemsSpecification.cs` | 最简单的 Spec，理解基本结构 |
| `src/ApplicationCore/Specifications/CatalogFilterPaginatedSpecification.cs` | 带过滤 + 分页 |
| `src/ApplicationCore/Specifications/CustomerOrdersWithItemsSpecification.cs` | 嵌套 Include（N+1 问题解决） |
| `src/ApplicationCore/Specifications/CatalogItemNameSpecification.cs` | 按名称搜索 |

### 学习要点

**从简单到复杂：**

```csharp
// 1. 最简单：单条件 + Include
public sealed class BasketWithItemsSpecification : Specification<Basket>
{
    public BasketWithItemsSpecification(string buyerId)
    {
        Query.Where(b => b.BuyerId == buyerId)
             .Include(b => b.Items);  // 预加载关联数据
    }
}

// 2. 中等复杂：可选过滤 + 分页
public class CatalogFilterPaginatedSpecification : Specification<CatalogItem>
{
    public CatalogFilterPaginatedSpecification(int skip, int take, int? brandId, int? typeId)
    {
        if (take == 0) take = int.MaxValue;
        Query
            .Where(i => (!brandId.HasValue || i.CatalogBrandId == brandId) &&
                        (!typeId.HasValue || i.CatalogTypeId == typeId))
            .Skip(skip).Take(take);
    }
}

// 3. 较复杂：嵌套 Include（解决 N+1 查询问题）
public class CustomerOrdersWithItemsSpecification : Specification<Order>
{
    public CustomerOrdersWithItemsSpecification(string buyerId)
    {
        Query.Where(o => o.BuyerId == buyerId)
             .Include(o => o.OrderItems)
                 .ThenInclude(i => i.ItemOrdered);  // 二级关联
    }
}
```

**使用方式：**

```csharp
var spec = new BasketWithItemsSpecification(buyerId);
var basket = await _repository.FirstOrDefaultAsync(spec);
```

### 全部规约清单

| 规约 | 用途 |
|------|------|
| `BasketWithItemsSpecification` | 按 ID 或 buyerId 查询购物车及其商品 |
| `CatalogFilterPaginatedSpecification` | 按品牌/类型过滤 + 分页 |
| `CatalogFilterSpecification` | 简单分类过滤 |
| `CatalogItemNameSpecification` | 按名称搜索商品 |
| `CatalogItemsSpecification` | 查询所有商品 |
| `CustomerOrdersSpecification` | 查询用户订单（不含详情） |
| `CustomerOrdersWithItemsSpecification` | 查询用户订单 + 商品详情 |
| `OrderWithItemsByIdSpec` | 按 ID 查询单个订单 + 商品 |

---

## 6. 第四阶段：Repository 仓储模式

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/ApplicationCore/Interfaces/IRepository.cs` | 接口定义 —— 注意泛型约束 |
| `src/ApplicationCore/Interfaces/IReadRepository.cs` | 只读仓储接口 |
| `src/Infrastructure/Data/EfRepository.cs` | 实现 —— 极简，利用基类 |

### 学习要点

```csharp
// 接口定义（ApplicationCore 层 —— 不知道 EF Core 的存在）
public interface IRepository<T> : IRepositoryBase<T> where T : class, IAggregateRoot { }
public interface IReadRepository<T> : IReadRepositoryBase<T> where T : class, IAggregateRoot { }

// 实现（Infrastructure 层）
public class EfRepository<T> : RepositoryBase<T>, IReadRepository<T>, IRepository<T>
    where T : class, IAggregateRoot
{
    public EfRepository(CatalogContext dbContext) : base(dbContext) { }
}
```

**关键认知：**

1. 泛型约束 `where T : class, IAggregateRoot` —— **只有聚合根才能拥有 Repository**，这强制了 DDD 的边界
2. 实现类几乎为空，因为 `Ardalis.Specification.EntityFrameworkCore` 的 `RepositoryBase<T>` 提供了所有 CRUD 方法
3. `IReadRepository` 和 `IRepository` 分离 —— 遵循接口隔离原则（ISP），查询操作不需要写入能力

---

## 7. 第五阶段：MediatR 命令/查询模式

### 核心概念

MediatR 实现了中介者模式，将请求的发送方和处理方解耦。在本项目中，Web 层使用 MediatR 来组织 Feature（功能模块）。

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/Web/Features/MyOrders/GetMyOrders.cs` | Query（查询请求）定义 |
| `src/Web/Features/MyOrders/GetMyOrdersHandler.cs` | Handler（处理器）实现 |
| `src/Web/Features/OrderDetails/GetOrderDetails.cs` | 另一个 Query 示例 |
| `src/Web/Features/OrderDetails/GetOrderDetailsHandler.cs` | 带嵌套数据加载的 Handler |
| `src/Web/Configuration/ConfigureWebServices.cs` | MediatR 注册配置 |

### 学习要点

```csharp
// 1. 定义 Query（输入 + 输出类型）
public class GetMyOrders : IRequest<IEnumerable<OrderViewModel>>
{
    public string UserName { get; set; }
}

// 2. 实现 Handler
public class GetMyOrdersHandler : IRequestHandler<GetMyOrders, IEnumerable<OrderViewModel>>
{
    private readonly IReadRepository<Order> _orderRepository;

    public async Task<IEnumerable<OrderViewModel>> Handle(
        GetMyOrders request, CancellationToken cancellationToken)
    {
        var specification = new CustomerOrdersSpecification(request.UserName);
        var orders = await _orderRepository.ListAsync(specification, cancellationToken);
        return orders.Select(o => new OrderViewModel { /* 映射 */ });
    }
}

// 3. 在 Controller/Page 中使用
var orders = await _mediator.Send(new GetMyOrders(username));
```

**组织方式：** 每个功能一个文件夹，Query/Command 和 Handler 放在一起 —— 按功能组织而非按技术层组织。

---

## 8. 第六阶段：FastEndpoints API 模式

### 核心概念

FastEndpoints 是一个高性能的 API 框架，每个端点是一个独立的类，替代传统的 Controller + Action 方式。

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/PublicApi/AuthEndpoints/AuthenticateEndpoint.cs` | 认证端点 —— AllowAnonymous |
| `src/PublicApi/CatalogItemEndpoints/CatalogItemGetByIdEndpoint.cs` | GET —— 带类型安全返回值 |
| `src/PublicApi/CatalogItemEndpoints/CatalogItemListPagedEndpoint.cs` | 分页查询 |
| `src/PublicApi/CatalogItemEndpoints/CreateCatalogItemEndpoint.cs` | POST + 文件上传 |
| `src/PublicApi/CatalogItemEndpoints/UpdateCatalogItemEndpoint.cs` | PUT 更新 |
| `src/PublicApi/CatalogItemEndpoints/DeleteCatalogItemEndpoint.cs` | DELETE |
| `src/PublicApi/Middleware/ExceptionMiddleware.cs` | 统一异常处理 |

### 学习要点

```csharp
// FastEndpoint 的标准结构
public class CatalogItemGetByIdEndpoint(
    IRepository<CatalogItem> itemRepository,   // ① 构造器注入
    IUriComposer uriComposer)
    : Endpoint<GetByIdCatalogItemRequest,      // ② 请求类型
               Results<Ok<GetByIdCatalogItemResponse>, NotFound>>  // ③ 强类型响应
{
    public override void Configure()
    {
        Get("api/catalog-items/{catalogItemId}"); // ④ 路由配置
        AllowAnonymous();                         // ⑤ 授权配置
    }

    public override async Task<Results<Ok<GetByIdCatalogItemResponse>, NotFound>>
        ExecuteAsync(GetByIdCatalogItemRequest request, CancellationToken ct)
    {
        var item = await itemRepository.GetByIdAsync(request.CatalogItemId, ct);
        if (item is null) return TypedResults.NotFound();    // ⑥ 类型安全返回
        return TypedResults.Ok(new GetByIdCatalogItemResponse { /* ... */ });
    }
}
```

**与 Controller 对比：**

| 传统 Controller | FastEndpoints |
|----------------|---------------|
| 一个类多个 Action | 一个类一个端点 |
| `[HttpGet]` 属性 | `Configure()` 方法 |
| `ActionResult<T>` | `Results<Ok<T>, NotFound>` |
| 需要手动绑定模型 | 自动绑定到 Request 类型 |

### 端点组织

```
PublicApi/
├── AuthEndpoints/          ← 认证相关
├── CatalogItemEndpoints/   ← 商品 CRUD
├── CatalogBrandEndpoints/  ← 品牌查询
├── RoleManagementEndpoints/
├── RoleMembershipEndpoints/
└── UserManagementEndpoints/
```

---

## 9. 第七阶段：EF Core 数据库配置

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/Infrastructure/Data/CatalogContext.cs` | 主 DbContext —— 注意 `ApplyConfigurationsFromAssembly` |
| `src/Infrastructure/Identity/AppIdentityDbContext.cs` | Identity 专用 DbContext |
| `src/Infrastructure/Data/Config/BasketConfiguration.cs` | 私有集合的 Fluent API 配置 |
| `src/Infrastructure/Data/Config/OrderConfiguration.cs` | Owned Type（值对象）映射 |
| `src/Infrastructure/Data/Config/CatalogItemConfiguration.cs` | 基本字段映射 |
| `src/Infrastructure/Data/CatalogContextSeed.cs` | 种子数据 |
| `src/Infrastructure/Dependencies.cs` | DbContext 注册 |

### 学习要点

**1. 两个独立的 DbContext**

这是一个重要设计：CatalogContext 管理业务数据，AppIdentityDbContext 管理用户身份。它们可以指向不同的数据库。

**2. 私有集合映射**

```csharp
// BasketConfiguration.cs
public void Configure(EntityTypeBuilder<Basket> builder)
{
    var navigation = builder.Metadata.FindNavigation(nameof(Basket.Items));
    navigation?.SetPropertyAccessMode(PropertyAccessMode.Field);  // 直接访问私有字段 _items
}
```

这行配置让 EF Core 绕过公开的 `IReadOnlyCollection` 属性，直接操作私有的 `_items` 字段 —— 实现了聚合根的封装性。

**3. Owned Type（值对象映射）**

```csharp
// OrderConfiguration.cs
builder.OwnsOne(o => o.ShipToAddress, a =>
{
    a.WithOwner();
    a.Property(a => a.ZipCode).HasMaxLength(18).IsRequired();
    a.Property(a => a.Street).HasMaxLength(180).IsRequired();
});
```

`Address` 没有自己的数据库表，其字段嵌入到 `Orders` 表中。

**4. 内存数据库（开发/测试用）**

```json
// appsettings.json
{ "UseOnlyInMemoryDatabase": true }
```

设置此项后，无需 SQL Server 即可运行应用。

---

## 10. 第八阶段：认证与授权

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/Web/Program.cs` | Cookie 认证 + Identity 配置 |
| `src/PublicApi/Extensions/ServiceCollectionExtensions.cs` | JWT Bearer 认证 |
| `src/Infrastructure/Identity/IdentityTokenClaimService.cs` | Token 生成逻辑 |
| `src/Infrastructure/Identity/AppIdentityDbContext.cs` | Identity 数据存储 |
| `src/PublicApi/AuthEndpoints/AuthenticateEndpoint.cs` | 登录 API |

### 认证方式对比

| 场景 | 方式 | 适用于 |
|------|------|--------|
| 浏览器访问 Web | Cookie 认证 | MVC / Razor Pages |
| API 调用 | JWT Bearer Token | PublicApi / Blazor Admin |
| 第三方登录 | OAuth (GitHub) | Web（可选） |

### JWT Token 生成流程

```
用户 → POST /api/authenticate → 验证密码 → 生成 JWT → 返回 Token
                                    ↓
                        IdentityTokenClaimService
                        - 从 UserManager 获取用户角色
                        - 构建 Claims（用户名 + 角色）
                        - HS256 签名
                        - 7 天过期
```

### 默认测试账号

| 用户 | 邮箱 | 角色 |
|------|------|------|
| 普通用户 | `demouser@microsoft.com` | User |
| 管理员 | `admin@microsoft.com` | Admin |

---

## 11. 第九阶段：Blazor WebAssembly 管理后台

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/BlazorAdmin/Services/HttpService.cs` | 通用 HTTP 客户端封装 |
| `src/BlazorAdmin/Services/CatalogItemService.cs` | 具体业务服务 |
| `src/BlazorAdmin/Services/CachedCatalogItemServiceDecorator.cs` | 装饰器模式缓存 |
| `src/BlazorShared/BaseUrlConfiguration.cs` | API 地址配置 |
| `src/BlazorAdmin/Pages/CatalogItemPage/` | Razor 组件页面 |

### 架构关系

```
BlazorAdmin (浏览器中运行)
    ↓ HTTP 请求
PublicApi (FastEndpoints)
    ↓ 调用 Repository
Infrastructure (EF Core)
    ↓ SQL
Database
```

### 装饰器模式亮点

```csharp
// 不修改原有服务，通过包装增加缓存能力
public class CachedCatalogItemServiceDecorator : ICatalogItemService
{
    private readonly ICatalogItemService _innerService;  // 包装原始服务
    private readonly IMemoryCache _cache;

    public async Task<EditCatalogItemResult> GetByIdAsync(int id)
    {
        var cacheKey = $"CatalogItem_{id}";
        if (_cache.TryGetValue(cacheKey, out EditCatalogItemResult? cached))
            return cached;              // 命中缓存
        var result = await _innerService.GetByIdAsync(id);  // 未命中，调用原始服务
        _cache.Set(cacheKey, result);
        return result;
    }
}
```

---

## 12. 第十阶段：测试策略

### 四层测试体系

| 测试类型 | 项目 | 依赖 | 速度 |
|----------|------|------|------|
| 单元测试 | `tests/UnitTests/` | NSubstitute 模拟 | 极快 |
| 集成测试 | `tests/IntegrationTests/` | 内存数据库 | 快 |
| 功能测试 | `tests/FunctionalTests/` | Mvc.Testing 完整管道 | 中等 |
| API 集成测试 | `tests/PublicApiIntegrationTests/` | API 管道 | 中等 |

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `tests/UnitTests/ApplicationCore/Entities/BasketTests/BasketAddItem.cs` | 最简单的领域测试 |
| `tests/UnitTests/ApplicationCore/Services/BasketServiceTests/AddItemToBasket.cs` | Service 层测试 + NSubstitute |
| `tests/UnitTests/Builders/` | Test Builder 模式 |
| `tests/IntegrationTests/Repositories/` | 内存数据库集成测试 |
| `tests/FunctionalTests/` | 完整 HTTP 管道测试 |

### 测试示例

```csharp
// 单元测试：纯领域逻辑
[Fact]
public void AddsItemToEmptyBasket()
{
    var basket = new Basket("buyer1");
    basket.AddItem(1, 10m, 2);
    Assert.Single(basket.Items);
    Assert.Equal(2, basket.Items.First().Quantity);
}

// 服务测试：使用 NSubstitute 模拟依赖
[Fact]
public async Task AddsItemToBasketWhenBasketDoesNotExist()
{
    var basketRepo = Substitute.For<IRepository<Basket>>();
    basketRepo.FirstOrDefaultAsync(Arg.Any<Specification<Basket>>())
        .Returns((Basket)null);

    var service = new BasketService(basketRepo, null);
    await service.AddItemToBasket("user1", 1, 10m, 1);

    await basketRepo.Received(1).AddAsync(Arg.Any<Basket>());
}
```

### 运行测试

```bash
# 全部测试
dotnet test ./eShopOnWeb.sln --collect:"XPlat Code Coverage"

# 单个项目
dotnet test tests/UnitTests/UnitTests.csproj

# 按名称过滤
dotnet test --filter "FullyQualifiedName~BasketAddItem"
```

---

## 13. 第十一阶段：中间件与请求管道

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/Web/Program.cs`（第 92-131 行） | Web 管道顺序 |
| `src/PublicApi/Program.cs`（第 62 行起） | API 管道顺序 |
| `src/Web/UserContextEnrichmentMiddleware.cs` | 自定义中间件示例 |
| `src/PublicApi/Middleware/ExceptionMiddleware.cs` | 全局异常处理 |

### Web 请求管道顺序

```
请求 →
  1. SeedDatabase（首次运行时初始化数据）
  2. PathBase（反向代理路径前缀）
  3. HealthChecks（健康检查端点）
  4. TroubleshootingMiddlewares（调试中间件）
  5. HttpsRedirection（HTTP → HTTPS）
  6. BlazorFrameworkFiles + StaticFiles（静态资源）
  7. Routing（路由匹配）
  8. CookiePolicy（Cookie 策略）
  9. Authentication（身份验证）
  10. Authorization（权限检查）
  11. UserContextEnrichmentMiddleware（用户信息注入日志/追踪）
  12. MapControllerRoute / MapRazorPages（端点执行）
→ 响应
```

### 自定义中间件详解

```csharp
// UserContextEnrichmentMiddleware.cs —— 将用户 ID 注入可观测性系统
public async Task InvokeAsync(HttpContext context)
{
    string? userId = context.User?.FindFirstValue(ClaimTypes.NameIdentifier);
    if (userId is not null)
    {
        Activity.Current?.SetTag("user.id", userId);  // OpenTelemetry 追踪
        using (logger.BeginScope(new Dictionary<string, object> { ["UserId"] = userId }))
        {
            await next(context);  // 后续中间件都能在日志中看到 UserId
        }
    }
    else { await next(context); }
}
```

---

## 14. 第十二阶段：依赖注入配置

### 必读文件

| 文件路径 | 学习重点 |
|----------|---------|
| `src/Web/Configuration/ConfigureCoreServices.cs` | 核心服务注册 |
| `src/Web/Configuration/ConfigureWebServices.cs` | MediatR 注册 |
| `src/Web/Configuration/ConfigureCookieSettings.cs` | 认证设置 |
| `src/Web/Extensions/ServiceCollectionExtensions.cs` | Blazor + 健康检查 |
| `src/PublicApi/Extensions/ServiceCollectionExtensions.cs` | JWT + CORS + Swagger |
| `src/Infrastructure/Dependencies.cs` | 数据库上下文注册 |

### 生命周期选择

```csharp
// Scoped（每次 HTTP 请求一个实例）—— Repository 和 DbContext
services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
services.AddScoped(typeof(IReadRepository<>), typeof(EfRepository<>));
services.AddScoped<IBasketService, BasketService>();

// Singleton（应用生命周期一个实例）—— 配置对象
services.AddSingleton<IUriComposer>(new UriComposer(catalogSettings));

// 开放泛型注册 —— 一行注册所有 Repository<T>
services.AddScoped(typeof(IAppLogger<>), typeof(LoggerAdapter<>));
```

### DI 组合顺序（Web 项目）

```
Program.cs
  ├── Dependencies.ConfigureServices()        → 数据库上下文
  ├── ConfigureCoreServices.AddCoreServices() → 仓储、领域服务
  ├── ConfigureWebServices.AddWebServices()   → MediatR、视图模型服务
  ├── ConfigureCookieSettings.ConfigureCookieSettings() → 认证
  └── ServiceCollectionExtensions              → Blazor、健康检查
```

---

## 15. 关键设计模式速查表

| 模式 | 用途 | 关键文件 |
|------|------|---------|
| **聚合根** | 领域边界保护 | `Basket.cs`, `Order.cs` |
| **Repository** | 数据访问抽象 | `IRepository.cs` → `EfRepository.cs` |
| **Specification** | 查询逻辑封装 | `src/ApplicationCore/Specifications/` |
| **MediatR** | 请求/处理解耦 | `src/Web/Features/` |
| **FastEndpoints** | API 端点组织 | `src/PublicApi/*Endpoints/` |
| **值对象 (Owned Type)** | 无 ID 的领域概念 | `Address.cs` → `OrderConfiguration.cs` |
| **装饰器** | 透明增加行为（缓存） | `CachedCatalogItemServiceDecorator.cs` |
| **领域事件** | 解耦跨关注点逻辑 | `OrderCreatedEvent.cs` → `OrderCreatedHandler.cs` |
| **Guard Clauses** | 防御性编程 | `CatalogItem.UpdateDetails()` |
| **Result Pattern** | 替代异常控制流 | `BasketService` 返回 `Result<T>` |

---

## 16. 动手练习建议

### 入门级

1. **追踪一次购物流程**：从浏览商品 → 添加到购物车 → 下单，在代码中找到每一步经过的 Controller/Page → Service → Repository → Database 的完整路径。

2. **添加一个新的 Specification**：创建一个 `CatalogItemsByPriceRangeSpecification`，支持按价格区间过滤商品。

3. **阅读并运行所有单元测试**：理解每个测试在验证什么业务规则。

### 中级

4. **添加新的聚合根**：例如 `WishList`（愿望清单），包含用户 ID 和商品列表。实现实体、Repository 接口、EF 配置。

5. **添加新的 FastEndpoint**：为 WishList 创建 CRUD API 端点。

6. **添加新的 MediatR Feature**：在 Web 项目中添加 `GetMyWishList` 查询。

### 高级

7. **实现领域事件**：当商品价格变更时，发布事件通知所有将该商品加入购物车的用户。

8. **添加新的测试层**：为新功能编写单元测试、集成测试、功能测试。

9. **对比微服务版本**：微软还有 `eShopOnContainers`（微服务版本），对比两者的架构选择和权衡。

---

## 17. 延伸学习资源

### 书籍

- **《Clean Architecture》** — Robert C. Martin（理解分层架构思想）
- **《Domain-Driven Design》** — Eric Evans（聚合根、值对象、领域事件的理论基础）
- **《Implementing Domain-Driven Design》** — Vaughn Vernon（DDD 实践）

### 在线资源

- [Microsoft eShopOnWeb Wiki](https://github.com/dotnet-architecture/eShopOnWeb/wiki) — 官方 Wiki
- [Ardalis Clean Architecture Template](https://github.com/ardalis/CleanArchitecture) — 本项目使用的架构模板
- [FastEndpoints 文档](https://fast-endpoints.com/) — API 框架文档
- [Ardalis Specification 文档](https://specification.ardalis.com/) — 规约模式库文档

### 代码阅读顺序建议

如果你时间有限，优先阅读以下 10 个文件（按顺序）：

1. `src/ApplicationCore/Entities/BasketAggregate/Basket.cs`
2. `src/ApplicationCore/Specifications/BasketWithItemsSpecification.cs`
3. `src/ApplicationCore/Interfaces/IRepository.cs`
4. `src/Infrastructure/Data/EfRepository.cs`
5. `src/ApplicationCore/Services/BasketService.cs`
6. `src/Web/Features/MyOrders/GetMyOrdersHandler.cs`
7. `src/PublicApi/CatalogItemEndpoints/CatalogItemGetByIdEndpoint.cs`
8. `src/Infrastructure/Data/Config/OrderConfiguration.cs`
9. `src/Infrastructure/Identity/IdentityTokenClaimService.cs`
10. `tests/UnitTests/ApplicationCore/Entities/BasketTests/BasketAddItem.cs`

---

> **提示：** 学习架构不要急于求成。理解"为什么这样设计"比"怎么写代码"更重要。每看一个模式，问自己：如果不用这个模式，会有什么问题？
