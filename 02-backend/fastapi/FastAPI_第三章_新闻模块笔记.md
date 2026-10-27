# FastAPI 第三章｜AI 掘金头条：新闻模块

> **定位**：学完后用于快速复习、回忆核心概念、照着代码恢复手感，而不是逐页重读 PPT。  
> **资料依据**：《第三章_AI掘金头条-新闻模块.pdf》（39 页，黑马程序员《FastAPI 基础到实战》），并参照配套物料中的《API 接口规范文档》《database.sql》《项目后端设计说明文档》。  
> **覆盖范围**：项目结构与分层、模块化路由 `APIRouter`、ORM 配置与数据表、跨域 CORS、新闻分类/列表/详情三个接口、索引与排序、`update().where().values()`。  
> **阅读方法**：先看每节的「一句话记忆」，合上笔记说出原理，再看代码，最后做文末练习。

[TOC]

---

## 一、先建立整体认识

### 1. 这一章从"练语法"变成"做项目"

前两章是单文件练习，这一章开始进入完整项目：**AI 掘金头条**。课件第 2 页给出的项目功能与配套物料：

| 项目功能 | 配套物料 |
| --- | --- |
| 新闻模块、用户模块、收藏和浏览历史…… | 前端代码、接口规范文档、数据库 SQL 文件、设计说明文档 |

每一份物料的作用（课件第 3 页）：

| 物料 | 作用 |
| --- | --- |
| **前端项目代码** | Vue 框架实现页面结构、组件交互与接口调用，可在浏览器呈现完整界面 |
| **接口规范文档** | 定义各接口的请求方式、参数格式、响应结构、错误码，是前后端的统一交互标准 |
| **数据库 SQL 文件** | 表结构定义、初始化数据与约束规则，快速搭建数据存储环境 |
| **设计说明文档** | 系统说明整个后端系统的架构、模块划分、数据设计与接口规范 |

**一句话记忆：讲义负责讲思路，物料负责给准绳——接口字段看规范文档，表字段看 SQL 文件。**

本章只做**新闻模块**的三个接口：

```text
GET /api/news/categories   获取新闻分类列表
GET /api/news/list         获取新闻列表（分页 + 分类筛选）
GET /api/news/detail       获取新闻详情（浏览量 +1、相关新闻）
```

### 2. 前后端怎么对接

课件第 22 页的图很关键——两个地址不是同一个"源"：

```text
前端 Vue                后端 FastAPI
http://localhost:5173/  http://127.0.0.1:8000/
```

端口不同 → 浏览器判定为跨域 → 必须配置 CORS（第六章）。

前端调用方式（`src/store/modules/news.js`）：

```javascript
const response = await axios.get(`${apiConfig.baseURL}/api/news/list`, { params })
if (response.data && response.data.code === 200) {
  const newsData = response.data.data.list
}
```

后端返回的统一结构（接口规范文档）：

```json
{
  "code": 200,
  "message": "success",
  "data": {}
}
```

**一句话记忆：前端只认 `code === 200`，真正的数据在 `data` 里，字段名跟接口规范文档一字不差。**

### 3. 本章主线：接口实现四步法

课件第 17、19、33 页反复出现同一张图，这就是本章的骨架：

```mermaid
flowchart LR
    A["① 模块化路由<br/>APIRouter + include_router"] --> B["② 定义模型类<br/>Base + 表模型"]
    B --> C["③ 数据库 CRUD<br/>select / add / update / delete"]
    C --> D["④ 路由调用逻辑<br/>Depends 注入 + 返回响应"]
```

| 步骤 | 要做的事 | 依据 |
| --- | --- | --- |
| ① 模块化路由 | 定义 `APIRouter` 实例、注册 `include_router()` | 接口规范文档 |
| ② 定义模型类 | `class Base(DeclarativeBase)` + 表模型类 | 数据库表 |
| ③ 数据库 CRUD | `select()` / `add()` / `update()` / `delete()` | 业务需求 |
| ④ 路由调用逻辑 | `Depends` 注入数据库依赖、调用逻辑、响应结果 | 接口规范文档 |

**记住这条链**：路由是入口，模型是映射，CRUD 是手段，逻辑是编排。

---

## 二、运行前端项目与环境准备

### 1. 两步启动

课件第 4 页：

| 步骤 | 内容 |
| --- | --- |
| **01 安装 Node** | 安装运行构建工具（Vite / Webpack）的运行时 |
| **02 运行项目** | 进入项目根目录打开终端，执行 `npm run dev` |

```bash
# 1. 确认 Node 已安装
node -v

# 2. 进入前端项目目录（含 package.json 的那一层）
cd 03-前端项目代码/xwzx-news

# 3. 安装依赖（首次运行需要）
npm install

# 4. 启动开发服务器
npm run dev
```

启动成功后浏览器访问终端提示的地址，通常是 `http://localhost:5173/`。

### 2. 前端指向哪个后端

`src/config/api.js` 里写死了后端地址：

```javascript
export const apiConfig = {
  baseURL: 'http://127.0.0.1:8000',
}
```

**一句话记忆：前端只认 `127.0.0.1:8000`，后端不是跑在这个地址上，页面就一定是空的。**

### 3. 建议的联调顺序

1. 先启动后端，打开 `http://127.0.0.1:8000/docs`，用 Swagger 逐个试三个接口。
2. 接口都通了，再启动前端 `npm run dev`。
3. 页面空白时，先看浏览器 Network 面板：是 404（路径不对）、500（后端报错）还是 CORS 报错（第六章）。

> 这样排查的好处是：**问题只会出在一层**。前后端同时开调，报错时很难判断是谁的问题。

---

## 三、工程结构：五层分工

### 1. 目录结构

课件第 5 页把工程结构拆成五块，配套《设计说明文档》给出了完整目录：

