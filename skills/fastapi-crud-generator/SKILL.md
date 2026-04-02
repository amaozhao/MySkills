---
name: fastapi-crud-generator
description: FastAPI CRUD 模块生成器 - 给定 Pydantic Schema 或 SQLAlchemy Model，自动生成 Router/Service/Repository 代码。在需要生成 CRUD API、创建新资源端点或快速搭建 API 骨架时使用。
disable-model-invocation: true
---

# FastAPI CRUD Generator

## 概述

根据 Pydantic Schema 或 SQLAlchemy Model 自动生成完整的 CRUD 模块代码。

## 输入

提供以下任一：
- Pydantic Schema 定义
- SQLAlchemy Model 定义
- 或直接描述需要的功能

## 生成流程

1. **分析模型**：识别字段、类型、约束
2. **生成 Schema**：Request/Response Pydantic models
3. **生成 Repository**：数据访问层
4. **生成 Service**：业务逻辑层
5. **生成 Router**：API 端点

## 输出结构

```
app/
├── api/endpoints/{resource}.py   # Router
├── services/{resource}.py         # Service
├── repositories/{resource}.py     # Repository
└── schemas/{resource}.py          # Schema
```

## 规则

- 遵循 Router → Service → Repository 分层
- Repository 只做数据访问，不 commit
- Service 处理业务逻辑和 commit
- 使用 async/await
- 所有函数有类型注解

## 使用方式

```
/fastapi-crud-generator

输入：用户管理模块，需要 email、name、is_active 字段
```

## 示例

输入：
```python
class User(Base):
    id: int
    email: str
    name: str
    hashed_password: str
    is_active: bool = True
    created_at: datetime
```

输出：
- `schemas/user.py` - UserCreate, UserUpdate, UserResponse
- `repositories/user.py` - UserRepository
- `services/user.py` - create_user, get_user, update_user, delete_user
- `api/endpoints/user.py` - POST/GET/PUT/DELETE 端点
