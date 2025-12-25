
# 企业级权限系统（用户–角色–权限）设计文档（完整版本）

## 1. 系统设计目标
- 统一用户、角色、权限管理
- RBAC+ABAC 灵活扩展
- 支持菜单、按钮、API 细粒度权限
- 支持多租户（可选）
- 可审计、可扩展、可持续维护

## 2. 权限模型概述
User、Role、Permission 三层结构，并通过映射表关联：
```
User <-- UserRole --> Role <-- RolePermission --> Permission
```

## 3. 数据模型设计
### 3.1 Users（用户表）
- Id, UserName, DisplayName, PasswordHash, Department, Position, IsExternal, IsActive 等

### 3.2 Roles（角色表）
- Id, RoleName, Description, IsSystem

### 3.3 Permissions（权限表）
包含菜单、页面、按钮、API 资源，并支持树状结构
- Id, PermissionName, Type, ParentId, Url, HttpMethod

### 3.4 UserRoles
- UserId, RoleId

### 3.5 RolePermissions
- RoleId, PermissionId

### 3.6 LoginHistory
记录用户登录日志

## 4. 核心流程
### 4.1 登录流程
用户登录 → 验证密码 → 加载角色 → 加载权限 → Claims/JWT → 返回页面

### 4.2 权限加载流程
User → Roles → RolePermissions → Permissions

### 4.3 权限校验
前端：菜单/按钮渲染；后端 API 校验

## 5. 权限体系结构图（ASCII）
```
用户系统 → 身份认证模块 → 权限加载 → API 权限验证 → 前端权限控制
```

## 6. 权限粒度
- 菜单权限
- 页面权限
- 按钮权限
- API 权限

## 7. 多租户扩展（可选）
所有表加入 TenantId

## 8. 企业常见场景
- 部门权限
- 外包人员权限隔离
- ABAC 动态权限

## 9. 管理后台功能
- 用户管理
- 角色管理
- 权限资源管理
- 审计日志

## 10. 最佳实践
- Token 不放大量权限
- 菜单与 API 权限严格分离
- 角色数量控制
- 权限语义命名如：User.Create、Order.Approve

