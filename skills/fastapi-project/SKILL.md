---
name: fastapi-project
description: FastAPI 全栈项目最佳实践 - 完整的技术栈参考，按需引用各子模块
---

# FastAPI Project Best Practices

## 概述

本模块提供 FastAPI 全栈项目的完整技术参考，包含数据库、架构、认证、测试、配置等核心主题。根据需要引用对应文件。

## 快速导航

| 需要什么？ | 引用文件 | 优先级 |
|-----------|---------|--------|
| 项目骨架 | [SKIP 直接阅读目录结构](#目录结构) | - |
| 数据库配置 | [database.md](./database.md) | 必读 |
| 数据库迁移 | [migrations.md](./migrations.md) | 必读 |
| 分层架构 | [architecture.md](./architecture.md) | 必读 |
| 依赖注入 | [DI.md](./DI.md) | 必读 |
| 认证鉴权 | [auth.md](./auth.md) | 必读 |
| 异常处理 | [errors.md](./errors.md) | 必读 |
| 测试策略 | [testing.md](./testing.md) | 推荐 |
| 配置管理 | [config.md](./config.md) | 推荐 |
| 中间件 | [middleware.md](./middleware.md) | 推荐 |
| 缓存 | [cache.md](./cache.md) | 可选 |
| WebSocket | [websocket.md](./websocket.md) | 可选 |

## 目录结构

```
app/
├── main.py                 # 应用入口
├── api/                   # API 路由（Router 层）
│   ├── deps.py            # 公共依赖
│   └── endpoints/
│       ├── users.py
│       └── items.py
├── services/              # 业务逻辑层
│   └── user_service.py
├── repositories/           # 数据访问层
│   └── user_repo.py
├── models/                # SQLAlchemy 模型
│   ├── base.py
│   └── user.py
├── schemas/               # Pydantic 模型
│   ├── user.py
│   └── item.py
└── core/                  # 核心配置
    ├── config.py          # → config.md
    ├── database.py        # → database.md
    ├── security.py        # → auth.md
    └── exceptions.py     # → errors.md
```

## 快速开始

### 1. 新建项目

```bash
# 创建项目
uv init fastapi-project
cd fastapi-project

# 安装核心依赖
uv add fastapi uvicorn sqlalchemy pydantic-settings python-jose passlib[bcrypt]
uv add -D pytest pytest-asyncio httpx

# 安装数据库驱动
uv add asyncpg aiosqlite  # PostgreSQL + SQLite
uv add alembic
```

### 2. 配置数据库

参考 [database.md](./database.md) 设置异步数据库连接。

### 3. 实现分层架构

参考 [architecture.md](./architecture.md) 构建 Router → Service → Repository 分层。

### 4. 添加认证

参考 [auth.md](./auth.md) 实现 JWT 认证。

### 5. 测试

参考 [testing.md](./testing.md) 编写单元测试和集成测试。

---

## 模块说明

### 核心模块（必读）

- **[DI.md](./DI.md)** - FastAPI 依赖注入系统，Depends 正确用法
- **[config.md](./config.md)** - 项目配置管理，环境变量、多环境
- **[database.md](./database.md)** - SQLAlchemy 2.0 异步数据库配置
- **[migrations.md](./migrations.md)** - Alembic 异步迁移配置
- **[architecture.md](./architecture.md)** - 分层架构 + Repository 模式
- **[auth.md](./auth.md)** - JWT 认证，实现登录和 Token 验证
- **[errors.md](./errors.md)** - 统一异常处理，BusinessException

### 增强模块（推荐）

- **[testing.md](./testing.md)** - 测试策略，单元测试 + 集成测试
- **[middleware.md](./middleware.md)** - CORS、限流、日志、健康检查
- **[cache.md](./cache.md)** - Redis 缓存装饰器、分布式锁
- **[websocket.md](./websocket.md)** - WebSocket 实时通信

---

## 使用方式

### 按需引用

当你需要某个功能时，直接阅读对应文件：

```
需要缓存？ → 阅读 cache.md
需要 WebSocket？ → 阅读 websocket.md
```

### 完整学习

如果你是 FastAPI 新手，建议按顺序学习：

1. [config.md](./config.md) - 了解配置管理
2. [database.md](./database.md) - 设置数据库
3. [DI.md](./DI.md) - 理解依赖注入
4. [architecture.md](./architecture.md) - 掌握分层架构
5. [auth.md](./auth.md) - 添加认证
6. [errors.md](./errors.md) - 处理异常
7. [testing.md](./testing.md) - 编写测试

---

## 依赖关系

```
                    ┌─────────────┐
                    │  config.md  │
                    └──────┬──────┘
                           │
              ┌─────────────┼─────────────┐
              │            │            │
              ▼            ▼            ▼
       ┌──────────┐  ┌──────────┐ ┌──────────┐
       │DI.md     │  │auth.md   │ │errors.md │
       └────┬─────┘  └────┬─────┘ └────┬─────┘
            │            │            │
            └────────────┼────────────┘
                         │
              ┌──────────┴──────────┐
              │  architecture.md    │
              │  (layered + repo)   │
              └──────────┬──────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │database  │   │migrations│   │testing   │
   │.md       │   │.md       │   │.md       │
   └──────────┘   └──────────┘   └──────────┘
                         │
              ┌──────────┴──────────┐
              │middleware.md │cache.md│
              └─────────────────────┘
```

---

## 常见场景

### 场景 1：需要添加新功能

1. 明确功能需求
2. 查看相关模块（如需要缓存 → cache.md）
3. 复制代码示例
4. 按项目实际调整

### 场景 2：遇到问题

| 问题 | 参考文件 |
|------|---------|
| 数据库连接失败 | database.md - 连接池配置 |
| 认证失败 | auth.md - Token 验证 |
| 单元测试怎么写 | testing.md - fixture |
| 部署到生产 | middleware.md - 健康检查 |

### 场景 3：代码审查

使用对应模块的"最佳实践"和"常见陷阱"检查代码：

```
代码有 SQL？ → architecture.md - 是否在 Repository 层
有异常处理？ → errors.md - 是否使用 BusinessException
有缓存？ → cache.md - 是否处理穿透/击穿
```

---

## 相关资源

- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 文档](https://docs.sqlalchemy.org/en/20/)
- [Pydantic 文档](https://docs.pydantic.dev/)

---

## 更新日志

- 2026-03-14: 初始版本，包含 12 个模块
