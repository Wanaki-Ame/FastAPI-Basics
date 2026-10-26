# FastAPI 第二章｜中间件、依赖注入与 ORM

> **定位**：学完后用于快速复习、回忆核心概念、照着代码恢复手感，而不是逐页重读 PPT。  
> **资料依据**：《第二章_FastAPI 进阶.pdf》（50 页，黑马程序员《FastAPI 基础到实战》，主题「中间件、依赖注入、ORM」）。  
> **覆盖范围**：中间件定义与执行顺序、依赖注入三步、SQLAlchemy 异步引擎与模型类、启动建表、会话依赖项 `yield`、查询/条件/聚合/分页、增删改查。  
> **阅读方法**：先看每节的「一句话记忆」，合上笔记说出原理，再看代码，最后做文末练习。

[TOC]

---

## 一、先建立整体认识

### 1. 这三块内容分别在解决什么问题

第一章解决的是"单个接口怎么写"，这一章解决的是"接口一多，重复代码怎么收"。

| 内容 | 解决的问题 | 作用范围 |
| --- | --- | --- |
| **中间件** | 每个请求都要做的同一件事（日志、认证、CORS、耗时） | **所有**请求，一个都跑不掉 |
| **依赖注入** | 多个接口共用的参数与逻辑（分页参数、数据库会话、当前用户） | **声明了才生效**的接口 |
| **ORM** | 不想手写 SQL，又要管理连接与事务 | 数据访问层 |

**一句话记忆：中间件管全局，依赖主管局部，ORM 管数据。**

一次请求经过的位置关系：

```text
客户端请求
   │
   ▼
┌──────────────────────────────┐
│ 中间件 A（请求阶段）           │
│  ┌────────────────────────┐  │
│  │ 中间件 B（请求阶段）     │  │
│  │  ┌──────────────────┐  │  │
│  │  │ 依赖项解析        │  │  │  ← Depends 在这里执行
│  │  │ （Query / DB 会话）│  │  │
│  │  ├──────────────────┤  │  │
│  │  │ 路径操作函数      │  │  │  ← 业务代码
│  │  └──────────────────┘  │  │
│  │ 中间件 B（响应阶段）     │  │
│  └────────────────────────┘  │
│ 中间件 A（响应阶段）           │
└──────────────────────────────┘
   │
   ▼
返回响应
```

### 2. 学习顺序建议

1. 先看**中间件**（P2–P6），它最容易理解，代码量也最少。
2. 再看**依赖注入**（P7–P13），它是后面数据库会话能写出来的前提。
3. 最后看 **ORM**（P14–P43），这一半内容都在讲同一件事：用对象操作数据库。

> 第二章的依赖注入是为第三章的 `get_database` 铺路的：看不懂 `Depends`，就一定看不懂数据库会话为什么能自动关闭。

---

## 二、中间件：给所有请求套一层壳

### 1. 为什么需要中间件

课件第 2 页给的两个场景：

- **多个接口都需要验证用户身份** → 每个函数开头都要写一遍取 token、校验、抛异常。
- **多个接口都需要记录日志、性能数据** → 每个函数结尾都要算一次耗时。

这些逻辑**与具体业务无关，但每个接口都要做**，一旦需求变化就要改几十处。中间件就是把这个"公共动作"抽出来，只写一次。

### 2. 中间件是什么

**一句话记忆：中间件是一个每次请求都会执行、并且在响应返回前再执行一次的包装层。**

课件原文：

> 中间件（Middleware）是一个在每次请求进入 FastAPI 应用时都会被执行的函数。它在请求到达实际的路径操作（路由处理函数）之前运行，并且在响应返回给客户端之前再运行一次。

所以一个中间件函数里天然有两个阶段：

- `await call_next(request)` **之前** → 请求阶段
- `await call_next(request)` **之后** → 响应阶段

### 3. 执行流程：洋葱模型

```mermaid
sequenceDiagram
    participant C as 客户端
    participant A as 中间件 A
    participant B as 中间件 B
    participant R as 路由处理函数

    C->>A: 请求进入
    A->>B: call_next(request)
    B->>R: call_next(request)
    R-->>B: response
    B-->>A: 传递响应
    A-->>C: 返回响应
```

课件第 4 页的文字版：

```text
客户端 → 中间件A → 中间件B → 路由处理函数
客户端 ← 中间件A ← 中间件B ← 路由处理函数
```

**越早进入的，越晚退出**——这就是"洋葱"。

### 4. 怎么定义

**一句话记忆：函数顶部加 `@app.middleware("http")`，签名固定是 `(request, call_next)`。**

```python
@app.middleware("http")
async def middleware(request, call_next):
    print('中间件开始处理 -- start')
    response = await call_next(request)
    print('中间件处理完成 -- end')
    return response
```

三条不能省的规矩：

| 规矩 | 少了会怎样 |
| --- | --- |
| 必须是 `async def` | 普通 `def` 无法 `await` |
| 必须 `await call_next(request)` | 请求走不到路由，接口直接报错 |
| 必须 `return response` | 客户端拿不到响应 |

### 5. 多个中间件的执行顺序

**一句话记忆：后定义的先执行——请求自下而上，响应自上而下。**

```python
@app.middleware("http")
async def mw1(request, call_next):      # 请求阶段第 2 个
    print("mw1 前")
    response = await call_next(request)
    print("mw1 后")                     # 响应阶段第 1 个
    return response


@app.middleware("http")
async def mw2(request, call_next):      # 后定义 → 最外层 → 请求阶段第 1 个
    print("mw2 前")
    response = await call_next(request)
    print("mw2 后")                     # 响应阶段第 2 个
    return response
```

