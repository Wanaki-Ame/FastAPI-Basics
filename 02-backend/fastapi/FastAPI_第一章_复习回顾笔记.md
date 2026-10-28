# FastAPI 第一章｜基础入门与请求处理

> **定位**：学完后用于快速复习、回忆核心概念、照着代码恢复手感，而不是逐页重读 PPT。  
> **资料依据**：《第一章_FastAPI入门.pdf》（64 页，黑马程序员《FastAPI 基础到实战》）。  
> **覆盖范围**：FastAPI 特点、同步与异步、环境搭建、路由、路径参数、查询参数、请求体、Pydantic、响应类型、`response_model`、`HTTPException`。  
> **阅读方法**：先看每节的「一句话记忆」，合上笔记说出原理，再看代码，最后做文末练习。

[TOC]

---

## 一、先建立整体认识

### 1. FastAPI 是什么？

**一句话记忆：FastAPI 是使用 Python 开发 Web API 的框架，负责接收 HTTP 请求、解析与校验数据、调用处理函数，再返回 HTTP 响应。**

课件强调的三个特点：

1. **支持异步**：适合处理存在网络、数据库等 I/O 等待的请求。
2. **基于类型注解校验数据**：通过 Python 类型提示以及 Pydantic 模型约束输入。
3. **自动生成交互式 API 文档**：在浏览器里查看、调用和调试接口。

记住一次请求的基本流向：

```text
客户端（浏览器/前端）
       │ 发起 HTTP 请求
       ▼
FastAPI 根据请求方法 + URL 找到路由
       │
       ▼
解析请求参数、执行类型校验
       │
       ▼
调用对应的 Python 函数处理业务
       │
       ▼
生成响应（通常是 JSON）
       │
       ▼
客户端接收结果
```

例如前端查询一本书：

```text
GET /book/123
      ↓
FastAPI 找到 /book/{id} 对应的函数
      ↓
解析 id=123，并检查是否符合 int 类型
      ↓
函数查询或生成图书信息
      ↓
返回 JSON 数据
```

### 2. 同步与异步：到底在区别什么？

**一句话记忆：同步函数按通常方式执行；异步函数可以在等待异步 I/O 时把执行机会交给其他任务。**

课件第 6～8 页通过「等待 10 次，每次 1 秒」说明区别：

```python
import asyncio
import time

# 顺序执行：总等待时间大约 10 秒
def sync_example():
    for _ in range(10):
        time.sleep(1)

# 协作式并发：10 个等待操作可以重叠，总等待时间大约 1 秒
async def async_example():
    tasks = [asyncio.sleep(1) for _ in range(10)]
    await asyncio.gather(*tasks)
```

| 对比点 | `def` | `async def` |
|---|---|---|
| 写法 | 普通函数 | 协程函数 |
| 等待方式 | 常见同步调用会阻塞当前执行线程 | 遇到可等待的异步操作时可让出执行机会 |
| 典型调用 | 同步库、普通计算 | 支持 `await` 的异步网络/数据库操作 |
| 常见误区 | “同步接口一定串行处理所有请求” | “只要加 `async` 就一定更快” |

**理解补充（非课件原结论）**：

- `async def` 不会自动把 CPU 密集型计算变快。
- 不应在 `async def` 中直接使用 `time.sleep()` 模拟异步等待；应使用 `await asyncio.sleep()`。
- FastAPI 可以同时使用 `def` 和 `async def`；选择时主要看所调用的依赖库是否支持异步。
- 上述 10 秒与 1 秒仅是课件的**等待任务演示**，不是所有真实 Web API 的固定性能差距。

**自问**：为什么一个接口写了 `async def`，调用阻塞型数据库驱动后，仍可能影响并发？

---

## 二、创建、运行与访问项目

### 1. 虚拟环境的作用

**一句话记忆：每个项目使用独立的 Python 包环境，避免项目之间发生依赖版本冲突。**

课件第 14～17 页介绍了项目创建、虚拟环境、运行与文档访问。以下终端命令是对该操作流程的可执行整理。

在项目目录中创建环境：

```bash
python -m venv .venv
```