```text
toutiao_backend/
├── crud/                 # 数据访问层：数据库增删改查逻辑（封装数据库操作）
│   ├── news.py
│   ├── users.py
│   ├── favorite.py
│   └── history.py
├── models/               # 数据模型层：SQLAlchemy ORM 模型
│   ├── news.py
│   ├── users.py
│   ├── favorite.py
│   └── history.py
├── routers/              # 路由层：按模块划分
│   ├── news.py
│   ├── users.py
│   ├── favorite.py
│   └── history.py
├── schemas/              # 数据验证模型：Pydantic
│   ├── news.py
│   └── ...
├── utils/                # 工具函数
├── config/               # 配置相关
│   └── db_conf.py
└── main.py               # 应用入口
```

### 2. 每层各管什么

| 层 | 目录 | 职责 | 不该做的事 |
| --- | --- | --- | --- |
| 路由层 | `routers/` | 解析参数、调 CRUD、拼响应 | 不写 SQL、不定义表 |
| 模型层 | `models/` | 表结构映射（列、类型、索引） | 不写业务查询 |
| 数据访问层 | `crud/` | 封装 `select` / `add` / `update` / `delete` | 不处理 HTTP |
| 校验层 | `schemas/` | Pydantic 请求/响应模型 | 不碰数据库 |
| 配置层 | `config/` | 引擎、会话工厂、缓存配置 | 不写业务 |

### 3. 一次请求在层间怎么走

```mermaid
flowchart TD
    R["浏览器 GET /api/news/list?categoryId=1"] --> RT["routers/news.py<br/>解析 categoryId / page / pageSize"]
    RT --> DB["Depends(get_db)<br/>取得 AsyncSession"]
    RT --> CR["crud/news.py<br/>select + where + offset/limit"]
    CR --> M["models/news.py<br/>映射 news 表"]
    M --> DB2[("MySQL news_app")]
    CR --> RT
    RT --> S["schemas/news.py<br/>校验/序列化响应"]
    S --> OUT["{ code, message, data }"]
```

**一句话记忆：路由不碰 SQL，CRUD 不碰 HTTP，模型只描述表——各司其职，改一处不牵连三处。**

---

## 四、模块化路由

### 1. 问题：`main.py` 会爆炸

课件第 7 页列了同一时期项目里的接口：`/users`、`/token`、`/news_categories`、`/news_list`、`/news_detail`、`/history_list`、`/favorite_list`…… 全部写在 `main.py` 里会变成一坨。

### 2. 定义：把接口拆进独立文件

课件第 9 页：

```python
from fastapi import APIRouter

# 创建 APIRouter 实例
router = APIRouter(prefix="/api/news", tags=["news"])

@router.get("/categories")
async def get_categories():
    return {"msg": "获取分类成功"}
```

三个关键点：

| 写法 | 含义 |
| --- | --- |
| `APIRouter()` | 一个"迷你应用"，可以挂载很多路由 |
| `prefix="/api/news"` | 该模块所有路径的公共前缀，写一次省十次 |
| `tags=["news"]` | 文档分组，`/docs` 里按标签折叠 |
| `@router.get(...)` | 注意是 `router` 不是 `app` |

**一句话记忆：`app` 是总闸，`router` 是分闸；`prefix` 决定路径，`tags` 决定文档分组。**

### 3. 注册：挂载到主应用

```python
# main.py
from routers import news

app.include_router(news.router)      # 把 news 模块的所有接口挂上去
```

挂载后最终路径 = `prefix` + 路由函数上的路径：

```text
prefix="/api/news" + "/categories"  →  /api/news/categories
```

### 4. 三个优势

课件第 8 页：

| 优势 | 说明 |
| --- | --- |
| **项目结构更清晰** | 接口按模块拆分，不会混在一起 |
| **更易维护** | 每个模块负责自己的接口，便于快速查找和定向修改 |
| **避免 `main.py` 爆炸** | `main.py` 只负责启动应用，不再堆满业务代码 |

### 5. 课件三问三答

| 问题 | 答案 |
| --- | --- |
| 什么是模块化路由？有什么优势？ | 把每个业务功能的接口拆分到独立文件里，再统一挂载到主应用中。优势：结构更清晰、更易维护 |
| `APIRouter(prefix="/api/news", tags=["news"])` 的含义？ | 创建一个路由实例，`/api/news` 为公共前缀，`news` 为文档分组标签 |
| 如何注册路由？ | 在 `main.py` 中 `app.include_router(news.router)` |

---

## 五、配置 ORM 与数据表

### 1. 先把数据导进去

课件第 13 页：MySQL 导入 SQL 文件。

```text
PyCharm → Database 插件 → 连接数据库
        → 右键 → SQL Script → Run SQL Script
        → 浏览 database.sql → 确认执行
```

数据库名：**`news_app`**（对应 `database.sql` 中的建库语句）。

### 2. 引擎、会话与依赖项

课件第 14 页（配置 ORM）：

