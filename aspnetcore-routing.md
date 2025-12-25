
# ASP.NET Core 路由与终结点（Minimal API）实战手册

> 适用于 .NET 6/7/8/9/10 的 Endpoint Routing & Minimal APIs。包含：路由简介、`Map`/`MapGet`/`MapPost` 等方法、`HttpContext.GetEndpoint()`、路由参数（必填/默认/可选）、路由约束（内置 & 自定义）、终结点选择顺序，以及 WebRoot 与 `UseStaticFiles`/`MapStaticAssets`。

---

## 1. 路由简介（Endpoint Routing）

ASP.NET Core 的路由负责**匹配**传入的 HTTP 请求并**分派**到可执行的终结点（Endpoint）。终结点是应用中可执行的请求处理单元，通常由 `MapGet`、`MapPost` 等注册。现代应用使用 **UseRouting**（匹配）与 **UseEndpoints**（执行）两段中间件实现端点路由；在 Minimal API 模式下，这两者通常由框架自动配置。

示例：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");
app.Run();
```
上例定义了一个 GET 终结点，当请求方法为 GET 且 URL 为 `/` 时被匹配并执行，否则返回 404。

---

## 2. 路由映射 API：`Map` / `MapGet` / `MapPost` / `MapMethods`

Minimal API 支持一组以 `Map{Verb}` 命名的扩展方法来声明路由与处理器（Route Handler），包括 `MapGet`、`MapPost`、`MapPut`、`MapDelete` 以及通用的 `MapMethods`。处理器可以是 Lambda、本地函数、实例方法或静态方法，且可异步。

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/inline", () => "GET inline");
app.MapPost("/submit", (MyDto dto) => Results.Created($"/items/{dto.Id}", dto));
app.MapMethods("/head-or-options", new[] { "HEAD", "OPTIONS" }, () => "special");

app.Run();
```
以上示例展示了多种映射方式与 `MapMethods` 的用法。

> 其他与框架集成的映射：`MapControllers`（MVC 控制器）、`MapRazorPages`、`MapHub<T>`（SignalR）、`MapGrpcService<T>` 等，均通过端点路由连接到管道。

---

## 3. 读取当前选择的终结点：`HttpContext.GetEndpoint()`

在中间件或处理器内，可通过 `HttpContext.GetEndpoint()` 取得当前请求匹配到的终结点对象及其元数据。该方法由路由中间件设置；在 `UseRouting` 之前调用会得到 `null`，在 `UseRouting` 与 `UseEndpoints` 之间可获得已选终结点。

```csharp
app.Use(async (context, next) =>
{
    var ep = context.GetEndpoint();
    Console.WriteLine($"Endpoint: {ep?.DisplayName ?? "(null)"}");
    await next(context);
});

app.UseRouting();
app.UseAuthorization();

app.MapGet("/sensitive", () => "data").WithDisplayName("Sensitive");
```
上例说明了在不同管道阶段访问终结点的差异。

---

## 4. 路由模板与参数（必填、默认、可选）

ASP.NET Core 的路由模板使用花括号声明参数，如 `/{id}`。参数类型与是否必填可在模板中定义：

- **必填参数**：`/products/{id}`。未提供将无法匹配。
- **可选参数**：在参数名后加 `?`，如 `/{id?}`；方法签名通常接收可空类型以体现可选性。
- **默认值**：在参数后通过 `=` 赋默认值，如 `/{status=all}`，未提供时取默认值。

示例：
```csharp
app.MapGet("/todo/{id}/{title?}", (int id, string? title) =>
    Results.Ok(new { id, title = title ?? "(none)" }));

app.MapGet("/todo/status/{status=all}", (string status) => Results.Ok(status));
```
以上演示了可选参数与默认值的组合用法。

> 传统 MVC 的约定路由也支持可选与默认值，例如 `{controller=Home}/{action=Index}/{id?}`。同样受端点路由的匹配规则约束。

---

## 5. 路由约束（内置）

路由约束用于在匹配阶段验证参数格式，常用约束包括：

- **类型约束**：`int`、`bool`、`datetime`、`decimal`、`double`、`float`、`guid`、`long` 等。示例：`/products/{id:int}`。
- **长度约束**：`minlength(n)`、`maxlength(n)`、`length(n)` 或 `length(min,max)`。示例：`/{slug:alpha:minlength(4):maxlength(8)}`。
- **范围约束**：`range(min,max)`；也可组合其他数值约束。示例：`/{id:int:range(1,100)}`。
- **字母约束**：`alpha` 仅匹配 A–Z/a–z。示例：`/hello/{name:alpha}`。

组合约束示例：
```csharp
app.MapGet("/product/{id:int:range(1,100)}", (int id) => Results.Ok(id));
app.MapGet("/user/{name:alpha:length(3,20)}", (string name) => Results.Ok(name));
```
以上示例展示如何在模板中链式应用多个约束。
> 约束类位于 `Microsoft.AspNetCore.Routing.Constraints` 命名空间；路由系统在匹配阶段根据这些约束筛除不合格候选。

---

## 6. 自定义路由约束（`IRouteConstraint`）

当内置约束不足以表达业务规则时，可实现 `IRouteConstraint` 自定义约束，并在应用的约束映射（ConstraintMap）中注册后使用：