激活虚拟环境：

| 系统 / 终端 | 命令 |
|---|---|
| Windows PowerShell | `.\.venv\Scripts\Activate.ps1` |
| Windows CMD | `.venv\Scripts\activate.bat` |
| Linux / macOS | `source .venv/bin/activate` |

安装依赖：

```bash
python -m pip install fastapi uvicorn
```

退出虚拟环境：

```bash
deactivate
```

### 2. 第一个 FastAPI 程序

新建 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

逐行理解：

| 代码 | 含义 |
|---|---|
| `from fastapi import FastAPI` | 导入框架类 |
| `app = FastAPI()` | 创建 FastAPI 应用实例 |
| `@app.get("/")` | 注册处理根路径的 GET 路由 |
| `async def root():` | 定义处理请求的函数 |
| `return {...}` | 返回可自动转换为 JSON 的数据 |

### 3. 启动服务

```bash
uvicorn main:app --reload
```

命令拆解：

- `uvicorn`：运行 ASGI Web 应用的服务器。
- `main`：Python 文件名 `main.py`，**不带 `.py`**。
- `app`：文件中的 `app = FastAPI()` 对象。
- `--reload`：检测代码变更并重新加载，**用于开发调试**。

默认访问：

| 地址 | 用途 |
|---|---|
| `http://127.0.0.1:8000/` | 查看根路由返回内容 |
| `http://127.0.0.1:8000/docs` | Swagger UI 交互式文档 |

**常见错误**：如果文件叫 `api.py`，而实例仍叫 `app`，启动命令就应该是 `uvicorn api:app --reload`。

---

## 三、路由：URL 如何对应到函数？

### 1. 什么是路由？

**一句话记忆：路由就是请求方法、URL 路径和处理函数之间的映射。**

```python
@app.get("/user/hello")
async def hello():
    return {"msg": "我正在学习 FastAPI ......"}
```

访问：

```text
GET http://127.0.0.1:8000/user/hello
```

返回：

```json
{"msg": "我正在学习 FastAPI ......"}
```

其中：

- `@app.get(...)` 是 Python **装饰器**，用来注册 GET 接口。
- `"/user/hello"` 是请求路径。
- `hello` 是实际处理请求的函数。
- `return` 定义要返回的数据。

### 2. 为什么还要学习参数？

如果查询不同的书都单独写一个路由，代码就会重复：

```text
/book/1 → 第 1 本书
/book/2 → 第 2 本书
/book/3 → 第 3 本书
```

更好的方式是使用**同一个路由 + 不同参数**：

```python
@app.get("/book/{id}")
async def get_book(id: int):
    return {"id": id, "title": f"这是第{id}本书"}
```

这样一个函数就能处理不同的图书编号。

---

## 四、三种请求参数：必须重点掌握

课件第 26 页将参数分为 **路径参数、查询参数、请求体参数**。核心不是死记函数名，而是先判断：**参数放在 HTTP 请求的什么位置？**

| 类型 | 位置 | URL / 数据示例 | 常见用途 | FastAPI 中的常见写法 |
|---|---|---|---|---|
| 路径参数 Path | URL 路径内部 | `/book/123` | 指定某个资源 | `Path(...)` |
| 查询参数 Query | URL 中 `?` 后 | `/books?skip=0&limit=10` | 筛选、搜索、分页 | `Query(...)` |
| 请求体 Body | HTTP 请求消息体 | JSON `{ "title": "Python" }` | 提交较复杂数据 | `BaseModel` + `Field(...)` |

> **速记**：Path 定位资源，Query 描述查询条件，Body 携带提交的数据。  
> 以上是典型用途，并不是说某一种 HTTP 方法只能使用某一种参数。

### 1. 路径参数 Path

**一句话记忆：URL 中 `{变量名}` 对应函数中的参数；类型注解可以自动校验它。**

基础示例：

```python
@app.get("/book/{id}")
async def get_book(id: int):
    return {"id": id, "title": f"这是第{id}本书"}
```

```text
GET /book/123    → id = 123
GET /book/abc    → 类型校验不通过
```