```python
from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession, create_async_engine

# 数据库 URL（注意 utf8mb4，新闻正文含 emoji）
ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/news_app?charset=utf8mb4"

# 创建异步引擎
async_engine = create_async_engine(
    ASYNC_DATABASE_URL, echo=True, pool_size=10, max_overflow=20
)

# 会话工厂
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False
)

# 依赖项，用于获取数据库会话
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

与第二章的写法只有一处命名差异：**课件这里叫 `get_db`，第二章叫 `get_database`**，作用完全一样。习惯上项目里更常用 `get_db`。

**一句话记忆：`config/db_conf.py` 放引擎与 `get_db`，全项目共享一个引擎、每个请求一个会话。**

### 3. 两张核心表

新闻模块只涉及两张表（来自 `database.sql`）：

**`news_category`（新闻分类表）**

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `id` | INT UNSIGNED PK | 分类 ID |
| `name` | VARCHAR(50) UNIQUE | 分类名称 |
| `sort_order` | INT 默认 0 | 排序顺序 |
| `created_at` | TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | 更新时间 |

**`news`（新闻表）**

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `id` | INT UNSIGNED PK | 新闻 ID |
| `title` | VARCHAR(255) | 标题 |
| `description` | VARCHAR(500) | 简介 |
| `content` | TEXT | 正文 |
| `image` | VARCHAR(255) | 封面图 URL |
| `author` | VARCHAR(50) | 作者 |
| `category_id` | INT UNSIGNED | 分类 ID（外键） |
| `views` | INT UNSIGNED 默认 0 | 浏览量 |
| `publish_time` | TIMESTAMP | 发布时间 |
| `created_at` / `updated_at` | TIMESTAMP | 创建/更新时间 |

两条现成的索引（和课件第 37 页对得上）：

```sql
INDEX `fk_news_category_idx` (`category_id`),
INDEX `idx_publish_time` (`publish_time` DESC),
CONSTRAINT `fk_news_category` FOREIGN KEY (`category_id`)
  REFERENCES `news_category` (`id`) ON DELETE RESTRICT ON UPDATE CASCADE
```

> 这两个索引正好对应本章两个高频查询：**按分类过滤**（`category_id`）与**按时间排序**（`publish_time`）。

### 4. 模型类规范

课件第 20 页：

- 基类，继承 `DeclarativeBase`
- 数据库表模型类，继承基类
- **属性及类型参照数据库表定义**

```python
from datetime import datetime
from sqlalchemy import String, Text, Integer, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class NewsCategory(Base):
    __tablename__ = "news_category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True, comment="分类名称")
    sort_order: Mapped[int] = mapped_column(Integer, default=0, comment="排序顺序")
    created_at: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), comment="创建时间")
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now(), comment="更新时间")
```

**注意与第二章的区别**：本章的表自带 `created_at` / `updated_at`（数据库有默认值），所以基类只需要 `pass`，不必像第二章那样把时间字段提到基类。**跟着表走，不要照抄上一章。**

---

## 六、跨域 CORS

### 1. 什么是同源

课件第 21 页：

> 跨域资源共享（CORS）是一种浏览器安全机制，用于允许运行在一个源（Origin）的 Web 应用，通过浏览器向另一个源的服务器发起跨域 HTTP 请求，并在服务器授权的前提下获取资源。

**同源的三个条件（缺一不可）：协议、域名、端口。**

| 前端地址 | 后端地址 | 是否同源 | 原因 |
| --- | --- | --- | --- |
| `http://localhost:5173` | `http://localhost:5173` | ✅ 同源 | 完全相同 |
| `http://localhost:5173` | `http://127.0.0.1:8000` | ❌ 跨域 | 域名 + 端口都不同 |
| `http://localhost:5173` | `https://localhost:5173` | ❌ 跨域 | 协议不同 |
| `http://localhost:5173` | `http://localhost:8000` | ❌ 跨域 | 端口不同（5173 vs 8000） |

### 2. 为什么浏览器会报错

课件第 24 页的提问：**分析下图浏览器报错的原因？**

答案：**浏览器向另一个源的服务器发起了跨域 HTTP 请求**，而同源策略只允许同源请求。此时请求可能已经到达后端并成功返回，但浏览器**拒绝把响应交给页面**，控制台就会打印 CORS 相关的错误。

**一句话记忆：跨域不是后端拒绝了你，是浏览器拦住了响应。**

### 3. 解决：CORS 中间件

课件第 23 页：

```python
from fastapi.middleware.cors import CORSMiddleware

# 允许的来源（可以是域名列表）
origins = [
    "http://localhost",
    "http://localhost:3000",
    "https://your-frontend-domain.com"
]

# 添加 CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],      # 允许访问的源
    allow_credentials=True,   # 允许携带 Cookie
    allow_methods=["*"],      # 允许所有请求方法
    allow_headers=["*"],      # 允许所有请求头
)
```

四个参数对照：

| 参数 | 作用 | 生产环境建议 |
| --- | --- | --- |
| `allow_origins` | 允许哪些源访问 | 写具体域名，如 `["http://localhost:5173"]` |
| `allow_credentials` | 是否允许携带 Cookie | 需要登录态才开 |
| `allow_methods` | 允许的 HTTP 方法 | `["GET", "POST", "PUT", "DELETE"]` |
| `allow_headers` | 允许的请求头 | 至少要有 `Authorization`、`Content-Type` |

> ⚠️ `allow_origins=["*"]` 与 `allow_credentials=True` 同时使用在浏览器里是有额外限制的组合。开发阶段图省事可以这样写，**上线前一定把 `allow_origins` 改成实际前端域名**。

### 4. 课件三问三答

| 问题 | 答案 |
| --- | --- |
| 同源的三个条件？ | **协议、域名、端口** 完全相同 |
| 浏览器报错的原因？ | 浏览器向另一个源的服务器发起了跨域 HTTP 请求，同源策略拦截了响应 |
| 如何解决？ | 后端**全局配置 CORS 中间件**（`app.add_middleware(CORSMiddleware, ...)`） |

---

## 七、新闻模块三个接口

先说共用的路由实例：

```python
router = APIRouter(prefix="/api/news", tags=["news"])
```

### 1. 获取新闻分类列表

**接口**：`GET /api/news/categories`

| 参数 | 类型 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- | --- |
| `skip` | integer | 否 | 0 | 跳过记录数 |
| `limit` | integer | 否 | 100 | 返回记录数上限 |

课件第 18 页的骨架：

```python
@router.get("/categories")
async def get_categories(skip: int = 0, limit: int = 100):
    return {
        "code": 200,
        "message": "success",
        "data": "新闻分类列表"
    }
```

