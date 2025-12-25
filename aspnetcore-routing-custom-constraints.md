
# 第 6 部分：自定义路由约束（IRouteConstraint）详解

> 本节围绕 ASP.NET Core 路由系统中的**自定义路由约束**展开，聚焦 `IRouteConstraint` 的实现、注册与使用，以及在 Minimal API、约定路由与特性路由中的综合实践与测试要点。

---

## 1. 为什么需要自定义约束？

内置约束（如 `int`、`guid`、`alpha`、`length`、`range` 等）已覆盖常见格式校验，但实际业务场景常出现更细粒度的匹配需求，例如：

- 只接受以特定前缀开头的编码（如以 `X` 或 `PRD-` 开头）。
- 限定仅匹配某一集合中的枚举值（如 `draft/published/archived`）。
- 需要针对不同环境（Dev/QA/Prod）采用不同的校验策略。

**自定义约束**通过实现 `IRouteConstraint.Match(...)` 将业务规则放在**URL 匹配阶段**提前拦截，避免进入处理器后再做无效逻辑。

---

## 2. 基本实现（IRouteConstraint）

下面以“以 `X` 开头”的字符串为例，演示完整实现：

```csharp
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.AspNetCore.Routing.Template;

public sealed class StartsWithXConstraint : IRouteConstraint
{
    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (!values.TryGetValue(routeKey, out var v) || v is null)
            return false; // 缺参 → 不匹配

        var s = Convert.ToString(v);
        if (string.IsNullOrEmpty(s)) return false;

        // 业务规则：以 X 开头（不区分大小写）
        return s.StartsWith("X", StringComparison.OrdinalIgnoreCase);
    }
}
```

> 说明：`Match` 在**路由匹配阶段**被调用；返回 `true` 表示该候选路由的该参数通过约束校验，`false` 则剔除该候选。

---

## 3. 注册到约束映射（ConstraintMap）

自定义约束类定义后，需要将其注册到框架的约束映射中，供路由模板通过简短别名引用：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<RouteOptions>(o =>
{
    // 将别名 startsWithX 映射到约束类型
    o.ConstraintMap["startsWithX"] = typeof(StartsWithXConstraint);
});

var app = builder.Build();
```

> 小贴士：约束别名应简洁且能表达含义，避免与内置约束名称冲突。

---

## 4. 在路由模板中使用（Minimal API / 约定路由 / 特性路由）

### 4.1 Minimal API
```csharp
app.MapGet("/code/{id:startsWithX}", (string id) => Results.Ok(new { id }))
   .WithName("CodeStartsWithX");
```
当请求命中 `/code/X123` 时匹配成功；`/code/A123` 则返回 404。

### 4.2 约定路由（传统 MVC）
```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id:startsWithX?}");
```
将约束应用到约定路由的 `id` 参数，可选参数加 `?`。

### 4.3 特性路由（Attribute Routing）
```csharp
[Route("api/items")]
public class ItemsController : ControllerBase
{
    [HttpGet("{code:startsWithX}")]
    public IActionResult Get(string code) => Ok(code);
}
```
在控制器/动作上以模板语法直接使用自定义约束。

---

## 5. 复合场景：与内置约束组合、默认值与可选参数

自定义约束可以与内置约束**链式组合**，也可配合**默认值**与**可选参数**实现更精细的匹配：

```csharp
// 以 X 开头 + 固定长度 8 + 仅数字
app.MapGet("/ticket/{no:startsWithX:length(8):int}", (int no) => Results.Ok(no));

// 可选参数 & 默认值：未提供时使用默认状态 all
app.MapGet("/status/{state=all}", (string state) => Results.Ok(state));

// 可选参数 + 约束：
app.MapGet("/user/{code:startsWithX?}", (string? code) => Results.Ok(code));
```

> 顺序与优先级：链式约束在同一参数上依次校验；只要有一个约束失败，则该候选路由被淘汰。

---

## 6. 测试与可维护性建议

- **单元测试**：直接实例化约束并构造 `RouteValueDictionary` 调用 `Match`；对大小写、空值、非法字符等边界进行覆盖。
- **端到端测试**：使用 `WebApplicationFactory<TEntry>` 发起真实 HTTP 请求，断言 `/code/X...` 返回 200 而 `/code/A...` 返回 404。
- **命名与文档**：在代码处添加 XML 注释，标明业务意图与示例；为团队 wiki/README 补充模板写法与使用示例。
- **性能**：约束逻辑应尽量**常数时间**判断，避免 IO；若规则复杂，考虑在处理器内部做进一步验证并返回 400，而不是 404。

---

## 7. 进阶：可配置约束与 DI

有时约束需要依赖配置或服务（如白名单集合）。可在约束构造函数注入配置或服务，但要注意：

- `IRouteConstraint` 本身不是通过 DI 解析的；若需要服务支持，可在 `Match` 中通过 `httpContext?.RequestServices` 获取所需服务：

```csharp
public sealed class WhitelistConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, string routeKey,
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (httpContext is null) return false;
        var svc = httpContext.RequestServices.GetService<IItemWhitelist>();
        if (svc is null) return false;

        var value = values.TryGetValue(routeKey, out var v) ? Convert.ToString(v) : null;
        return !string.IsNullOrEmpty(value) && svc.Allowed(value);
    }
}
```

> 注意：尽量保持约束逻辑**轻量**；复杂校验可在处理器中返回 400/422 以减少路由阶段开销。

---

## 8. 常见坑与排查

- **未注册别名**：模板写了 `{id:startsWithX}` 但未在 `RouteOptions.ConstraintMap` 注册 → 所有请求均 404。
- **空/缺省值**：`Match` 中未处理 `null` 或空字符串 → 导致异常或错误匹配。
- **歧义匹配**：多个路由在同一 URL 下均满足约束且优先级相同 → 需要调整模板具体度或显式设置 `Order`。
- **中间件顺序**：将静态文件/异常处理放在路由前，有助于性能与可观察性；授权/CORS等策略可通过端点元数据统一附加。

---

## 9. 最小可运行示例（Program.cs）

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<RouteOptions>(o =>
{
    o.ConstraintMap["startsWithX"] = typeof(StartsWithXConstraint);
});

var app = builder.Build();

app.MapGet("/code/{id:startsWithX}", (string id) => Results.Ok(new { id }))
   .WithName("CodeStartsWithX");

app.Run();

// 自定义约束类见上文 2.
```

---

### 小结
- 实现 `IRouteConstraint` 将业务规则置于**匹配阶段**提前生效。
- 通过 `ConstraintMap` 注册别名，在 Minimal API、约定或特性路由模板中即可复用。
- 与内置约束链式组合，搭配可选参数/默认值，获得精细匹配与更清晰的 404/400 语义。