#### 使用 `Path()` 进一步限制输入

```python
from fastapi import Path

@app.get("/category/id/{category_id}")
async def get_category(
    category_id: int = Path(..., ge=1, le=100, description="新闻分类编号")
):
    return {"category_id": category_id}
```

含义：分类编号是整数，取值范围 **1～100（包含端点）**。

限制字符串长度：

```python
@app.get("/category/name/{category_name}")
async def get_category_by_name(
    category_name: str = Path(..., min_length=2, max_length=10)
):
    return {"category_name": category_name}
```

**`Path` 参数速查表**

| 写法 | 含义 | 举例 |
|---|---|---|
| `...` | 必填值标记 | `Path(...)` |
| `gt=0` | 大于 0 | 不能等于 0 |
| `ge=1` | 大于等于 1 | 可以等于 1 |
| `lt=100` | 小于 100 | 不能等于 100 |
| `le=100` | 小于等于 100 | 可以等于 100 |
| `min_length=2` | 字符串最短长度 2 | 适用于字符串参数 |
| `max_length=10` | 字符串最长长度 10 | 适用于字符串参数 |
| `description="..."` | 参数说明 | 显示在 API 文档中 |

注意：路径中定义的参数本身必须出现在 URL 里，不能通过给它设置默认值来省略路径片段。

### 2. 查询参数 Query

**一句话记忆：不属于路由路径、且是合适的简单标量参数，FastAPI 通常会将其解析为查询参数。**

基础分页示例（课件第 33 页）：

```python
@app.get("/news/news_list")
async def get_news_list(skip: int, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

访问：

```text
GET /news/news_list?skip=0&limit=10
```

理解：

- `?`：查询参数开始。
- `&`：连接多个查询参数。
- `skip`：没有默认值，因此必须传入。
- `limit=10`：有默认值，可以不传。

#### 使用 `Query()` 限制参数

课件第 36 页要求查询图书，参数约束为：

- 分类默认是 `Python开发`，长度 5～255。
- 价格限制在 50～100。

```python
from fastapi import Query

@app.get("/books/search")
async def search_books(
    category: str = Query("Python开发", min_length=5, max_length=255),
    price: float = Query(..., ge=50, le=100),
):
    return {"category": category, "price": price}
```

请求示例：

```text
GET /books/search?price=79
GET /books/search?category=Java开发&price=80
```

注意 `Query` 也支持 `gt`、`ge`、`lt`、`le`、`min_length`、`max_length`、`description` 等约束。

**容易混淆**：

- `Query(...)`：要求客户端提供这个参数。
- `Query(10)`：可以省略，省略后使用默认值 10。
- `price: float`：只声明浮点数类型；`Query(..., ge=50, le=100)` 还附加了范围约束。

### 3. 请求体参数 Body 与 Pydantic

**一句话记忆：提交 JSON 等结构化数据时，先定义 `BaseModel` 数据模型，再把它作为处理函数的参数类型。**

HTTP 请求主要包括：请求行、请求头、请求体（课件第 37 页）。请求体通常放要提交的数据。

基础示例：

```python
from pydantic import BaseModel

class User(BaseModel):
    username: str
    password: str

@app.post("/register")
async def register(user: User):
    return {"username": user.username}
```

客户端发送 JSON：

```json
{
  "username": "alice",
  "password": "123456"
}
```

FastAPI 会根据 `User` 模型解析请求体，并校验字段是否满足声明的类型要求。

**实践补充**：课件演示直接 `return user` 以便观察请求体。实际注册接口通常不应把用户密码原样返回，也不能明文保存密码；这里的示例只返回用户名。

#### 使用 `Field()` 进一步限定模型字段

课件第 42 页的图书创建练习：

```python
from pydantic import BaseModel, Field

class BookCreate(BaseModel):
    title: str = Field(..., min_length=2, max_length=20)
    author: str = Field(..., min_length=2, max_length=10)
    publisher: str = Field(default="黑马出版社")
    price: float = Field(..., gt=0)

@app.post("/books")
async def create_book(book: BookCreate):
    return {"message": "创建成功", "book": book}