按接口规范文档，`data` 应该是**分类对象数组**：

```json
{
  "code": 200,
  "message": "success",
  "data": [
    { "id": 1, "created_at": "...", "updated_at": "...", "name": "科技", "sort_order": 0 }
  ]
}
```

**参考实现**（要点：`sort_order` 排序优先，缺失时再按 id）：

```python
from sqlalchemy import select

@router.get("/categories")
async def get_categories(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    stmt = (
        select(NewsCategory)
        .order_by(NewsCategory.sort_order.asc(), NewsCategory.id.asc())
        .offset(skip)
        .limit(limit)
    )
    result = await db.execute(stmt)
    categories = result.scalars().all()
    return {"code": 200, "message": "success", "data": categories}
```

> 分类是导航栏数据，量小、访问频繁，实际项目里常加缓存（`news:categories`，2 小时），本章不做要求。

### 2. 获取新闻列表

**接口**：`GET /api/news/list`

| 参数 | 类型 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- | --- |
| `categoryId` | integer | **是** | — | 分类 ID |
| `page` | integer | 否 | 1 | 页码 |
| `pageSize` | integer | 否 | 10 | 每页数量，最大 100 |

课件第 25 页：

```python
@router.get("/list")
async def get_news(
    category_id: int = Query(..., alias="categoryId"),
    page: int = 1,
    page_size: int = Query(10, le=100, alias="pageSize")
):
    return {
        "code": 200,
        "message": "success",
        "data": {
            "list": "news_list",
            "total": "total_count",
            "hasMore": "has_more"
        }
    }
```

**这段代码有三个必须讲清的点：**

**① `alias` 是干什么的**

```text
前端传的是 categoryId / pageSize（驼峰）
Python 参数要求 category_id / page_size（蛇形）
alias="categoryId"  →  把 URL 上的名字映射成 Python 变量名
```

```python
category_id: int = Query(..., alias="categoryId")
#             ↑ Python 变量     ↑ 前端真实参数名
```

不加 `alias`，前端传 `categoryId=1` 会被判成"缺少必填参数 `category_id`"，直接 422。

**② 响应字段同样用驼峰**

接口规范文档里 `data` 的结构是：

```json
{
  "list": [ ... ],
  "total": 100,
  "hasMore": true
}
```

Python 里习惯写 `has_more`，但**响应键名必须和文档一致**（`hasMore`），前端拿到就直接用。所以要么手动拼字典，要么用 Pydantic 的 `alias` 输出。

**③ 查询功能的三件事**（课件第 26 页）

> 响应结果：当前分类新闻列表、新闻总量、是否还有更多新闻。

```text
跳过数量 = (页码 - 1) × 每页数量
        ↓
select().offset(跳过数量).limit(pageSize)     ← 当前页数据
select(func.count())                          ← 总数量
(offset + 当前新闻列表条数) < 总量             ← 是否还有更多
```

**参考实现：**

```python
from sqlalchemy import select, func

@router.get("/list")
async def get_news(
    category_id: int = Query(..., alias="categoryId"),
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size

    # 1. 当前分类的新闻列表（按发布时间倒序）
    stmt = (
        select(News)
        .where(News.category_id == category_id)
        .order_by(News.publish_time.desc())
        .offset(offset)
        .limit(page_size)
    )
    result = await db.execute(stmt)
    news_list = result.scalars().all()

    # 2. 当前分类的新闻总量
    total = await db.scalar(
        select(func.count()).select_from(News).where(News.category_id == category_id)
    )

    # 3. 是否还有更多
    has_more = (offset + len(news_list)) < total

    return {
        "code": 200,
        "message": "success",
        "data": {"list": news_list, "total": total, "hasMore": has_more},
    }
```

> `hasMore` 的算法比"总数 / 每页数"更稳：它用**实际取到的条数**判断，最后一页不满、或中间有删除，都不会算错。

### 3. 获取新闻详情

**接口**：`GET /api/news/detail`

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | integer | 是 | 新闻 ID |

课件第 28 页：

```python
@router.get("/detail")
async def read_news_detail(news_id: int = Query(..., alias="id")):
    return {
        "code": 200,
        "message": "success",
        "data": {
            "id": "新闻id",
            "title": "新闻标题",
            "content": "",
            "relatedNews": "相关新闻"
        }
    }
```

同样用 `alias="id"` 把 URL 上的 `id` 映射成 `news_id`（避免和内置函数 `id` 重名，可读性更好）。

**这个接口要做三件事**（课件第 29 页）：

```text
按 id 查询新闻  →  浏览量 + 1  →  查询相关新闻（同分类的其他新闻）  →  响应结果
```

| 步骤 | 关键代码 |
| --- | --- |
| 按 id 查询 | `select().where(News.id == 新闻id)` |
| 浏览量 +1 | `update().where().values(字段=值)` |
| 相关新闻 | `select().where(News.id != news_id, News.category_id == 分类id)` |

**参考实现：**

```python
from fastapi import HTTPException
from sqlalchemy import select, update

@router.get("/detail")
async def read_news_detail(
    news_id: int = Query(..., alias="id"),
    db: AsyncSession = Depends(get_db),
):
    # 1. 按 id 查新闻
    news = await db.get(News, news_id)
    if news is None:
        raise HTTPException(status_code=404, detail="新闻不存在")

    # 2. 浏览量 + 1（直接写库，避免"先查再改"的并发丢更新）
    await db.execute(
        update(News).where(News.id == news_id).values(views=News.views + 1)
    )

    # 3. 相关新闻：同分类、排除自己
    related_result = await db.execute(
        select(News)
        .where(News.id != news_id, News.category_id == news.category_id)
        .order_by(News.publish_time.desc())
        .limit(5)
    )
    related = related_result.scalars().all()

    return {
        "code": 200,
        "message": "success",
        "data": {
            "id": news.id,
            "title": news.title,
            "content": news.content,
            "image": news.image,
            "author": news.author,
            "publishTime": news.publish_time,
            "categoryId": news.category_id,
            "views": news.views + 1,
            "relatedNews": related,
        },
    }
```