输出：

```text
mw2 前 → mw1 前 → （路由函数）→ mw1 后 → mw2 后
```

课件第 6 页的结论就是四个字：**自下而上**。

### 6. 典型应用场景

课件第 3 页列了五类：

| 场景 | 做法 |
| --- | --- |
| **日志记录** | 记录方法、URL、耗时、状态码 |
| **身份认证** | 统一校验 token，不合法直接拦截 |
| **跨域处理** | 统一添加 CORS 响应头 |
| **响应头处理** | 加 `X-Process-Time`、缓存策略、安全头 |
| **性能监控** | 统计接口响应时间，超时告警 |

**综合小例子（日志 + 耗时 + 响应头一次做完）：**

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    cost_ms = (time.perf_counter() - start) * 1000
    response.headers["X-Process-Time"] = f"{cost_ms:.2f}ms"
    print(f"{request.method} {request.url.path} -> {response.status_code} ({cost_ms:.2f}ms)")
    return response
```

**理解补充（非课件原结论）**：跨域一般不手写，直接用官方提供的 `CORSMiddleware`。

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],       # 生产环境应写具体域名
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 7. 课件三问三答

| 问题 | 答案 |
| --- | --- |
| 中间件作用是什么？ | 为**每个请求**添加统一的处理逻辑（日志、身份认证、跨域、设置响应头、性能监控等） |
| 中间件怎么定义？ | 函数顶部使用装饰器 `@app.middleware("http")` |
| 多个中间件的执行顺序？ | **自下而上** |

---

## 三、依赖注入：只声明"我要什么"

### 1. 从一段重复代码说起

课件第 7、9 页摆了三个接口，除了返回值，参数与逻辑完全一样：

```python
@app.get('/news/news_list')
async def get_news_list(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    # 分页逻辑...
    return {"list": "新闻列表"}


@app.get("/users/user_list")
async def get_user_list(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    # 分页逻辑...
    return {"users": "用户列表"}


@app.get("/news/category")
async def get_category(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    # 分页逻辑...
    return {"category": "新闻分类"}
```

默认值从 10 改成 20，就要改三处——**这就是依赖注入要消掉的重复**。

### 2. 依赖项与注入

**一句话记忆：依赖项是"可复用的组件"，注入是"FastAPI 自动调用它，把结果塞进函数参数"。**

课件原文：

- **依赖项**：可重用的组件（函数 / 类），负责提供某种功能或数据。
- **注入**：FastAPI 自动帮你调用依赖项，并将结果"注入"到路径操作函数中。

关键点是**你不再调用它，而是声明你需要它**。

### 3. 优点

| 优点 | 说明 |
| --- | --- |
| **代码复用** | 一次编写，多处使用 |
| **解耦** | 业务逻辑与基础设施代码分离 |
| **易于测试** | 轻松地用模拟依赖替换真实依赖进行测试 |

### 4. 应用场景

课件第 11 页给了四类：

| 场景 | 说明 |
| --- | --- |
| **处理请求参数** | 从请求中提取和验证参数（路径参数、查询参数、请求体） |
| **共享业务逻辑** | 抽取封装多个路由公用的逻辑代码 |
| **共享数据库连接** | 管理数据库会话的创建、使用、关闭 |
| **安全和认证** | 验证用户身份、检查权限和角色要求等 |

> 本笔记后半部分的 `get_database` 就是"共享数据库连接"这一条的直接应用。

### 5. 使用三步

**一句话记忆：创建依赖项 → 导入 Depends → 声明依赖项。**

```python
from fastapi import Depends, Query


# ① 创建依赖项
async def common_parameters(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    return {"skip": skip, "limit": limit}


# ② 导入 Depends：见上方 import


# ③ 声明依赖项
@app.get("/news/news_list")
async def get_news_list(commons: dict = Depends(common_parameters)):
    return commons
```

请求 `/news/news_list?skip=5&limit=20` 时，FastAPI 依次做四件事：

```text
解析 skip、limit（含 ge / le 校验）
        ↓
调用 common_parameters，得到 {"skip": 5, "limit": 20}
        ↓
把返回值赋给 commons
        ↓
执行路由函数
```

**理解补充（非课件原结论）**：依赖项还可以是**类**，或依赖另一个依赖项（子依赖），FastAPI 会递归解析，并在同一次请求内复用结果。

### 6. 中间件和依赖注入怎么分工

课件第 8 页的对比很直白：

| | 中间件控制谁？ | 依赖注入控制谁？ |
| --- | --- | --- |
| 范围 | **Everyone** | **我说了算** |
| 选择标准 | 所有接口都要做、与业务无关 | 部分接口要用、返回值要参与业务 |

具体判断：

- 认证、日志、CORS、响应头 → 中间件
- 分页参数、数据库会话、当前登录用户 → 依赖注入

### 7. 课件三问三答

| 问题 | 答案 |
| --- | --- |
| FastAPI 中依赖注入系统有什么用？ | 抽取可复用的组件，实现代码复用、解耦，且可轻松替换依赖项进行测试 |
| 怎么使用依赖注入系统？ | **创建依赖项 → 导入 Depends → 声明依赖项** |
| 和中间件怎么选？ | 中间件管所有人（全局），依赖注入按需生效（精确） |

---

## 四、ORM 与建表

### 1. ORM 是什么

**一句话记忆：ORM 让你用"操作对象"的方式操作数据库，不用手写 SQL。**

课件原文：

> ORM（Object-Relational Mapping，对象关系映射）是一种编程技术，用于在面向对象编程语言和关系型数据库之间建立映射。它允许开发者通过操作对象的方式与数据库进行交互，而无需直接编写复杂的 SQL 语句。

对照一下同一件事的两种写法：

```python
# 手写 SQL：字符串里改个字段名，IDE 不会提示，运行才知道错
# SELECT * FROM book WHERE author = '曹雪芹';

# ORM：字段是类属性，写错立刻报错
result = await db.execute(select(Book).where(Book.author == "曹雪芹"))
```

课件列出的四条优势：

- 减少重复的 SQL 代码
- 代码更简洁易读
- 自动处理数据库连接和事务
- 自动防止 SQL 注入攻击

### 2. 常见 ORM 对比

课件第 15 页：

| 排名 | ORM 工具 | 特点 | 适应场景 |
| --- | --- | --- | --- |
| 1 | **SQLAlchemy ORM** | 功能最强、最灵活、企业级 | 各类 API、微服务、数据应用 |
| 2 | Django ORM | 封装好、上手快 | Django 项目、管理后台 |
| 3 | Tortoise ORM | 全异步 | 异步 Web 服务、高并发 API |

本课程选 **SQLAlchemy**，并配 `aiomysql` 走异步。

### 3. 使用流程总览

课件第 16 页把整章压成三步：

```mermaid
flowchart LR
    S1["① 安装<br/>sqlalchemy[asyncio]<br/>aiomysql"] --> S2["② 建库、建表<br/>run_sync(Base.metadata.create_all)"]
    S2 --> S3["③ 操作数据<br/>查询 / 新增 / 修改 / 删除"]
```

| 阶段 | 内容 |
| --- | --- |
| **1. 安装** | `sqlalchemy[asyncio]`、`aiomysql`（异步数据库驱动） |
| **2. 建库、建表** | `run_sync(Base.metadata.create_all)` |
| **3. 操作数据** | 查询、新增、修改、删除 |

其中"建表"又拆成三块（课件第 18 页）：**创建数据库引擎 → 定义模型类 → 启动应用时建表**。

### 4. 第一步：创建异步引擎

**一句话记忆：引擎负责管连接池，全项目只建一个。**

```python
from sqlalchemy.ext.asyncio import create_async_engine

ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/fastapi_test?charset=utf8"

# 创建异步引擎
async_engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,        # 可选：输出 SQL 日志
    pool_size=10,     # 设置连接池中保持的持久连接数
    max_overflow=20   # 设置连接池允许创建的额外连接数
)
```

| 参数 | 作用 |
| --- | --- |
| `echo=True` | 把实际执行的 SQL 打印到控制台，调试用；上线建议关闭 |
| `pool_size=10` | 连接池里常驻的连接数 |
| `max_overflow=20` | 高峰期允许临时超出的连接数（最多 10 + 20 = 30 条） |

**易错**：URL 必须写 `mysql+aiomysql://`。写成 `mysql://` 或 `mysql+pymysql://` 用的是同步驱动，在异步环境里会出问题。

### 5. 第二步：定义模型类

**一句话记忆：基类放公共字段，模型类对应一张表，属性对应列。**

课件第 20 页的两步：

1. 基类，继承 `DeclarativeBase`（包含通用属性和字段的映射）
2. 定义数据库表对应的模型类

```python
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


# 基类：包含通用属性和字段的映射
class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), default=datetime.now, comment="创建时间")
    update_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now(),
        default=datetime.now, comment="修改时间")


# 模型类：对应数据库中的 book 表
class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    bookname: Mapped[str] = mapped_column(String(255))
    author: Mapped[str] = mapped_column(String(255))
    price: Mapped[int] = mapped_column(comment="价格")
```

逐项理解：

| 写法 | 含义 |
| --- | --- |
| `__tablename__` | 表名，不写会报错 |
| `Mapped[int]` | Python 侧的类型注解 |
| `mapped_column(...)` | 数据库侧的列定义（类型、主键、注释） |
| `String(255)` | 数据库里的 `VARCHAR(255)` |
| `primary_key=True` | 主键，自增整数 |
| `insert_default=func.now()` | 插入时由数据库生成时间 |
| `onupdate=func.now()` | 每次更新自动刷新 `update_time` |

**设计要点**：`create_time` / `update_time` 写在基类里，所有表自动继承，不用每张表重复写。

### 6. 第三步：启动应用时建表

**一句话记忆：`create_all` 要包在 `run_sync` 里，并放在应用启动时执行。**

```python
async def create_tables():
    async with async_engine.begin() as conn:            # 取连接 + 开事务，退出自动提交
        await conn.run_sync(Base.metadata.create_all)  # create_all 是同步函数，需 run_sync 包一层


@app.on_event("startup")
async def startup_event():
    await create_tables()
```

两条重要限制：

- `create_all` **只建不存在的表**，已存在的表不会被修改；改字段结构要用 Alembic 迁移。
- `conn.run_sync(...)` 是在异步环境里调用同步方法的固定写法。

**理解补充（非课件原结论）**：`@app.on_event("startup")` 在新版 FastAPI 中已废弃，等价写法是 `lifespan`。

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    await create_tables()
    yield

app = FastAPI(lifespan=lifespan)
```

### 7. 在路由中使用 ORM：会话依赖项

**一句话记忆：会话依赖项用 `yield` 把 session 借给路由，用完之后统一提交、回滚、关闭。**

课件第 24 页的核心一句：**创建依赖项，使用 `Depends` 注入到处理函数。**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import async_sessionmaker, AsyncSession


# ① 创建异步会话工厂
AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,          # 绑定数据库引擎
    class_=AsyncSession,        # 指定会话类
    expire_on_commit=False      # 会话对象不过期，不重新查询数据库
)


# ② 依赖项：获取数据库会话
async def get_database():
    async with AsyncSessionLocal() as session:
        try:
            yield session                # 返回数据库会话给路由处理函数
            await session.commit()       # 无异常，提交事务
        except Exception:
            await session.rollback()     # 有异常则回滚
            raise
        finally:
            await session.close()        # 关闭会话


# ③ 注入到路由
@app.get("/book/books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book))    # Book 模型类
    books = result.scalars().all()
    return books
```

`yield` 依赖的执行时序：

```text
进入路由前：创建 session
        ↓
yield session（把 session 交给路由函数）
        ↓
路由函数执行：用 db 做增删改查
        ↓
路由返回后：commit → close
  中途异常：rollback → 抛出 → close
```

| 要点 | 说明 |
| --- | --- |
| 用 `yield` 不用 `return` | `yield` 前的代码在路由执行前跑，`yield` 后的代码在路由执行后跑 |
| `expire_on_commit=False` | 提交后对象不过期，响应序列化时不必再查库，避免异步下的 `MissingGreenlet` |
| `try / except / finally` | 异常能回滚，会话一定会关闭 |
| `Depends(get_database)` | 每个请求一个独立 session，互不干扰 |

### 8. 本节三句话

| 问题 | 答案 |
| --- | --- |
| 建表三步是什么？ | `create_async_engine` → 定义 `Base` 与模型类 → `run_sync(Base.metadata.create_all)` |
| 会话怎么给到路由？ | 写 `get_database` 依赖项，`yield session`，路由用 `Depends(get_database)` 接 |
| 为什么 `expire_on_commit=False`？ | 提交后对象不过期，避免序列化时再次查库 |

---

## 五、数据库操作：增删改查

### 1. 四个操作对四个方法

课件第 26 页：

| 操作 | 核心方法 |
| --- | --- |
| 01 查询 | `select()` |
| 02 新增 | `add()` |
| 03 更新 | 先查再改（重新赋值） |
| 04 删除 | `delete()` |

### 2. 查询基础

**一句话记忆：`select()` 负责"查什么"，`db.execute()` 负责"去查"，结果再从 ORM 对象里取。**

```python
# 取全部
@app.get("/book/get_books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book))
    book = result.scalars().all()
    return book


# 取单条：两种写法
@app.get("/book/get_book")
async def get_book(db: AsyncSession = Depends(get_database)):
    # 写法一：取第一条
    # result = await db.execute(select(Book))
    # book = result.scalars().first()

    # 写法二：按主键查（推荐）
    book = await db.get(Book, 1)
    return book
```

### 3. 查询条件

**一句话记忆：`select(Book).where(条件, 条件2, ...)`，多个条件是"与"关系。**

四种条件写法：

- **比较判断**：`==`、`>`、`<`、`>=`、`<=`
- **模糊查询**：`like()`
- **与非查询**：`&`（与）、`|`（或）、`~`（非）
- **包含查询**：`in_()`

#### 比较判断

```python
@app.get("/book/{book_id}")
async def get_book_list(book_id: int, db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.id == book_id))
    book = result.scalar_one_or_none()
    return book
```

#### 模糊查询 `like()`

- `%`：零个、一个或多个字符
- `_`：一个单个字符

```python
@app.get("/book/get_books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.author.like("曹%")))
    book = result.scalars().all()
    return book
```

#### 与非查询 `&` `|` `~`

```python
@app.get("/book/get_books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    result = await db.execute(
        select(Book).where((Book.author == "曹雪芹") & (Book.price == 200))
    )
    book = result.scalars().all()
    return book
```

**易错**：每个条件都要**加括号**。`&` 的优先级高于 `==`，`Book.author == "曹雪芹" & Book.price == 200` 会被解析成很奇怪的表达式，直接报错。

#### 包含查询 `in_()`

```python
@app.get("/book/get_books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    id_list = [1, 2, 3, 4, 5, 6]
    result = await db.execute(select(Book).where(Book.id.in_(id_list)))
    book = result.scalars().all()
    return book
```

### 4. 聚合查询

**一句话记忆：聚合计算写成 `func.方法(模型类.属性)`，结果用 `scalar()` 取。**

| 方法 | 含义 |
| --- | --- |
| `count` | 统计行数量 |
| `avg` | 求平均值 |
| `max` | 求最大值 |
| `min` | 求最小值 |
| `sum` | 求和 |

```python
from sqlalchemy import func

@app.get("/book/count")
async def get_count(db: AsyncSession = Depends(get_database)):
    # result = await db.execute(select(func.count(Book.id)))
    # result = await db.execute(select(func.max(Book.price)))
    # result = await db.execute(select(func.sum(Book.price)))
    result = await db.execute(select(func.avg(Book.price)))
    count = result.scalar()          # 单值，用 scalar() 而不用 scalars()
    return count
```

### 5. 分页查询

**一句话记忆：`offset` 是跳过多少条，`limit` 是本页取多少条，`offset = (页码 - 1) × 每页数量`。**

课件第 35 页的对照表：

| 当前页码 | 每页数量（limit） | 跳过数量（offset） |
| --- | --- | --- |
| 1 | 10 | 0 |
| 2 | 10 | 10 |
| 3 | 10 | 20 |
| 4 | 10 | 30 |

```python
@app.get("/book/get_books")
async def get_book_list(
    page: int = 1,
    page_size: int = 3,
    db: AsyncSession = Depends(get_database)
):
    skip = (page - 1) * page_size
    stmt = select(Book).offset(skip).limit(page_size)
    result = await db.execute(stmt)
    books = result.scalars().all()
    return {"books": books}
```

**理解补充（非课件原结论）**：真实分页一般还要返回总条数，前端才能算总页数。

```python
total = await db.scalar(select(func.count(Book.id)))
return {"total": total, "page": page, "page_size": page_size, "books": books}
```

### 6. 从结果对象取值：四种方式别混

课件第 36、37 页的总结：

```text
核心思路：select() → db.execute() → 从 ORM 对象获取数据 → 响应结果
另有：db.get(模型类, 主键值)
```

| 方法 | 用途 | 查不到时 |
| --- | --- | --- |
| `scalars().all()` | 获取**所有**数据，返回列表 | 返回 `[]` |
| `scalars().first()` | 提取**第一个**数据 | 返回 `None` |
| `scalar_one_or_none()` | 提取**一个**或 `null`（查到多条会报错） | 返回 `None` |
| `scalar()` | 提取**标量值**（配合聚合查询） | 视情况 `None` / `0` |

### 7. 新增

**一句话记忆：造对象 → `add()` 进事务 → `commit()` 落库。**

```python
@app.post("/book/add_book")
async def add_book(book: BookBase, db: AsyncSession = Depends(get_database)):
    # 获取 book 参数，创建图书对象（__dict__ 返回 book 对象的属性字典）
    book_obj = Book(**book.__dict__)
    db.add(book_obj)
    await db.commit()
    return book
```

**理解补充（非课件原结论）**：Pydantic v2 推荐 `book.model_dump()` 代替 `book.__dict__`。另外若 `get_database` 里已经 `commit()`，路由里再提交一次不会出错，但**至少要提交一次**，否则数据不落库。

### 8. 更新

**一句话记忆：先 `get` 查出来，改属性，再 `commit`——不是"发一条 UPDATE 语句"。**

```python
@app.put("/book/update_book/{book_id}")
async def update_book(book_id: int, data: BookUpdate, db: AsyncSession = Depends(get_database)):
    # 1. 查询
    book = await db.get(Book, book_id)
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")

    # 2. 修改属性（重新赋值）
    book.bookname = data.bookname
    book.author = data.author
    book.price = data.price

    # 3. 提交
    await db.commit()
    return book
```

### 9. 删除

**一句话记忆：先 `get` 查出来，`delete()` 标记删除，再 `commit`。**

```python
@app.delete("/book/delete_book/{book_id}")
async def delete_book(book_id: int, db: AsyncSession = Depends(get_database)):
    db_book = await db.get(Book, book_id)
    if db_book is None:
        raise HTTPException(status_code=404, detail="Book not found")

    await db.delete(db_book)
    await db.commit()
    return {"message": "Book deleted"}
```

### 10. 本节小结

| 操作 | 固定套路 |
| --- | --- |
| 查询 | `select()` → `await db.execute()` → `scalars()/scalar()` |
| 新增 | `Book(**data)` → `db.add()` → `commit()` |
| 更新 | `db.get()` → 重新赋值 → `commit()` |
| 删除 | `db.get()` → `db.delete()` → `commit()` |

---

## 六、整章综合示例：把知识点连起来

这个示例是把课件知识点**重新组合**的复习程序，不是课件原有的完整项目。

### 1. 目录结构

```text
fastapi_demo/
├── database.py      # 引擎、会话工厂、get_database 依赖项
├── models.py        # Base 与 Book 模型
├── schemas.py       # Pydantic 请求模型
└── main.py          # 应用、中间件、路由
```

想把全部代码合成一个 `main.py` 也能跑，只是分层后更接近真实项目。

### 2. database.py

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

ASYNC_DATABASE_URL = "mysql+aiomysql://root:123456@localhost:3306/fastapi_test?charset=utf8"

async_engine = create_async_engine(
    ASYNC_DATABASE_URL,
    echo=True,
    pool_size=10,
    max_overflow=20,
)

AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_database():
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

### 3. models.py

```python
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), default=datetime.now, comment="创建时间")
    update_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now(),
        default=datetime.now, comment="修改时间")


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True)
    bookname: Mapped[str] = mapped_column(String(255))
    author: Mapped[str] = mapped_column(String(255))
    price: Mapped[int] = mapped_column(comment="价格")
```

### 4. schemas.py

```python
from pydantic import BaseModel


class BookBase(BaseModel):
    bookname: str
    author: str
    price: int


class BookUpdate(BaseModel):
    bookname: str | None = None
    author: str | None = None
    price: int | None = None
```

### 5. main.py

```python
import time
from contextlib import asynccontextmanager

from fastapi import FastAPI, Depends, HTTPException, Request
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

from database import async_engine, get_database
from models import Base, Book
from schemas import BookBase, BookUpdate


# ---------- 启动时建表 ----------
async def create_tables():
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


@asynccontextmanager
async def lifespan(app: FastAPI):
    await create_tables()
    yield


app = FastAPI(title="FastAPI 第二章复习", lifespan=lifespan)


# ---------- 中间件：统一日志与耗时 ----------
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    cost = (time.perf_counter() - start) * 1000
    response.headers["X-Process-Time"] = f"{cost:.2f}ms"
    print(f"{request.method} {request.url.path} -> {response.status_code}")
    return response


# ---------- 查询：列表 ----------
@app.get("/book/get_books")
async def get_book_list(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book))
    return result.scalars().all()


# ---------- 查询：分页 + 总数 ----------
@app.get("/book/get_books/page")
async def page_books(
    page: int = 1,
    page_size: int = 10,
    db: AsyncSession = Depends(get_database)
):
    skip = (page - 1) * page_size
    result = await db.execute(select(Book).offset(skip).limit(page_size))
    total = await db.scalar(select(func.count(Book.id)))
    return {"total": total, "books": result.scalars().all()}


# ---------- 查询：单条 ----------
@app.get("/book/get_book/{book_id}")
async def get_book(book_id: int, db: AsyncSession = Depends(get_database)):
    book = await db.get(Book, book_id)
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    return book


# ---------- 新增 ----------
@app.post("/book/add_book")
async def add_book(book: BookBase, db: AsyncSession = Depends(get_database)):
    book_obj = Book(**book.model_dump())
    db.add(book_obj)
    await db.commit()
    return book


# ---------- 更新 ----------
@app.put("/book/update_book/{book_id}")
async def update_book(book_id: int, data: BookUpdate,
                      db: AsyncSession = Depends(get_database)):
    book = await db.get(Book, book_id)
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    for field, value in data.model_dump(exclude_none=True).items():
        setattr(book, field, value)
    await db.commit()
    return book


# ---------- 删除 ----------
@app.delete("/book/delete_book/{book_id}")
async def delete_book(book_id: int, db: AsyncSession = Depends(get_database)):
    db_book = await db.get(Book, book_id)
    if db_book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    await db.delete(db_book)
    await db.commit()
    return {"message": "Book deleted"}
```

### 6. 测试清单

启动 `uvicorn main:app --reload`，打开 `http://127.0.0.1:8000/docs` 依次试：

| 测试 | 预期行为 |
| --- | --- |
| `GET /book/get_books` | 返回图书列表（空库为 `[]`） |
| `POST /book/add_book` | 新增成功，控制台打印 SQL |
| `GET /book/get_book/1` | 返回 id 为 1 的书 |
| `GET /book/get_book/999` | 返回 404 |
| `GET /book/get_books/page?page=2&page_size=3` | 返回第 2 页共 3 条与总数 |
| `PUT /book/update_book/1` | 字段被改，`update_time` 自动刷新 |
| `DELETE /book/delete_book/1` | 删除成功，再查返回 404 |
| 看任意响应的 `X-Process-Time` 头 | 中间件生效 |

**注意路由顺序**：`/book/get_books/page` 要写在 `/book/get_book/{book_id}` 之前，否则固定路径会被路径参数抢先匹配。

---

## 七、原课件练习题整理（建议先独立完成）

以下按课件要求整理，答案集中在下一节。

### 练习 1：创建数据表——用户表（课件第 22 页）

使用 SQLAlchemy ORM 创建用户表，字段：用户 id、用户名、密码、创建时间、更新时间，其中**用户 id 为主键**。

### 练习 2：抽取分页依赖项（课件第 7–13 页）

三个接口（新闻列表、用户列表、新闻分类）都有相同的 `skip` / `limit` 参数，要求用依赖注入消除重复，默认值 `skip=0`、`limit=10`，约束 `skip >= 0`、`limit <= 60`。

### 练习 3：耗时中间件（课件第 3、5 页）

写一个中间件，为每个请求统计耗时，并把耗时写入响应头 `X-Process-Time`。

### 练习 4：图书查询条件（课件第 28–33 页）

分别实现：

1. 按 id 查单本书。
2. 查作者姓"曹"的书（模糊查询）。
3. 查作者为"曹雪芹"且价格为 200 的书（与非查询）。
4. 查 id 在 `[1,2,3,4,5,6]` 内的书（包含查询）。

### 练习 5：聚合与分页（课件第 34–35 页）

1. 统计图书总数、平均价格。
2. 实现分页接口，参数 `page`、`page_size`，返回当前页数据与总数。

### 练习 6：图书增删改（课件第 39–43 页）

实现新增、更新、删除三个接口；更新与删除要处理"记录不存在"的情况（返回 404）。

---

## 八、练习参考答案

以下答案默认 `db: AsyncSession = Depends(get_database)` 已按第四章写好。

### 练习 1

```python
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), default=datetime.now, comment="创建时间")
    update_time: Mapped[datetime] = mapped_column(
        DateTime, insert_default=func.now(), onupdate=func.now(),
        default=datetime.now, comment="修改时间")


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True, comment="用户id")
    username: Mapped[str] = mapped_column(String(50), unique=True, comment="用户名")
    password: Mapped[str] = mapped_column(String(255), comment="密码")
```

注意 `create_time` / `update_time` 从基类继承，**不要重复定义**。

### 练习 2

```python
from fastapi import Depends, Query


async def common_parameters(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    return {"skip": skip, "limit": limit}


@app.get("/news/news_list")
async def get_news_list(commons: dict = Depends(common_parameters)):
    return {"params": commons, "list": "新闻列表"}


@app.get("/users/user_list")
async def get_user_list(commons: dict = Depends(common_parameters)):
    return {"params": commons, "users": "用户列表"}


@app.get("/news/category")
async def get_category(commons: dict = Depends(common_parameters)):
    return {"params": commons, "category": "新闻分类"}
```

### 练习 3

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = f"{(time.perf_counter() - start) * 1000:.2f}ms"
    return response
```

### 练习 4

```python
# 1. 按 id 查单本
@app.get("/book/{book_id}")
async def get_book_by_id(book_id: int, db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.id == book_id))
    return result.scalar_one_or_none()


# 2. 作者姓"曹"
@app.get("/book/author/cao")
async def get_cao_books(db: AsyncSession = Depends(get_database)):
    result = await db.execute(select(Book).where(Book.author.like("曹%")))
    return result.scalars().all()


# 3. 曹雪芹 且 200 元
@app.get("/book/author/caoxueqin")
async def get_special_books(db: AsyncSession = Depends(get_database)):
    result = await db.execute(
        select(Book).where((Book.author == "曹雪芹") & (Book.price == 200))
    )
    return result.scalars().all()


# 4. id 在指定列表内
@app.get("/book/ids")
async def get_books_by_ids(db: AsyncSession = Depends(get_database)):
    id_list = [1, 2, 3, 4, 5, 6]
    result = await db.execute(select(Book).where(Book.id.in_(id_list)))
    return result.scalars().all()
```

### 练习 5

```python
from sqlalchemy import func

# 1. 聚合
@app.get("/book/count")
async def get_count(db: AsyncSession = Depends(get_database)):
    total = await db.scalar(select(func.count(Book.id)))
    avg_price = await db.scalar(select(func.avg(Book.price)))
    return {"total": total, "avg_price": float(avg_price or 0)}


# 2. 分页
@app.get("/book/get_books/page")
async def page_books(
    page: int = 1,
    page_size: int = 3,
    db: AsyncSession = Depends(get_database)
):
    skip = (page - 1) * page_size
    result = await db.execute(select(Book).offset(skip).limit(page_size))
    total = await db.scalar(select(func.count(Book.id)))
    return {"total": total, "page": page, "page_size": page_size,
            "books": result.scalars().all()}
```

### 练习 6

```python
@app.post("/book/add_book")
async def add_book(book: BookBase, db: AsyncSession = Depends(get_database)):
    book_obj = Book(**book.model_dump())
    db.add(book_obj)
    await db.commit()
    return book


@app.put("/book/update_book/{book_id}")
async def update_book(book_id: int, data: BookUpdate,
                      db: AsyncSession = Depends(get_database)):
    book = await db.get(Book, book_id)
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    book.bookname = data.bookname
    book.author = data.author
    book.price = data.price
    await db.commit()
    return book


@app.delete("/book/delete_book/{book_id}")
async def delete_book(book_id: int, db: AsyncSession = Depends(get_database)):
    db_book = await db.get(Book, book_id)
    if db_book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    await db.delete(db_book)
    await db.commit()
    return {"message": "Book deleted"}
```

---

## 九、最容易混淆的知识点

| 容易混淆 | 正确认识 |
| --- | --- |
| 中间件执行顺序 | 后定义的先执行，请求**自下而上**，响应反过来 |
| `await call_next(request)` | 这是"把请求交给下游"，不是"发起一个新请求"，漏写 `await` 接口必报错 |
| 中间件改响应 | 改 `headers` 可直接赋值；改**响应体**要重新构造 `Response(...)` |
| 中间件 vs 依赖注入 | 前者管所有请求，后者只对声明了 `Depends` 的接口生效 |
| `yield` vs `return`（依赖项） | `yield` 后的代码在路由执行完才跑，`return` 就直接结束了 |
| `commit` 位置 | 依赖项里提交，或路由里提交，至少要提交一次，否则不落库 |
| `expire_on_commit=False` | 防止提交后对象过期、序列化时再查库（异步下报 `MissingGreenlet`） |
| `mysql://` vs `mysql+aiomysql://` | 异步必须用带驱动的 URL |
| `where(a & b)` | 条件必须加括号：`where((a) & (b))` |
| `scalars()` vs `scalar()` | 聚合结果是单值，用 `scalar()`；实体对象用 `scalars()` |
| `scalars().first()` vs `scalar_one_or_none()` | 前者多条取第一条，后者多条直接报错 |
| `create_all` | 只建不存在的表，不修改已有表结构 |
| `db.add()` 后不 `commit()` | 只在事务里，进程结束数据就没了 |

### 课件示例中的整理说明

1. **第 12 页依赖注入示例**：装饰器写作 `@app.get("news/news_list'")`——路径少了开头的 `/`，且引号不匹配。本笔记统一改为 `@app.get("/news/news_list")`。
2. **第 20 页模型类示例**：课件用 `......` 省略了后续字段，但第 31–35 页的查询示例用到了 `Book.price`。本笔记在模型里补上 `price` 字段，使前后示例可以真正跑通。
3. **第 21 页启动建表**：课件使用 `@app.on_event("startup")`，该写法在新版 FastAPI 中已废弃。本笔记在正文保留课件写法以便对照，并在补充里给出 `lifespan` 版本。
4. **第 39 页新增示例**：`Book(**book.__dict__)` 在 Pydantic v2 下更推荐写 `Book(**book.model_dump())`。

以上四点属于**可运行性与写法一致性修正**，不改变原课件知识点。

---

## 十、闭卷自测：10 分钟检验掌握情况

尝试不看前文回答：

1. 中间件的两个运行时机分别是什么？
2. `@app.middleware("http")` 修饰的函数签名是什么？`call_next` 的作用？
3. 三个中间件 mw1、mw2、mw3 按此顺序定义，请求阶段的执行顺序是？
4. 中间件和依赖注入分别适合什么场景？各举两例。
5. 依赖注入的三步是什么？
6. `get_database` 里为什么用 `yield`？`try / except / finally` 各做什么？
7. `expire_on_commit=False` 解决什么问题？
8. 建表的三步是什么？`create_all` 能否修改已有表结构？
9. `select`、`db.execute`、`scalars()` 三者在一次查询里各负责什么？
10. 分页的 `offset` 怎么算？第 5 页、每页 20 条时 `offset` 是多少？
11. 新增、更新、删除各自的固定套路是什么？
12. 数据库 URL 为什么必须写 `mysql+aiomysql://`？

**达标标准**：能不看笔记写出"引擎 + 模型 + 会话依赖项 + 五个 CRUD 接口 + 一个耗时中间件"的完整程序。

**参考答案**（先自己回答，再对照）：

1. 请求到达路由处理函数之前一次；响应返回客户端之前再一次。  
2. `async def middleware(request, call_next)`；`call_next` 把请求交给后续中间件或路由，返回 `Response`。  
3. mw3 → mw2 → mw1（后定义最外层）；响应顺序相反。  
4. 全局且与业务无关 → 中间件（认证、日志、CORS）；部分接口且需要返回值 → 依赖注入（分页参数、数据库会话）。  
5. 创建依赖项 → 导入 `Depends` → 声明依赖项。  
6. 需要"借用"资源给路由用完再归还：`yield` 前创建 session，后提交；`try` 包住 `yield`，`except` 回滚并 `raise`，`finally` 关闭。  
7. 提交后对象不过期，序列化时不必再查库，避免 `MissingGreenlet`。  
8. `create_async_engine` → 定义 `Base` 与模型类 → `run_sync(Base.metadata.create_all)`；不能修改已有表结构。  
9. `select()` 描述查什么，`db.execute()` 真正执行，`scalars()` 从结果里取实体对象。  
10. `(页码 - 1) × 每页数量`；(5-1)×20 = **80**。  
11. 新增 `add` → `commit`；更新 `get` → 赋值 → `commit`；删除 `get` → `delete` → `commit`。  
12. 因为 `aiomysql` 是异步驱动，同步驱动无法在异步事件循环中正常使用。

---

## 十一、一页速查

### 常用命令

```bash
python -m pip install "sqlalchemy[asyncio]" aiomysql
python -m uvicorn main:app --reload
```

```sql
-- 建库（建表交给 create_all）
CREATE DATABASE fastapi_test DEFAULT CHARSET utf8mb4;
```

### 中间件模板

```python
@app.middleware("http")
async def my_middleware(request, call_next):
    # 请求阶段
    response = await call_next(request)
    # 响应阶段
    response.headers["X-Demo"] = "1"
    return response
```

### 依赖注入模板

```python
async def common_parameters(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, le=60)
):
    return {"skip": skip, "limit": limit}


@app.get("/items")
async def list_items(commons: dict = Depends(common_parameters)):
    return commons
```

### ORM 模板

```python
# 引擎
async_engine = create_async_engine(ASYNC_DATABASE_URL, echo=True)

# 模型
class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)

class Book(Base):
    __tablename__ = "book"
    id: Mapped[int] = mapped_column(primary_key=True)
    bookname: Mapped[str] = mapped_column(String(255))

# 会话依赖项
AsyncSessionLocal = async_sessionmaker(bind=async_engine, class_=AsyncSession,
                                       expire_on_commit=False)

async def get_database():
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

### 核心知识结构

```text
FastAPI 进阶
├── 中间件
│   ├── @app.middleware("http")
│   ├── request 阶段 / response 阶段
│   └── 多中间件：请求自下而上
├── 依赖注入
│   ├── 创建依赖项
│   ├── Depends 注入
│   └── 适用：参数、业务、数据库会话、认证
└── ORM（SQLAlchemy 异步）
    ├── 安装：sqlalchemy[asyncio] + aiomysql
    ├── 建表
    │   ├── create_async_engine
    │   ├── DeclarativeBase + Mapped + mapped_column
    │   └── run_sync(Base.metadata.create_all)
    └── 操作
        ├── 查：select / where / func / offset+limit
        ├── 增：add
        ├── 改：get → 赋值
        └── 删：delete
```

### 复习顺序建议

- **第 1 遍（理解）**：看一至五节，弄清"中间件管全局、依赖主管局部、ORM 管数据"。
- **第 2 遍（练手）**：自己写第六节的分层示例，不要复制粘贴。
- **第 3 遍（检测）**：完成第七节练习和第十节闭卷自测。
- **以后快速回顾**：优先看第五节操作小结、第九节易混点、第十一节速查。

> **本章边界**：课件围绕中间件、依赖注入与 ORM 建表/增删改查，并没有系统讲解表关联（外键、`relationship`）、事务隔离级别、Alembic 迁移、连接池调优、分页性能与索引、多表联查（`join`）。这些内容后续单独整理，不要与本章混在一起。