```

示例请求体：

```json
{
  "title": "Python编程入门",
  "author": "张三",
  "price": 59.9
}
```

因为 `publisher` 提供了默认值，可以省略。

**`Field` 参数速查表**

| 写法 | 作用 |
|---|---|
| `Field(...)` | 必填 |
| `Field(default="...")` | 提供默认值 |
| `Field(gt=0)` | 大于 0 |
| `Field(ge=0)` | 大于等于 0 |
| `Field(lt=100)` | 小于 100 |
| `Field(le=100)` | 小于等于 100 |
| `Field(min_length=2)` | 最小长度 |
| `Field(max_length=20)` | 最大长度 |
| `Field(description="...")` | 字段描述 |

### 4. `Path`、`Query`、`Field` 的关系

这三个工具**都是在声明输入的要求**，但应用位置不同：

```text
HTTP 请求
├── URL 路径：/book/123               → Path
├── URL 查询：?skip=0&limit=10         → Query
└── 请求体：{"title":"Python"}       → BaseModel / Field
```

遇到代码时，按下面的顺序判断：

1. 装饰器路径中有 `{id}` 吗？有，则函数对应参数是路径参数。
2. 参数是简单查询条件吗？通常作为 Query。
3. 参数是一个 `BaseModel` 数据对象吗？通常从请求体读取。
4. 是否需要范围、长度、默认值和字段说明？再使用 `Path`、`Query`、`Field`。

---

## 五、响应：接口怎样把结果交给客户端？

### 1. 默认返回 JSON

**一句话记忆：一般返回 Python 字典或列表即可，FastAPI 会负责转换为 JSON 响应。**

```python
@app.get("/hello")
async def hello():
    return {"message": "Hello World"}
```

这里不需要每次手动调用 `json.dumps()`。

### 2. 常见响应类型

课件第 44～53、59～62 页重点介绍下表：

| 响应类 | 返回内容 | 常见场景 |
|---|---|---|
| `JSONResponse` | JSON | 普通 API 数据 |
| `HTMLResponse` | HTML | 返回简单网页 |
| `PlainTextResponse` | 纯文本 | 返回文字 |
| `FileResponse` | 文件内容 | 发送图片、PDF 等文件 |
| `StreamingResponse` | 分段输出 | 流式数据 |
| `RedirectResponse` | 重定向 | 跳转到另一个 URL |

对应导入位置：

```python
from fastapi.responses import (
    JSONResponse,
    HTMLResponse,
    PlainTextResponse,
    FileResponse,
    StreamingResponse,
    RedirectResponse,
)
```

### 3. 设置响应类型的两种方式

**方式 A：在装饰器中指定 `response_class`。**

```python
from fastapi.responses import HTMLResponse

@app.get("/html", response_class=HTMLResponse)
async def get_html():
    return "<h1>Hello World</h1>"
```

**方式 B：在函数中直接返回一个响应对象。**

```python
from fastapi.responses import FileResponse

@app.get("/file")
async def get_file():
    file_path = "./files/1.jpeg"
    return FileResponse(file_path)
```

上面的文件接口要求项目中实际存在 `files/1.jpeg`；否则无法正常返回文件。

**区别记忆**：

- 固定响应类型：通常写 `response_class=...`。
- 明确构造文件、重定向或流式响应对象：通常直接 `return 某个Response(...)`。

### 4. 使用 `response_model` 约束返回数据

**一句话记忆：`response_model` 定义返回给客户端的数据结构，而不是客户端传入的数据结构。**

课件第 54～60 页以新闻信息为例：

```python
from pydantic import BaseModel

class NewsOut(BaseModel):
    id: int
    title: str
    content: str

@app.get("/news/{id}", response_model=NewsOut)
async def get_news(id: int):
    return {
        "id": id,
        "title": f"第{id}条新闻",
        "content": "新闻正文",
        "internal_note": "仅服务器内部使用",
    }