> ⚠️ 三个坑：① `db.get()` 后又被 `update()` 改了 `views`，返回的 `news.views` 是旧值，所以响应里要 `+1`（或者刷新对象）；② 响应字段用驼峰（`publishTime`、`categoryId`、`relatedNews`）；③ 记录不存在要返回 404，而不是 `null`。

---

## 八、接口实现四步法与完整示例

### 1. 四步法复习表

| 步骤 | 目录/文件 | 本章具体内容 |
| --- | --- | --- |
| ① 模块化路由 | `routers/news.py` | `APIRouter(prefix="/api/news", tags=["news"])`，`main.py` 里 `include_router` |
| ② 定义模型类 | `models/news.py` | `Base(DeclarativeBase)` + `NewsCategory` / `News` |
| ③ 数据库 CRUD | `crud/news.py` | `select` / `add` / `update` / `delete` |
| ④ 路由调用逻辑 | `routers/news.py` | `Depends(get_db)`、调用 CRUD、拼响应 |

### 2. 完整可运行的最小实现

**`config/db_conf.py`**

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/news_app?charset=utf8mb4"

async_engine = create_async_engine(
    ASYNC_DATABASE_URL, echo=True, pool_size=10, max_overflow=20
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine, class_=AsyncSession, expire_on_commit=False
)


async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

**`models/news.py`**

```python
from datetime import datetime
from sqlalchemy import String, Text, Integer, DateTime, Index, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class NewsCategory(Base):
    __tablename__ = "news_category"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50), unique=True, comment="分类名称")
    sort_order: Mapped[int] = mapped_column(Integer, default=0, comment="排序顺序")
    created_at: Mapped[datetime] = mapped_column(DateTime, insert_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now())


class News(Base):
    __tablename__ = "news"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(255), comment="新闻标题")
    description: Mapped[str | None] = mapped_column(String(500), comment="新闻简介")
    content: Mapped[str] = mapped_column(Text, comment="新闻内容")
    image: Mapped[str | None] = mapped_column(String(255), comment="封面图片URL")
    author: Mapped[str | None] = mapped_column(String(50), comment="作者")
    category_id: Mapped[int] = mapped_column(Integer, comment="分类ID")
    views: Mapped[int] = mapped_column(Integer, default=0, comment="浏览量")
    publish_time: Mapped[datetime] = mapped_column(DateTime, insert_default=func.now())
    created_at: Mapped[datetime] = mapped_column(DateTime, insert_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now())

    # 创建索引：提升查询速度
    __table_args__ = (
        Index("fk_news_category_idx", "category_id"),
        Index("idx_publish_time", "publish_time"),
    )
```

**`crud/news.py`**

```python
from sqlalchemy import select, func, update
from sqlalchemy.ext.asyncio import AsyncSession

from models.news import News, NewsCategory


async def get_categories(db: AsyncSession, skip: int = 0, limit: int = 100):
    stmt = (
        select(NewsCategory)
        .order_by(NewsCategory.sort_order.asc(), NewsCategory.id.asc())
        .offset(skip).limit(limit)
    )
    result = await db.execute(stmt)
    return result.scalars().all()


async def get_news_page(db: AsyncSession, category_id: int,
                        offset: int = 0, page_size: int = 10):
    """返回 (当前页数据, 总条数)"""
    stmt = (
        select(News)
        .where(News.category_id == category_id)
        .order_by(News.publish_time.desc())
        .offset(offset).limit(page_size)
    )
    rows = (await db.execute(stmt)).scalars().all()

    total = await db.scalar(
        select(func.count()).select_from(News).where(News.category_id == category_id)
    )
    return rows, total


async def get_news_by_id(db: AsyncSession, news_id: int):
    return await db.get(News, news_id)


async def increase_views(db: AsyncSession, news_id: int) -> bool:
    stmt = update(News).where(News.id == news_id).values(views=News.views + 1)
    result = await db.execute(stmt)
    await db.commit()
    # 更新 → 检查数据库是否真的命中了数据 → 命中了返回 True
    return result.rowcount > 0


async def get_related_news(db: AsyncSession, news_id: int,
                           category_id: int, limit: int = 5):
    stmt = (
        select(News)
        .where(News.id != news_id, News.category_id == category_id)
        .order_by(News.publish_time.desc())
        .limit(limit)
    )
    return (await db.execute(stmt)).scalars().all()
```

**`routers/news.py`**

```python
from fastapi import APIRouter, Depends, HTTPException, Query
from sqlalchemy.ext.asyncio import AsyncSession

from config.db_conf import get_db
from crud import news as news_crud

router = APIRouter(prefix="/api/news", tags=["news"])


@router.get("/categories")
async def get_categories(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    data = await news_crud.get_categories(db, skip, limit)
    return {"code": 200, "message": "success", "data": data}


@router.get("/list")
async def get_news(
    category_id: int = Query(..., alias="categoryId"),
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size
    rows, total = await news_crud.get_news_page(db, category_id, offset, page_size)
    return {
        "code": 200,
        "message": "success",
        "data": {
            "list": rows,
            "total": total,
            "hasMore": (offset + len(rows)) < total,
        },
    }


@router.get("/detail")
async def read_news_detail(
    news_id: int = Query(..., alias="id"),
    db: AsyncSession = Depends(get_db),
):
    news = await news_crud.get_news_by_id(db, news_id)
    if news is None:
        raise HTTPException(status_code=404, detail="新闻不存在")

    await news_crud.increase_views(db, news_id)
    related = await news_crud.get_related_news(db, news_id, news.category_id)

    return {
        "code": 200,
        "message": "success",
        "data": {
            "id": news.id,
            "title": news.title,
            "content": news.content,
            "image": news.image,
            "author": news.author,
            "publishTime": news.publish_time,
            "categoryId": news.category_id,
            "views": news.views + 1,
            "relatedNews": related,
        },
    }
```