```csharp
// 1) 自定义约束
public sealed class StartsWithXConstraint : IRouteConstraint
{
    public bool Match(HttpContext httpContext, IRouter route, string routeKey,
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (!values.TryGetValue(routeKey, out var v) || v is null) return false;
        var s = Convert.ToString(v);
        return !string.IsNullOrEmpty(s) && s.StartsWith("X", StringComparison.OrdinalIgnoreCase);
    }
}

// 2) 注册（Program.cs / Startup）
builder.Services.Configure<RouteOptions>(o =>
{
    o.ConstraintMap["startsWithX"] = typeof(StartsWithXConstraint);
});

// 3) 使用（模板中）
app.MapGet("/code/{id:startsWithX}", (string id) => Results.Ok(id));
```
自定义约束通过 `Match` 方法在**URL 匹配阶段**决定是否接受该参数；注册后即可按 `{param:yourConstraint}` 语法引用。

---

## 7. 终结点选择顺序（Endpoint Selection）

端点路由的 URL 匹配分多阶段执行：收集所有候选 → 按约束剔除 → 应用匹配策略 → **EndpointSelector** 最终决策。优先级由两部分决定：`RouteEndpoint.Order` 与**路由模板优先级（preference/precedence）**。模板越具体、字面文本段多的优先级越高，例如 `/hello` 比 `/{message}` 更具体；若同优先级出现多个匹配则会抛出“歧义匹配”异常。

这也解释了“路由定义顺序”与“模板优先级”的关系：常见场景无需依赖声明顺序，优先级会选出更具体的模板；仅在确需打破歧义时使用 `Order`。

---

## 8. WebRoot 与静态文件：`UseStaticFiles` & `MapStaticAssets`

- **WebRoot（默认 `wwwroot`）**：静态文件的默认根目录，可通过 `WebApplicationOptions.WebRootPath` 或宿主扩展方法更改。
- **UseStaticFiles**：将静态文件中间件加入管道，默认仅从 WebRoot 提供文件；可通过 `StaticFileOptions` 指定自定义目录（`PhysicalFileProvider`）、请求前缀（`RequestPath`）等。
- **MapStaticAssets**：新式静态资源端点（.NET 9/10 文档），提供构建期指纹、压缩与缓存头优化，相比 `UseStaticFiles` 更注重性能与可部署性。

示例：
```csharp
var builder = WebApplication.CreateBuilder(new WebApplicationOptions
{
    WebRootPath = "myroot" // 将 WebRoot 改为 myroot
});
var app = builder.Build();

// 1) 从 WebRoot（myroot）提供
app.UseStaticFiles();

// 2) 额外从项目外部目录提供 /assets 前缀下的静态文件
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(Path.Combine(builder.Environment.ContentRootPath, "external-assets")),
    RequestPath = "/assets"
});

// 3)（可选）使用 MapStaticAssets 以获得构建期优化
app.MapStaticAssets();
```
以上展示了自定义 WebRoot、从非默认目录提供静态文件，以及启用优化型静态资源端点的方式。

> **管道顺序建议**：将静态文件放在路由（UseRouting）之前，以便对静态请求快速短路；整体推荐顺序：异常处理 → HSTS/HTTPS → **静态文件** → 路由 → CORS → 认证 → 授权 → 终结点映射。

---

## 9. 实用技巧与最佳实践

- 为终结点**命名**并进行链接生成（`WithName`/`LinkGenerator`），使前后端协作更稳定。
- 将跨切面策略（如授权、CORS）通过**端点元数据**附加到具体终结点，避免散落逻辑。
- 在中间件中使用 `GetEndpoint()` 读取元数据以实现审计/策略控制。

---

### 参考资料
- Microsoft Learn：Routing in ASP.NET Core（路由与端点、选择顺序、示例）。[链接] (https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-10.0) 
- Microsoft Learn：Route handlers in Minimal API apps（`MapGet`/`MapPost` 等）。[链接] (https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/route-handlers?view=aspnetcore-10.0)  
- Microsoft Learn：`EndpointHttpContextExtensions.GetEndpoint` API 文档。[链接] (https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.endpointhttpcontextextensions.getendpoint?view=aspnetcore-9.0)  
- Microsoft Learn：`Microsoft.AspNetCore.Routing.Constraints` 命名空间（内置约束列表）。[链接] (https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.routing.constraints?view=aspnetcore-10.0)  
- Microsoft Learn：Static files in ASP.NET Core（WebRoot、`UseStaticFiles` 与 `MapStaticAssets`）。[链接] (https://learn.microsoft.com/en-us/aspnet/core/fundamentals/static-files?view=aspnetcore-10.0)  
- 进阶阅读：路由模板优先级与匹配顺序讨论（Stack Overflow）。[链接] (https://stackoverflow.com/questions/68173485/is-asp-net-core-5-0-conventional-routing-order-dependent-or-not)  
- 教程与示例：从自定义目录提供静态文件（TutorialsTeacher）。[链接] (https://www.tutorialsteacher.com/core/how-to-serve-static-files-from-another-folder-other-than-wwwroot)  
- 实践建议：中间件注册顺序最佳实践（ByteCrafted）。[链接] (https://bytecrafted.dev/posts/aspnet-core/middleware-order-best-practices/)  