```

客户端最终看到的是符合 `NewsOut` 模型的字段：

```json
{
  "id": 1,
  "title": "第1条新闻",
  "content": "新闻正文"
}
```

上述 `internal_note` 不在模型中，普通 Pydantic 输出模型会过滤它。它体现了返回数据的**约束与筛选**。

**两个参数别混淆：**

| 参数 | 控制什么 | 示例 |
|---|---|---|
| `response_class` | HTTP 响应类型 | `HTMLResponse` |
| `response_model` | 输出数据的字段与类型 | `NewsOut` |

注意：`response_model` 是常见 API 输出约束手段，但不能代替完整的权限和数据安全设计。

---

## 六、异常处理：什么时候返回 404？

### 1. `HTTPException` 的作用

**一句话记忆：业务条件不满足时，用 `raise HTTPException(...)` 主动停止正常流程并返回指定 HTTP 错误。**

课件第 63 页的资源不存在示例：

```python
from fastapi import HTTPException

@app.get("/news/{id}")
async def get_news(id: int):
    id_list = [1, 2, 3, 4, 5, 6]
    if id not in id_list:
        raise HTTPException(status_code=404, detail="当前id不存在")
    return {"id": id}
```

请求：

```text
GET /news/100
```

响应状态码：`404`；响应内容：

```json
{"detail": "当前id不存在"}
```

### 2. 404 和参数校验错误的区别

| 情况 | 含义 | 常见结果 |
|---|---|---|
| 访问不存在的路由 | 没有匹配的路径 | `404 Not Found` |
| 合法编号但资源不存在 | 业务查询找不到数据，主动抛出 | `404 Not Found` |
| `id` 应为整数，却传入字母 | 请求输入不能通过类型校验 | `422 Unprocessable Entity` |
| 图书价格要求大于 0，却传入 -1 | 请求输入不能通过字段校验 | `422 Unprocessable Entity` |

> **理解补充**：422 是 FastAPI 输入校验失败时常见的默认响应，课件重点演示的是主动抛出 `HTTPException(404)`。不要把“参数非法”和“资源不存在”混成一种错误。

---

## 七、整章综合示例：把知识点连起来

这个示例是按课件知识点**重新组合的复习程序**，不是课件原有的完整项目。它不连接数据库，也不持久化数据。

把下面代码保存为 `main.py`：

```python
from fastapi import FastAPI, Path, Query, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel, Field

app = FastAPI(title="FastAPI 第一章复习")

# 1. 路由
@app.get("/")
def home():
    return {"message": "Hello FastAPI"}


@app.get("/user/hello")
def user_hello():
    return {"msg": "我正在学习 FastAPI ......"}


# 2. 路径参数
@app.get("/users/{user_id}")
def get_user(user_id: int = Path(..., gt=0)):
    return {"id": user_id, "name": f"普通用户{user_id}"}


# 3. 查询参数
@app.get("/books/search")
def search_books(
    category: str = Query("Python开发", min_length=5, max_length=255),
    price: float = Query(..., ge=50, le=100),
):
    return {"category": category, "price": price}


# 4. 请求体模型 + 字段验证
class BookCreate(BaseModel):
    title: str = Field(..., min_length=2, max_length=20)
    author: str = Field(..., min_length=2, max_length=10)
    publisher: str = Field(default="黑马出版社")
    price: float = Field(..., gt=0)


@app.post("/books")
def create_book(book: BookCreate):
    return {"message": "创建成功", "book": book}


# 5. 输出模型 + 404 异常
class NewsOut(BaseModel):
    id: int
    title: str
    content: str


@app.get("/news/{news_id}", response_model=NewsOut)
def get_news(news_id: int = Path(..., gt=0)):
    if news_id not in [1, 2, 3]:
        raise HTTPException(status_code=404, detail="当前id不存在")
    return {
        "id": news_id,
        "title": f"第{news_id}条新闻",
        "content": "新闻正文",
        "internal_note": "不要返回给客户端",
    }


# 6. HTML 响应
@app.get("/html", response_class=HTMLResponse)
def get_html():
    return "<h1>Hello FastAPI</h1>"
```

运行：

```bash
uvicorn main:app --reload
```

然后访问 `http://127.0.0.1:8000/docs`，依次尝试：