**`main.py`**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from routers import news

app = FastAPI(title="AI 掘金头条")

# 跨域：允许前端开发服务器访问
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 模块化路由
app.include_router(news.router)
```

### 3. 启动与验证

```bash
uvicorn main:app --reload
```

| 测试 | 预期 |
| --- | --- |
| `GET /api/news/categories` | 返回分类数组，`sort_order` 小的在前 |
| `GET /api/news/list?categoryId=1` | 返回该分类第一页 10 条 + `total` + `hasMore` |
| `GET /api/news/list?categoryId=1&page=2&pageSize=20` | 第 2 页，每页 20 条 |
| `GET /api/news/list` | 缺少 `categoryId` → 422（必填） |
| `GET /api/news/list?categoryId=1&pageSize=500` | `pageSize` 超过 100 → 422 |
| `GET /api/news/detail?id=1` | 返回详情，`views` 每次 +1 |
| `GET /api/news/detail?id=99999` | 404 |
| 打开前端 `localhost:5173` | 首页新闻列表正常渲染，无 CORS 报错 |

---

## 九、索引、排序与 `update` 语句

课件第 37、38 页在总结里补了三个工程化要点，都很实用。

### 1. 模型类中如何为字段创建索引

**一句话记忆：`__table_args__` 里放 `Index()`，字段写字符串名。**

```python
class News(Base):
    __tablename__ = "news"
    # ... 其余字段见第五章的模型定义 ...

    # 创建索引：提升查询速度
    __table_args__ = (
        Index('fk_news_category_idx', 'category_id'),
        Index('idx_publish_time', 'publish_time')
    )
```

| 索引 | 服务哪条查询 |
| --- | --- |
| `fk_news_category_idx (category_id)` | `where(News.category_id == ...)` 分类筛选 |
| `idx_publish_time (publish_time)` | `order_by(News.publish_time.desc())` 时间排序 |

> 索引不是越多越好：每个索引都会拖慢写入（INSERT/UPDATE 要维护索引）。只为**高频查询条件**建索引。

### 2. ORM 如何排序

```python
select().order_by(News.views.desc())     # 按浏览量降序
select().order_by(News.publish_time.asc())  # 按发布时间升序
select().order_by(News.category_id.asc(), News.publish_time.desc())  # 多字段
```

- **默认升序**，`desc()` 表示降序
- 有多个字段时，**排在前面的优先**

### 3. 用 ORM 修改数据

**一句话记忆：`update().where().values()`。**

```python
from sqlalchemy import update

async def increase_news_views(db: AsyncSession, news_id: int):
    stmt = update(News).where(News.id == news_id).values(views=News.views + 1)
    result = await db.execute(stmt)
    await db.commit()
    # 更新 → 检查数据库是否真的命中了数据 → 命中了返回 True
    return result.rowcount > 0
```

三个要点：

| 写法 | 含义 |
| --- | --- |
| `.values(views=News.views + 1)` | 让**数据库**做 `views = views + 1`，而不是先在 Python 里加 |
| `result.rowcount` | 实际影响的行数，`> 0` 说明这条新闻存在 |
| `await db.commit()` | 不提交就不落库 |

**为什么不"先查再改"？** 两个用户同时打开同一篇新闻：

```text
先查再改（错误）：A 读到 100 → B 读到 100 → A 写 101 → B 写 101   （丢了一次）
直接 update（正确）：A 执行 views = views + 1 → B 执行 views = views + 1  （102）
```

这就是**并发下的丢失更新**，`update().values()` 把加法交给数据库，天然避免。

---

## 十、原课件练习题整理（建议先独立完成）

### 练习 1：搭建项目骨架（课件第 5、9 页）

按 `crud/`、`models/`、`routers/`、`schemas/`、`config/` 分层创建目录，在 `routers/news.py` 里建一个 `prefix="/api/news"` 的路由实例，并在 `main.py` 中挂载。

### 练习 2：导入数据并配置 ORM（课件第 13、14 页）

导入 `database.sql`，写出引擎、会话工厂与 `get_db` 依赖项。

### 练习 3：实现分类接口（课件第 18、20 页）

`GET /api/news/categories`，支持 `skip` / `limit`，返回 `{code, message, data}`，`data` 是分类数组。

### 练习 4：实现新闻列表接口（课件第 25、26 页）

`GET /api/news/list?categoryId=1&page=1&pageSize=10`：
返回该分类的当页数据、总条数、`hasMore`。

### 练习 5：实现新闻详情接口（课件第 28、29 页）

`GET /api/news/detail?id=1`：
返回详情，浏览量 +1，附带同分类的相关新闻（排除自己）；不存在返回 404。

### 练习 6：配置跨域（课件第 23 页）

让 `http://localhost:5173` 的前端能正常请求后端，并在浏览器里验证没有 CORS 报错。

### 练习 7（思考题，课件第 37、38 页）

1. 为什么 `news` 表要给 `category_id` 和 `publish_time` 建索引？
2. 浏览量为什么用 `update().values(views=News.views + 1)` 而不是先查再改？

---

## 十一、练习参考答案

### 练习 1

```python
# routers/news.py
from fastapi import APIRouter

router = APIRouter(prefix="/api/news", tags=["news"])
```

```python
# main.py
from fastapi import FastAPI
from routers import news

app = FastAPI(title="AI 掘金头条")
app.include_router(news.router)
```

### 练习 2

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/news_app?charset=utf8mb4"

