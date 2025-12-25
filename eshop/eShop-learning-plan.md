
# eShop（dotnet/eShop）系统化学习计划（.NET 9 / Aspire）

> **对象**：希望系统掌握 ASP.NET Core/.NET 的开发者与团队  
> **示例项目**：`dotnet/eShop`（官方参考应用）  
> **周期**：6–8 周（可根据时间弹性调整）

---

## 1. 学习目标
- 跑通并理解 `dotnet/eShop`（.NET 9 + Aspire）的整体工程与运行机制。
- 掌握 ASP.NET Core Web/API、EF Core、依赖注入、配置、日志等基础能力。
- 理解 Clean Architecture 与 DDD 的基本结构与实践（实体、聚合、Repository、Specification、Domain Events）。
- 理解微服务通信与容器化（HTTP/gRPC/事件总线、Docker、API Gateway、BFF）。
- 掌握 Aspire 的应用组合与可观测性，具备企业级部署与性能优化能力。

---

## 2. 学习资源（官方与精选视频）

> 说明：以下资源是实践中口碑较好的官方与社区素材，适合配合源码学习与项目实操。

### 官方资料（必读）
- **dotnet/eShop 官方说明（运行与架构概览）**：包含 .NET 9、Aspire、Dashboard、服务编排与依赖说明。
- **eShop 架构图与技术栈说明**：端到端电商场景与服务模块介绍。

### 视频教程（强烈推荐）
- **Overview of eShopOnWeb（Ardalis）**：Clean Architecture、Specification Pattern、项目结构与运行（深度导览）。
- **Tour of Microsoft Reference ASP.NET Core App eShopOnWeb**：解决方案结构、依赖关系、领域模型与聚合。
- **eShopOnWeb 14 集系列课（Wahab Hussain）**：从 MVC、EF Core 到 Basket/Order/Identity/部署的完整实战。
- **eShopOnContainers 官方微服务学习路径**：Docker、API Gateway、事件总线、消息驱动、K8s 的综合实践。

> 提示：如需我把具体链接补充到你的企业 Wiki，我可以在你确认后统一加入。

---

## 3. 学习总路线图（6–8 周）

### 第 1–2 周：ASP.NET Core 基础 & 跑通 eShop
**目标**：完成环境搭建与项目运行，熟悉基础。

**任务**：
1. 安装 .NET SDK 9+、Docker Desktop、Visual Studio 2022（17.10+）、Aspire SDK。
2. 运行 AppHost：
   ```bash
   dotnet run --project src/eShop.AppHost/eShop.AppHost.csproj
   ```
   - 观察 Aspire Dashboard（Tracing/Metrics/Logging）。
3. 学习基础：中间件（pipeline）、路由、依赖注入、配置、日志；EF Core 基础（DbContext、Migrations、NoTracking）。

---

### 第 3–4 周：架构阅读（Clean Architecture + DDD）
**目标**：能读懂并修改业务层代码，理解分层边界。

**任务**：
1. 层次结构导读：
   - **ApplicationCore**：实体、值对象、接口、领域服务。
   - **Infrastructure**：EF Core、仓储实现、外部服务。
   - **Web/WebAPI**：UI 与接口层（Controller/Minimal API）。
2. 关键模式：Repository、Specification（查询条件组合）、Domain Events（领域事件解耦）。
3. 小练习：给 Catalog 增加一个过滤与分页的 Specification，并在 Web/API 中调用。

---

### 第 5–6 周：微服务与事件驱动（eShopOnContainers）
**目标**：理解 .NET 微服务的落地方案与通信方式。

**任务**：
1. 使用 Docker Compose 启动 eShopOnContainers：
   ```bash
   docker-compose build
   docker-compose up
   ```
2. 理解模块：Catalog/Ordering/Basket/Identity 的服务边界与数据库划分。
3. 学习通信：HTTP/gRPC、API Gateway（Ocelot/YARP）、BFF 模式；事件总线（RabbitMQ/MassTransit）。
4. 小练习：在 Basket → Ordering 的流程中增加一个事件订阅与日志追踪。

---

### 第 7–8 周：进阶主题与企业实践
**目标**：具备生产级项目能力（可观测性、性能、DevOps、部署）。

**任务**：
1. **可观测性**：使用 Aspire Dashboard 观测服务指标与分布式追踪，配置健康检查与重试策略。
2. **性能优化**：引入 Redis 缓存、优化 EF Core 查询、分析慢 SQL 与热点路径。
3. **DevOps**：配置 GitHub Actions（CI/CD）、制作 Docker 镜像、推送到私有镜像库。
4. **部署**：本地 K8s（kind/minikube）或云端；根据企业要求进行 IIS/容器混合部署。
5. **AI/语义搜索（可选）**：在 Catalog 中集成向量搜索（pgvector），实现语义检索与推荐。

---

## 4. 每周学习目标（速览）

| 周次 | 主题 | 目标 |
|---|---|---|
| 1 | ASP.NET Core 基础 | 跑通示例、了解项目结构 |
| 2 | EF Core + 运行机制 | 能修改与扩展后端逻辑 |
| 3 | Clean Architecture | 理解分层与边界 |
| 4 | DDD + Specification | 能读懂业务核心代码 |
| 5 | 微服务基础 | 跑通 eShopOnContainers |
| 6 | 事件总线、BFF | 理解分布式通信 |
| 7 | 可观测性、性能优化 | 使用 Aspire Dashboard |
| 8 | 部署、DevOps、AI 搜索 | 能独立部署并扩展功能 |

---

## 5. 里程碑与产出
- ✅ 能从零搭建并运行 eShop（AppHost/Dashboard）。
- ✅ 完成 Catalog 的 Specification 改造与分页过滤。
- ✅ 在 Containers 架构中完成一次事件驱动流程（Basket → Ordering）。
- ✅ 配置并使用可观测性（Tracing/Metrics/Logs）。
- ✅ 完成一次容器化构建与部署（本地或云端）。

---

## 6. 实践建议
- 优先阅读 **ApplicationCore**（领域模型）与 **Infrastructure**（数据存取）代码，再回到 **Web** 层。
- 编写最小增量功能作为练习（如：收藏、优惠券、推荐），串联前后端与持久层。
- 在团队内进行 **Code Review** 与 **架构复盘**，输出《变更设计说明》。
- 用 **Issue + PR** 管理学习任务，记录变更与心得。

---

## 7. 后续扩展（可选）
- GraphQL（HotChocolate）、CQRS/ES、多租户（SaaS）架构、OTEL 深度观测。
- 安全与合规：认证授权（OIDC/OAuth2）、审计日志、数据脱敏与隐私保护。

---

## 8. 小结
这份计划覆盖了从基础到企业落地的完整路径。按此推进，你将具备阅读与扩展 eShop 的能力，并能把其中的架构理念迁移到公司项目中。如果需要，我可以：
- 根据你的项目栈生成 **每日任务版**；
- 输出 **源码导读（逐层/逐模块）**；
- 补充 **架构图、时序图与服务依赖图**；
- 生成 **PDF/Word/Slides**，用于培训或汇报。