| 测试 | 预期行为 |
|---|---|
| `GET /` | 返回 Hello FastAPI |
| `GET /users/123` | 返回用户编号和名称 |
| `GET /users/abc` | 发生类型校验错误 |
| `GET /books/search?price=80` | 使用默认分类 |
| `GET /books/search?price=120` | 价格范围校验不通过 |
| `POST /books` 并填写 JSON | 验证 `BookCreate` 模型 |
| `GET /news/2` | 返回 `NewsOut` 定义的 3 个字段 |
| `GET /news/99` | 返回 404 |
| `GET /html` | 浏览器显示 HTML 标题 |

**复习目标**：不看代码，尝试自己重新实现以上 6 类接口。

---

## 八、原课件练习题整理（建议先独立完成）

以下保留课件原练习需求；解答集中放在下一节，避免看题时直接看到答案。

### 练习 1：基础路由（课件第 22 页）

访问 `/user/hello`，返回：

```json
{"msg": "我正在学习 FastAPI ......"}
```

### 练习 2：路径参数（课件第 28 页）

访问 `/user/123`，返回：

```json
{"id": 123, "name": "普通用户123"}
```

### 练习 3：路径参数约束（课件第 32 页）

设计两个接口：

1. 新闻分类 `id` 范围 1～100。
2. 新闻分类名称长度 2～10。

要求使用 `Path`。

### 练习 4：查询参数（课件第 36 页）

查询图书：

- 分类默认 `Python开发`，长度限制 5～255。
- 价格范围 50～100。

要求使用 `Query`。

### 练习 5：请求体（课件第 39 页）

新增图书，请求体包含书名、作者、出版社、售价。

### 练习 6：请求体约束（课件第 42 页）

在练习 5 的基础上增加：

- 书名必填，长度 2～20。
- 作者长度 2～10。
- 出版社默认 `黑马出版社`。
- 售价必填，大于 0。

要求使用 `BaseModel` + `Field`。

---

## 九、练习参考答案

以下答案按课件要求重新整理，默认文件开头已写：

```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel, Field

app = FastAPI()
```

### 练习 1、2

```python
@app.get("/user/hello")
def user_hello():
    return {"msg": "我正在学习 FastAPI ......"}


@app.get("/user/{user_id}")
def get_user(user_id: int):
    return {"id": user_id, "name": f"普通用户{user_id}"}
```

注意固定路径 `/user/hello` 应注册在动态路径 `/user/{user_id}` 前面，避免动态路由抢先匹配导致校验错误。

### 练习 3

```python
@app.get("/category/id/{category_id}")
def category_by_id(category_id: int = Path(..., ge=1, le=100)):
    return {"id": category_id}


@app.get("/category/name/{category_name}")
def category_by_name(
    category_name: str = Path(..., min_length=2, max_length=10)
):
    return {"name": category_name}
```

### 练习 4

```python
@app.get("/books/search")
def search_books(
    category: str = Query("Python开发", min_length=5, max_length=255),
    price: float = Query(..., ge=50, le=100),
):
    return {"category": category, "price": price}
```

### 练习 5

```python
class Book(BaseModel):
    title: str
    author: str
    publisher: str
    price: float


@app.post("/books")
def create_book(book: Book):
    return book
```

### 练习 6

```python
class BookValidated(BaseModel):
    title: str = Field(..., min_length=2, max_length=20)
    author: str = Field(..., min_length=2, max_length=10)
    publisher: str = Field(default="黑马出版社")
    price: float = Field(..., gt=0)


@app.post("/books/validated")
def create_validated_book(book: BookValidated):
    return book
```

---

## 十、最容易混淆的 8 个知识点

| 容易混淆 | 正确认识 |
|---|---|
| `main:app` | 前者是模块，后者是 FastAPI 实例变量 |
| `@app.get` 和 `@app.post` | 两者都是注册路由，HTTP 方法不同 |
| `Path()` 和 `Query()` | 前者对应 URL 路径部分，后者对应查询字符串 |
| `Query(...)` 和 `Query(10)` | 前者必填，后者有默认值 |
| `BaseModel` 和 `Field` | 前者定义数据结构，后者约束具体字段 |
| `response_class` 和 `response_model` | 前者控制响应类型，后者控制输出数据结构 |
| `return` 和 `raise HTTPException` | 前者正常返回，后者中断并产生错误响应 |
| `async def` 和真正异步 I/O | 写异步函数不等于内部所有操作都会非阻塞 |