async_engine = create_async_engine(
    ASYNC_DATABASE_URL, echo=True, pool_size=10, max_overflow=20
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine, class_=AsyncSession, expire_on_commit=False
)


async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

### 练习 3

```python
from fastapi import Depends, Query
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

@router.get("/categories")
async def get_categories(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=100),
    db: AsyncSession = Depends(get_db),
):
    stmt = (
        select(NewsCategory)
        .order_by(NewsCategory.sort_order.asc(), NewsCategory.id.asc())
        .offset(skip).limit(limit)
    )
    categories = (await db.execute(stmt)).scalars().all()
    return {"code": 200, "message": "success", "data": categories}
```

### 练习 4

```python
from sqlalchemy import select, func

@router.get("/list")
async def get_news(
    category_id: int = Query(..., alias="categoryId"),
    page: int = Query(1, ge=1),
    page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
    db: AsyncSession = Depends(get_db),
):
    offset = (page - 1) * page_size

    stmt = (
        select(News)
        .where(News.category_id == category_id)
        .order_by(News.publish_time.desc())
        .offset(offset).limit(page_size)
    )
    rows = (await db.execute(stmt)).scalars().all()

    total = await db.scalar(
        select(func.count()).select_from(News).where(News.category_id == category_id)
    )

    return {
        "code": 200,
        "message": "success",
        "data": {
            "list": rows,
            "total": total,
            "hasMore": (offset + len(rows)) < total,
        },
    }
```

### 练习 5

```python
from fastapi import HTTPException
from sqlalchemy import select, update

@router.get("/detail")
async def read_news_detail(
    news_id: int = Query(..., alias="id"),
    db: AsyncSession = Depends(get_db),
):
    news = await db.get(News, news_id)
    if news is None:
        raise HTTPException(status_code=404, detail="新闻不存在")

    await db.execute(
        update(News).where(News.id == news_id).values(views=News.views + 1)
    )

    related = (await db.execute(
        select(News)
        .where(News.id != news_id, News.category_id == news.category_id)
        .order_by(News.publish_time.desc())
        .limit(5)
    )).scalars().all()

    return {
        "code": 200,
        "message": "success",
        "data": {
            "id": news.id,
            "title": news.title,
            "content": news.content,
            "image": news.image,
            "author": news.author,
            "publishTime": news.publish_time,
            "categoryId": news.category_id,
            "views": news.views + 1,
            "relatedNews": related,
        },
    }
```

### 练习 6

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],   # 前端开发服务器
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

验证：打开浏览器控制台 Network，响应头里应出现 `access-control-allow-origin: http://localhost:5173`。

### 练习 7

1. `category_id` 是分类筛选的 `where` 条件，`publish_time` 是列表排序的 `order_by` 字段；没有索引，MySQL 每次都要全表扫描，新闻量一大就慢。`fk_news_category_idx` 还兼作外键索引，`idx_publish_time` 让"按时间倒序取前 N 条"能直接走索引。
2. "先查再改"会读到旧值再写回，两个并发请求都读到 100、都写 101，丢掉一次浏览；`update().values(views=News.views + 1)` 让数据库执行 `views = views + 1`，是原子操作，不会丢。

---

## 十二、最容易混淆的知识点

| 容易混淆 | 正确认识 |
| --- | --- |
| `app.get` 与 `router.get` | 路由模块里用 `router`，主应用里用 `app` |
| `prefix` 与函数路径 | 最终路径 = `prefix` + 函数上的路径，别重复写 `/api/news` |
| `include_router` 的位置 | 必须在 `app.add_middleware` 之后、启动之前，顺序不影响路径 |
| `alias="categoryId"` | 把**URL 上的参数名**映射成 Python 变量名，不加就 422 |
| 请求参数驼峰、响应字段驼峰 | 都是前端联调约定，Python 内部用蛇形，边界上转换 |
| `skip`/`limit` 与 `page`/`pageSize` | 前者是"跳过多少"，后者是"第几页"，本项目分页用后者 |
| `hasMore` 算法 | `(offset + 当前条数) < total`，不是 `total > page * pageSize` |
| CORS 报错 | 是**浏览器**拦截响应，不是后端返回 4xx/5xx |
| `allow_origins=["*"]` + 凭证 | 浏览器对"通配源 + 携带凭证"有额外限制，生产写具体域名 |
| `update()` 与 `db.get()` 后改属性 | 前者直接发 SQL，后者靠 ORM 脏检查；批量/并发场景用前者 |
| `result.rowcount` | 表示实际影响的行数，可用来判断记录是否存在 |
| `get_db` 里已 `commit` | CRUD 里再 `commit` 不算错，但要清楚事务边界在哪一层 |
| 索引 | 加速查询、拖慢写入，只为高频条件建 |

### 课件示例中的整理说明

1. **三个接口的返回体**：课件写的是占位字符串（`"新闻分类列表"`、`"news_list"`、`"total_count"`），本笔记按《API 接口规范文档》补成真实字段结构，方便照着跑通。
2. **`get_categories` 的参数**：课件写 `skip: int = 0, limit: int = 100`，未限制上限。本笔记加 `Query(100, ge=1, le=100)`，与接口文档"最大 100"一致。
3. **列表接口的排序**：课件示例未写 `order_by`，但第 37 页专门讲了 `idx_publish_time`。本笔记补上 `order_by(News.publish_time.desc())`，否则分页结果顺序不稳定。
4. **详情接口的浏览量**：课件在总结页用 `update().values()` 实现自增。注意 `db.get()` 拿到的对象不会被这条 SQL 同步，响应里要显式 `views + 1`。
5. **`get_db` 与 `crud` 都提交**：课件两处都写了 `commit()`，功能上没错，但事务边界会在两层各切一刀；实际项目建议只在一处提交。

