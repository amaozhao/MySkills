---
name: python-project
description: Python 项目最佳实践 - 完整技术参考，按需引用各子模块
---

# Python Project Best Practices

## 概述

本模块提供 Python 项目的完整技术参考，包含项目结构、类型提示、异步编程、代码风格等核心主题。

## 快速导航

| 需要什么？ | 引用文件 | 优先级 |
|-----------|---------|--------|
| 项目骨架 | [structure.md](./structure.md) | 必读 |
| 类型提示 | [type-hints.md](./type-hints.md) | 必读 |
| 异步编程 | [async.md](./async.md) | 必读 |
| 代码风格 | [style.md](./style.md) | 推荐 |
| 设计原则 | [principles.md](./principles.md) | 推荐 |
| 错误处理 | [errors.md](./errors.md) | 推荐 |
| 新特性 | [new-features.md](./new-features.md) | 可选 |
| 打包发布 | [packaging.md](./packaging.md) | 可选 |
| 单元测试 | [testing.md](./testing.md) | 推荐 |

## 推荐项目结构

```
project/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── core/
│       ├── models/
│       ├── services/
│       └── api/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── pyproject.toml
├── .env
├── .gitignore
└── README.md
```

---

## 快速开始

### 1. 创建项目

```bash
# 使用 uv 创建项目
uv init myproject
cd myproject

# 添加依赖
uv add fastapi uvicorn sqlalchemy
uv add -D pytest pytest-asyncio httpx
```

### 2. 配置 pyproject.toml

参考 [structure.md](./structure.md) 设置项目配置。

### 3. 使用类型提示

参考 [type-hints.md](./type-hints.md) 添加类型注解。

### 4. 编写异步代码

参考 [async.md](./async.md) 使用 asyncio。

---

## 模块说明

### 核心模块（必读）

- **[structure.md](./structure.md)** - 项目结构、src-layout、pyproject.toml
- **[type-hints.md](./type-hints.md)** - 类型提示、PEP 695 语法
- **[async.md](./async.md)** - asyncio 异步编程模式

### 增强模块（推荐）

- **[style.md](./style.md)** - PEP 8 代码风格
- **[principles.md](./principles.md)** - DRY/KISS/YAGNI 设计原则
- **[errors.md](./errors.md)** - 异常处理最佳实践
- **[testing.md](./testing.md)** - 单元测试基础

### 进阶模块（可选）

- **[new-features.md](./new-features.md)** - Python 3.12+ 新特性
- **[packaging.md](./packaging.md)** - 打包发布到 PyPI

---

## 使用方式

### 按需引用

当你需要某个功能时，直接阅读对应文件：

```
需要类型提示？ → 阅读 type-hints.md
需要异步编程？ → 阅读 async.md
```

### 完整学习

如果你是 Python 新手，建议按顺序学习：

1. [structure.md](./structure.md) - 项目结构
2. [type-hints.md](./type-hints.md) - 类型系统
3. [style.md](./style.md) - 代码规范
4. [async.md](./async.md) - 异步编程
5. [errors.md](./errors.md) - 错误处理
6. [testing.md](./testing.md) - 测试

---

## 依赖关系

```
                    ┌─────────────┐
                    │structure.md │
                    └──────┬──────┘
                           │
              ┌─────────────┼─────────────┐
              │            │            │
              ▼            ▼            ▼
       ┌──────────┐  ┌──────────┐ ┌──────────┐
       │type-hints│  │ async.md │ │ style.md │
       └────┬─────┘  └────┬─────┘ └────┬─────┘
            │            │            │
            └────────────┼────────────┘
                         │
              ┌──────────┴──────────┐
              │  principles.md     │
              └──────────┬──────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │errors.md │   │testing  │   │packaging │
   └──────────┘   └──────────┘   └──────────┘
```

---

## 常见场景

| 场景 | 参考文件 |
|------|---------|
| 新建项目 | structure.md |
| 类型报错 | type-hints.md |
| 异步性能问题 | async.md |
| 代码review | style.md |
| 设计决策 | principles.md |
| 异常处理 | errors.md |
| 写测试 | testing.md |
| 发布到 PyPI | packaging.md |

---

## 更新日志

- 2026-03-14: 初始版本