### 课件示例中的两处整理说明

1. **第 34～35 页 `Query` 示例**：函数参数写的是 `user_id`，但返回字典引用了 `id`。本笔记没有照抄这一处变量混用，统一使用正确的参数名。
2. **第 54～60 页响应模型示例**：路径是 `news`，返回内容却部分使用“这是第几本书”的图书描述。本笔记保留 `response_model` 的教学意图，将示例文案统一为新闻数据。

这两处属于**示例可读性与变量一致性修正**，不改变原课件知识点。

---

## 十一、闭卷自测：10 分钟检验掌握情况

尝试不看前文回答：

1. FastAPI 的主要用途是什么？
2. `uvicorn main:app --reload` 中的 `main`、`app`、`--reload` 分别指什么？
3. 为什么每个 Python 项目建议使用 `.venv`？
4. 什么是路由？解释 `@app.get("/book/{id}")`。
5. `/book/3` 和 `/books?price=80` 分别属于哪类参数？
6. `Path(..., ge=1, le=100)` 限制了什么？
7. `Query(10)` 和 `Query(...)` 的区别是什么？
8. `BaseModel` 和 `Field` 如何配合使用？
9. 返回字典、返回 HTML、返回文件，分别有什么常用写法？
10. `response_model` 和 `response_class` 有什么区别？什么场景使用 `HTTPException(404)`？

**达标标准**：能够口头回答上述问题，并在不看答案的情况下写出带有 Path、Query、Body、响应模型与 404 错误的简易接口。

---

## 十二、一页速查：忘了就看这里

### 常用命令

```bash
python -m venv .venv
python -m pip install fastapi uvicorn
uvicorn main:app --reload
```

### 基础框架模板

```python
from fastapi import FastAPI, Path, Query, HTTPException
from pydantic import BaseModel, Field

app = FastAPI()

# Path: URL /items/1
@app.get("/items/{item_id}")
def get_item(item_id: int = Path(..., gt=0)):
    return {"id": item_id}

# Query: URL /search?keyword=Python
@app.get("/search")
def search(keyword: str = Query(..., min_length=1)):
    return {"keyword": keyword}

# Body: POST /items，附带 JSON
class ItemIn(BaseModel):
    name: str = Field(..., min_length=2)
    price: float = Field(..., gt=0)

@app.post("/items")
def create_item(item: ItemIn):
    return item

# Exception: 不存在就报 404
@app.get("/demo/error")
def error_demo():
    raise HTTPException(status_code=404, detail="资源不存在")
```

### 核心知识结构

```text
FastAPI 基础
├── 运行环境
│   ├── venv 虚拟环境
│   └── Uvicorn + /docs
├── 路由
│   ├── @app.get / @app.post
│   └── def / async def
├── 输入校验
│   ├── Path：路径参数
│   ├── Query：查询参数
│   └── BaseModel + Field：请求体
├── 输出
│   ├── 默认 JSON
│   ├── response_class：HTML / 文件 / 文本等
│   └── response_model：限定响应结构
└── 错误处理
    └── HTTPException
```

### 复习顺序建议

- **第 1 遍（理解）**：看一至六节，理解一次 HTTP 请求如何被处理。
- **第 2 遍（练手）**：自己写第七节的综合示例，不要复制粘贴。
- **第 3 遍（检测）**：完成第八节练习和第十一节闭卷自测。
- **以后快速回顾**：优先看第四节参数对比、第十节易混点、第十二节速查。

> **本章边界**：原课件主要围绕基础程序、路由、输入与响应，并没有系统讲解数据库持久化、ORM、依赖注入、身份认证、项目分层、测试或 Docker。后续学习这些内容时应单独整理新章节，不要与本章核心知识混在一起。