以上属于**可运行性与一致性修正**，不改变原课件知识点。

---

## 十三、闭卷自测：10 分钟检验掌握情况

1. 模块化路由的三步是什么？`prefix` 和 `tags` 分别控制什么？
2. 最终 URL 是 `prefix` 加什么？
3. `APIRouter` 和 `FastAPI` 实例的关系是什么？
4. 工程结构里 `crud/`、`models/`、`routers/`、`schemas/` 各负责什么？
5. 同源的三个条件是什么？`localhost:5173` 访问 `127.0.0.1:8000` 为什么跨域？
6. CORS 中间件四个参数分别管什么？
7. `Query(..., alias="categoryId")` 里的 `alias` 解决什么问题？
8. 新闻列表接口要返回哪三样东西？`hasMore` 怎么算？
9. 新闻详情接口要做哪三件事？
10. `update().where().values()` 与"先查再改"相比好在哪？
11. `result.rowcount` 有什么用？
12. 为什么给 `category_id` 和 `publish_time` 建索引？

**达标标准**：能不看笔记写出"三层目录 + 三个新闻接口 + CORS 配置"，并用 Swagger 逐个调通。

**参考答案**（先自己回答，再对照）：

1. 建 `APIRouter` 实例 → 写 `@router.get` 接口 → `main.py` 里 `include_router`；`prefix` 管路径前缀，`tags` 管文档分组。  
2. 加路由函数上的子路径，如 `/api/news` + `/categories`。  
3. `APIRouter` 是"迷你应用/分闸"，必须挂到 `FastAPI` 实例这个总闸上才生效。  
4. `crud/` 封装增删改查；`models/` 是表结构映射；`routers/` 是接口与参数解析；`schemas/` 是 Pydantic 校验模型。  
5. 协议、域名、端口；域名（localhost vs 127.0.0.1）和端口（5173 vs 8000）都不同。  
6. `allow_origins` 允许的源、`allow_credentials` 是否带 Cookie、`allow_methods` 允许的方法、`allow_headers` 允许的请求头。  
7. 前端传驼峰参数名，Python 用蛇形变量名，`alias` 负责映射，不加会判为缺少必填参数。  
8. 当前分类的新闻列表、新闻总量、是否还有更多；`(offset + 当前条数) < total`。  
9. 按 id 查新闻、浏览量 +1、查同分类的其他新闻作为相关新闻。  
10. 把加法交给数据库，避免并发下的丢失更新，也少一次查询。  
11. 表示实际影响的行数，可判断目标记录是否存在（`rowcount > 0`）。  
12. 分别服务"按分类过滤"和"按时间排序"两个高频操作，避免全表扫描。

---

## 十四、一页速查

### 常用命令

```bash
# 后端
python -m pip install "sqlalchemy[asyncio]" aiomysql
uvicorn main:app --reload

# 前端
node -v
npm install
npm run dev          # 默认 http://localhost:5173
```

### 模块化路由模板

```python
# routers/news.py
from fastapi import APIRouter

router = APIRouter(prefix="/api/news", tags=["news"])

@router.get("/categories")
async def get_categories():
    return {"code": 200, "message": "success", "data": []}
```

```python
# main.py
from routers import news
app.include_router(news.router)
```

### CORS 模板

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 新闻列表查询模板

```python
offset = (page - 1) * page_size
rows = (await db.execute(
    select(News)
    .where(News.category_id == category_id)
    .order_by(News.publish_time.desc())
    .offset(offset).limit(page_size)
)).scalars().all()

total = await db.scalar(
    select(func.count()).select_from(News).where(News.category_id == category_id)
)
has_more = (offset + len(rows)) < total
```

### 核心知识结构

```text
AI 掘金头条 · 新闻模块
├── 工程结构
│   ├── routers/   路由层（按模块）
│   ├── models/    模型层（SQLAlchemy）
│   ├── crud/      数据访问层（增删改查）
│   ├── schemas/   校验层（Pydantic）
│   └── config/    配置（引擎、get_db）
├── 模块化路由
│   ├── APIRouter(prefix, tags)
│   └── app.include_router(...)
├── 跨域 CORS
│   ├── 同源：协议 + 域名 + 端口
│   └── CORSMiddleware
├── 新闻分类  GET /api/news/categories   skip / limit
├── 新闻列表  GET /api/news/list         categoryId / page / pageSize
│                              → list + total + hasMore
└── 新闻详情  GET /api/news/detail       id
                               → 详情 + 浏览量+1 + relatedNews
```

### 三个接口对照

| 接口 | 关键参数 | 关键 ORM |
| --- | --- | --- |
| `/categories` | `skip`、`limit` | `select().order_by(sort_order).offset().limit()` |
| `/list` | `categoryId`(alias)、`page`、`pageSize`(alias) | `select().where(category_id).order_by(publish_time.desc()).offset().limit()` + `func.count()` |
| `/detail` | `id`(alias) | `select().where(id)` + `update().where().values(views+1)` + `select().where(id != ..., category_id == ...)` |

### 复习顺序建议

- **第 1 遍（理解）**：看一至六节，把"分层 + 路由 + 跨域"三件事串起来。
- **第 2 遍（练手）**：自己写第八节的四文件实现，不要复制粘贴。
- **第 3 遍（检测）**：完成第十节练习和第十三节闭卷自测。
- **以后快速回顾**：优先看第七节的三个接口、第十二节易混点、第十四节速查。

> **本章边界**：课件围绕新闻模块讲解了工程结构、模块化路由、ORM 配置、CORS 与三个查询接口，**没有**系统讲解用户认证（`Authorization` 令牌校验）、收藏与历史模块、Redis 缓存、表关联 `relationship`、Alembic 迁移、统一异常处理与分页性能优化。这些内容后续单独整理，不要与本章混在一起。
