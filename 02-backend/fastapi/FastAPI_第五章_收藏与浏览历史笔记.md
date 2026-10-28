# FastAPI 第五章｜AI 掘金头条：收藏与浏览历史

> **定位**：学完后用于快速复习、回忆核心概念、照着代码恢复手感，而不是逐页重读 PPT。  
> **资料依据**：《第五章_AI掘金头条-收藏和浏览历史.pdf》（31 页，黑马程序员《FastAPI 基础到实战》），并参照配套代码 `toutiao_backend/`、`database.sql` 与《API 接口规范文档》。  
> **覆盖范围**：`favorite` / `history` 两张表、`UniqueConstraint` 唯一约束、9 个接口（收藏 5 个 + 历史 4 个）、ORM 联表查询 `join` 与字段别名 `label`、分页与 `hasMore`、`delete().rowcount` 判断。  
> **阅读方法**：先看每节的「一句话记忆」，合上笔记说出原理，再看代码，最后做文末练习。

[TOC]

---

## 一、先建立整体认识

### 1. 这一章在项目里的位置

第四章解决了"你是谁"，这一章解决"**你和新闻之间发生过什么**"：

```text
用户 + 新闻 → 收藏（我喜欢）      → favorite 表
用户 + 新闻 → 浏览历史（我看过）   → history 表
```

两张表都是典型的**关联表**：只存"谁"和"哪条新闻"，加上一个时间字段。

### 2. 本章的 9 个接口

**收藏模块**（课件第 1 页）：

| 接口 | 方法 | 说明 | 认证 |
| --- | --- | --- | --- |
| `/api/favorite/check` | GET | 检查新闻收藏状态 | ✅ |
| `/api/favorite/add` | POST | 添加收藏 | ✅ |
| `/api/favorite/remove` | DELETE | 取消收藏 | ✅ |
| `/api/favorite/list` | GET | 收藏列表（分页） | ✅ |
| `/api/favorite/clear` | DELETE | 清空收藏 | ✅ |

**浏览历史模块**（课件第 33 页）：

| 接口 | 方法 | 说明 | 认证 |
| --- | --- | --- | --- |
| `/api/history/add` | POST | 添加浏览记录 | ✅ |
| `/api/history/list` | GET | 浏览历史列表（分页） | ✅ |
| `/api/history/delete/{history_id}` | DELETE | 删除单条记录 | ✅ |
| `/api/history/clear` | DELETE | 清空浏览历史 | ✅ |

**一句话记忆：9 个接口全都要登录——因为"收藏/历史"本身就是"某个人的"数据，没有 `user_id` 根本无从谈起。**

### 3. 两条主线

| 主线 | 涉及接口 | 关键点 |
| --- | --- | --- |
| **写操作** | check / add / remove / delete / clear | `add` 与 `delete`，用 `rowcount` 判断是否命中 |
| **读操作** | list × 2 | **联表查询** `join` + 分页 + `hasMore` |

本章真正的技术难点只有一个：**收藏列表和历史列表要把 `News` 和关联表拼在一起查出来**。

---

## 二、两张表与模型类

### 1. 表结构

**`favorite`（收藏表）** —— `database.sql`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INT PK | 收藏 ID |
| `user_id` | INT FK → `user.id` | 用户 |
| `news_id` | INT FK → `news.id` | 新闻 |
| `created_at` | TIMESTAMP | 收藏时间 |

约束：`UNIQUE (user_id, news_id)` + 两个外键索引。

**`history`（浏览历史表）**：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | INT PK | 历史 ID |
| `user_id` | INT FK → `user.id` | 用户 |
| `news_id` | INT FK → `news.id` | 新闻 |
| `view_time` | TIMESTAMP | 浏览时间 |

约束：两个外键索引 + `idx_view_time`（浏览时间倒序）。

### 2. 模型类

```python
# models/favorite.py
from datetime import datetime

from sqlalchemy import UniqueConstraint, Index, Integer, ForeignKey, DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from models.news import News
from models.users import User


class Base(DeclarativeBase):
    pass


class Favorite(Base):
    __tablename__ = "favorite"

    # 创建索引
    # UniqueConstraint: 唯一约束, 当前用户，当前新闻，只能收藏一次
    __table_args__ = (
        UniqueConstraint("user_id", "news_id", name="user_news_unique"),
        Index("fk_favorite_user_idx", "user_id"),
        Index("fk_favorite_news_idx", "news_id"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True, comment="收藏ID")
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey(User.id), nullable=False, comment="用户ID")
    news_id: Mapped[int] = mapped_column(Integer, ForeignKey(News.id), nullable=False, comment="新闻ID")
    created_at: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now, nullable=False, comment="收藏时间")
```

```python
# models/history.py
from datetime import datetime

from sqlalchemy import Integer, DateTime, ForeignKey, Index
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from models.news import News
from models.users import User


class Base(DeclarativeBase):
    pass


class History(Base):
    __tablename__ = "history"

    __table_args__ = (
        Index("fk_history_user_idx", "user_id"),
        Index("fk_history_news_idx", "news_id"),
        Index("idx_view_time", "view_time"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True, comment="历史ID")
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey(User.id), nullable=False, comment="用户ID")
    news_id: Mapped[int] = mapped_column(Integer, ForeignKey(News.id), nullable=False, comment="新闻ID")
    view_time: Mapped[datetime] = mapped_column(
        DateTime, default=datetime.now, nullable=False, comment="浏览时间")
```

### 3. `UniqueConstraint` 的作用

课件第 30 页专门强调：

```python
__table_args__ = (
    UniqueConstraint('user_id', 'news_id', name='user_news_unique'),
)
```

| 写法 | 作用 |
| --- | --- |
| `Index(..., "user_id")` | 加速查询（不限制重复） |
| `UniqueConstraint("user_id", "news_id")` | **组合唯一**：同一用户对同一新闻只能有一条记录 |

**一句话记忆：唯一约束是数据库层面的"防重复"，比在代码里 `if 已存在` 更可靠——并发下也不会重复。**

> 注意：`favorite` 有唯一约束，`history` **没有**。这是业务设计的差异：收藏必须唯一，浏览历史理论上可以有多条（本项目代码改为"存在则更新时间"，效果上也是唯一）。

---

## 三、收藏模块：5 个接口

先建路由实例：

```python
router = APIRouter(prefix="/api/favorite", tags=["favorite"])
```

### 1. 检查收藏状态

**接口**：`GET /api/favorite/check?newsId=1`

```text
进入请求 → 验证用户是否登录 → 检查用户是否收藏了当前新闻 → 响应 { isFavorite }
```

```python
@router.get("/check")
async def check_favorite(
        news_id: int = Query(..., alias="newsId"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    is_favorited = await favorite.is_news_favorite(db, user.id, news_id)
    return success_response(
        message="检查收藏状态成功",
        data=FavoriteCheckResponse(isFavorite=is_favorited))
```

```python
async def is_news_favorite(db: AsyncSession, user_id: int, news_id: int):
    query = select(Favorite).where(
        Favorite.user_id == user_id, Favorite.news_id == news_id)
    result = await db.execute(query)
    return result.scalar_one_or_none() is not None
```

**要点**：查询条件必须**同时**带 `user_id` 和 `news_id`，否则就是"全世界有没有人收藏过这条新闻"，业务含义完全不同。

### 2. 添加收藏

**接口**：`POST /api/favorite/add`，请求体 `{ "newsId": 1 }`

```text
进入请求 → 验证登录 → add() 添加收藏 → 响应
```

```python
@router.post("/add")
async def add_favorite(
        data: FavoriteAddRequest,
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    result = await favorite.add_news_favorite(db, user.id, data.news_id)
    return success_response(message="添加收藏成功", data=result)
```

```python
async def add_news_favorite(db: AsyncSession, user_id: int, news_id: int):
    favorite = Favorite(user_id=user_id, news_id=news_id)
    db.add(favorite)
    await db.commit()
    await db.refresh(favorite)
    return favorite
```

请求模型（注意是**请求体**，不是查询参数）：

```python
class FavoriteAddRequest(BaseModel):
    news_id: int = Field(..., alias="newsId")
```

**坑**：重复收藏会撞 `user_news_unique` 约束，抛 `IntegrityError`，被全局异常处理器转成 400「数据约束冲突」。更友好的做法是**先 `is_news_favorite` 判断**，已收藏就返回成功或 409。

### 3. 取消收藏

**接口**：`DELETE /api/favorite/remove?newsId=1`

```text
进入请求 → 验证登录 → delete() 删除收藏表内当前新闻 → 检查命中数量
              ├── > 0 → 响应成功
              └── 0   → 抛出异常（404）
```

```python
@router.delete("/remove")
async def remove_favorite(
        news_id: int = Query(..., alias="newsId"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    result = await favorite.remove_news_favorite(db, user.id, news_id)
    if not result:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="收藏记录不存在")
    return success_response(message="删除收藏成功")
```

```python
async def remove_news_favorite(db: AsyncSession, user_id: int, news_id: int):
    stmt = delete(Favorite).where(
        Favorite.user_id == user_id, Favorite.news_id == news_id)
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount > 0
```

**一句话记忆：删除操作一定要用 `rowcount` 判断"到底删到了没有"**，删 0 条不能返回成功。

### 4. 收藏列表

**接口**：`GET /api/favorite/list?page=1&pageSize=10`

这是本章最复杂的接口，四件事：**验证登录 → 统计总量 → 联表查询 → 计算是否有更多**。

```python
@router.get("/list")
async def get_favorite_list(
        page: int = Query(1, ge=1),
        page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    rows, total = await favorite.get_favorite_list(db, user.id, page, page_size)

    favorite_list = [{
        **news.__dict__,
        "favorite_time": favorite_time,
        "favorite_id": favorite_id
    } for news, favorite_time, favorite_id in rows]

    has_more = total > page * page_size

    data = FavoriteListResponse(list=favorite_list, total=total, hasMore=has_more)
    return success_response(message="获取收藏列表成功", data=data)
```

CRUD 部分见下一节（联表查询单独讲）。

### 5. 清空收藏

**接口**：`DELETE /api/favorite/clear`

```python
@router.delete("/clear")
async def clear_favorite(
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    count = await favorite.remove_all_favorites(db, user.id)
    return success_response(message=f"清空了{count}条记录")
```

```python
async def remove_all_favorites(db: AsyncSession, user_id: int):
    stmt = delete(Favorite).where(Favorite.user_id == user_id)
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount or 0
```

**要点**：`where(user_id)` 一个都不能少——少了就是清空**全站**收藏。返回值用 `rowcount` 告诉用户"清了几条"，比"清空成功"信息量更大。

---

## 四、联表查询：`join` 与字段别名

这是课件第 30 页的核心内容。

### 1. 为什么必须联表

收藏表里只有 `news_id`，没有标题、封面、作者。而前端列表要展示的是**新闻卡片**：

```json
{ "id": 1, "title": "...", "image": "...", "favoriteTime": "..." }
```

所以必须把 `favorite` 和 `news` 拼起来查。用两次查询也能实现（先查收藏 id 列表，再 `where(id.in_(...))`），但会多一次往返，而且排序依赖收藏时间时很难写。

### 2. 写法

课件给的模板：

```python
select( 查询的主体模型类, 字段重命名 ).join( join 的模型类, join 的条件 )
```

实际代码：

```python
query = (
    select(
        News,
        Favorite.created_at.label("favorite_time"),
        Favorite.id.label("favorite_id"),
    )
    .join(Favorite, Favorite.news_id == News.id)
    .where(Favorite.user_id == user_id)
    .order_by(Favorite.created_at.desc())
    .offset(offset).limit(page_size)
)
result = await db.execute(query)
rows = result.all()      # [(新闻对象, 收藏时间, 收藏id), ...]
```

逐个拆解：

| 部分 | 含义 |
| --- | --- |
| `select(News, ...)` | 主体是新闻，结果每行是"新闻 + 附加字段"的元组 |
| `.label("favorite_time")` | 给字段起别名，元组解包和后续 `**dict` 时好取值 |
| `.join(Favorite, Favorite.news_id == News.id)` | 内连接条件：收藏记录指向这条新闻 |
| `.where(Favorite.user_id == user_id)` | **只查当前用户**的收藏 |
| `.order_by(Favorite.created_at.desc())` | 按收藏时间倒序（最近收藏在前） |
| `.offset(offset).limit(page_size)` | 分页 |
| `result.all()` | 返回**元组列表**，不是标量列表 |

### 3. `scalars()` 与 `all()` 的区别

| 方法 | 返回 | 什么时候用 |
| --- | --- | --- |
| `result.scalars().all()` | 单个对象的列表 | 只 `select(News)` |
| `result.all()` | **元组**的列表 | `select(News, 别的字段)` |

**一句话记忆：查了多列就用 `all()`，结果要自己解包；查单表才用 `scalars()`。**

### 4. 拼成响应

```python
favorite_list = [{
    **news.__dict__,              # 新闻的所有字段
    "favorite_time": favorite_time,
    "favorite_id": favorite_id,
} for news, favorite_time, favorite_id in rows]
```

- `for news, favorite_time, favorite_id in rows` → 元组解包，顺序与 `select` 里一致
- `**news.__dict__` → 把 ORM 对象的字段摊平进字典（含 `_sa_instance_state`，所以更规范的做法是 `NewsItemBase.model_validate(news).model_dump()`）

### 5. 响应模型

```python
# schemas/base.py —— 新闻条目的公共字段
class NewsItemBase(BaseModel):
    id: int
    title: str
    description: Optional[str] = None
    image: Optional[str] = None
    author: Optional[str] = None
    category_id: int = Field(alias="categoryId")
    views: int
    publish_time: Optional[datetime] = Field(None, alias="publishedTime")

    model_config = ConfigDict(from_attributes=True, populate_by_name=True)


# schemas/favorite.py
class FavoriteNewsItemResponse(NewsItemBase):
    favorite_id: int = Field(alias="favoriteId")
    favorite_time: datetime = Field(alias="favoriteTime")

    model_config = ConfigDict(populate_by_name=True, from_attributes=True)


class FavoriteListResponse(BaseModel):
    list: list[FavoriteNewsItemResponse]
    total: int
    has_more: bool = Field(alias="hasMore")
```

**设计思路**：把"新闻的公共字段"抽到 `NewsItemBase`，收藏项继承它再加自己的字段，历史项同理。这样两个列表接口的新闻部分字段永远一致。

---

## 五、浏览历史模块：4 个接口

```python
router = APIRouter(prefix="/api/history", tags=["history"])
```

### 1. 添加浏览记录

**接口**：`POST /api/history/add`，请求体 `{ "newsId": 1 }`

课件第 41 页的流程图有一个关键分支：

```text
进入请求 → 验证登录 → 检查是否浏览过当前新闻
                        ├── 否 → add() 新增历史记录
                        └── 是 → 更新浏览时间
```

```python
async def add_history(db: AsyncSession, user_id: int, news_id: int):
    query = select(History).where(
        History.user_id == user_id, History.news_id == news_id)
    result = await db.execute(query)
    existing_history = result.scalar_one_or_none()

    if existing_history:
        existing_history.view_time = datetime.now()     # 已看过 → 只更新时间
        await db.commit()
        await db.refresh(existing_history)
        return existing_history
    else:
        history = History(user_id=user_id, news_id=news_id)
        db.add(history)
        await db.commit()
        await db.refresh(history)
        return history
```

**一句话记忆：浏览历史是"最近看过"列表，不是"每次看都记一条"——同一篇新闻只保留最新时间。**

### 2. 浏览历史列表

**接口**：`GET /api/history/list?page=1&pageSize=10`

结构与收藏列表完全对称，只是排序字段换成 `view_time`、别名换成 `history_id`：

```python
query = (
    select(
        News,
        History.view_time.label("view_time"),
        History.id.label("history_id"),
    )
    .join(History, History.news_id == News.id)
    .where(History.user_id == user_id)
    .order_by(History.view_time.desc())
    .offset(offset).limit(page_size)
)
```

```python
history_list = [
    HistoryNewsItemResponse.model_validate({
        **news.__dict__,
        "view_time": view_time,
        "history_id": history_id,
    })
    for news, view_time, history_id in rows
]
```

### 3. 删除单条记录

**接口**：`DELETE /api/history/delete/{history_id}`（**路径参数**，注意和收藏的查询参数不同）

```python
@router.delete("/delete/{history_id}")
async def delete_history(history_id: int,
                         user: User = Depends(get_current_user),
                         db: AsyncSession = Depends(get_db)):
    result = await history.delete_history(db, user.id, history_id)
    if not result:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="历史记录不存在")
    return success_response(message="删除成功")
```

```python
async def delete_history(db: AsyncSession, user_id: int, history_id: int):
    query = delete(History).where(
        History.user_id == user_id, History.id == history_id)
    result = await db.execute(query)
    await db.commit()
    return result.rowcount > 0
```

> ⚠️ 课件配套代码这里有个 bug：路由参数叫 `history_id`（历史记录主键），但 `crud.delete_history` 的 `where` 条件写的是 `History.news_id == news_id`。按接口文档语义，**应当按 `History.id` 删除**，否则传历史 id 却按新闻 id 匹配，会删错或删不掉。本笔记已按 `History.id` 修正。

### 4. 清空浏览历史

```python
@router.delete("/clear")
async def clear_history(user: User = Depends(get_current_user),
                        db: AsyncSession = Depends(get_db)):
    result = await history.clear_history(db, user.id)
    return success_response(message="清空成功")
```

```python
async def clear_history(db: AsyncSession, user_id: int):
    query = delete(History).where(History.user_id == user_id)
    result = await db.execute(query)
    await db.commit()
    return result.rowcount or 0
```

---

## 六、收藏 vs 浏览历史：一张表看清差异

课件第 34 页把两个模块并排放，差别其实就是**一张表的字段差异**：

| 对比项 | 收藏 | 浏览历史 |
| --- | --- | --- |
| 表名 | `favorite` | `history` |
| 时间字段 | `created_at`（收藏时间） | `view_time`（浏览时间） |
| 唯一约束 | ✅ `UNIQUE(user_id, news_id)` | ❌（代码里"存在则更新时间"实现等效唯一） |
| 重复操作的语义 | 报错（已收藏） | 更新时间（再看一次） |
| 单条删除依据 | 查询参数 `newsId` | 路径参数 `historyId` |
| 列表排序 | `created_at DESC` | `view_time DESC` |
| 列表附加字段 | `favoriteId`、`favoriteTime` | `historyId`、`viewTime` |
| 恢复操作 | 添加收藏 | 添加浏览记录 |

**一句话记忆：两个模块是同一个套路的两次实现——谁（user_id）+ 哪条（news_id）+ 什么时候（时间）+ 联表查新闻。**

因为结构一样，实现时可以复制改写；但也正因为太像，最容易犯的错是**复制后忘了别名**（把 `favorite_time` 留在历史响应里）。

---

## 七、分页与 `hasMore`

### 1. 两种算法

第三章用的是"当前条数法"：

```python
has_more = (offset + len(news_list)) < total
```

本章项目代码用的是"页码法"：

```python
has_more = total > page * page_size
```

| 算法 | 优点 | 缺点 |
| --- | --- | --- |
| `(offset + len) < total` | 最后一页不满、中途删除都算得准 | 需要拿到实际条数 |
| `total > page * page_size` | 不依赖查询结果，少一次判断 | 末页刚好满时会多返回一次"有更多" |

**建议**：两种都能用，同一个项目里**统一一种**即可。本仓库第三章用第一种，本章项目用第二种，知道差别就不算错。

### 2. 分页三件套

```python
offset = (page - 1) * page_size        # 跳过多少条
.limit(page_size)                       # 本页取多少
total = 总量                            # 用于算 hasMore
```

### 3. 统计总量

```python
# 收藏
count_query = select(func.count()).where(Favorite.user_id == user_id)

# 历史
count_query = select(func.count(History.id)).where(History.user_id == user_id)
```

两种写法等价，`func.count(History.id)` 更明确"数的是主键行数"。

**注意**：统计和列表查询是两次 SQL，中间如果有并发写入，`total` 与 `list` 可能不完全吻合——这是分页的固有问题，不必过度纠结。

---

## 八、原课件练习题整理（建议先独立完成）

### 练习 1：模型类（课件第 30 页）

写出 `Favorite`、`History` 模型类，要求：
- 收藏表有 `UNIQUE(user_id, news_id)`；
- 两张表都有 `user_id`、`news_id` 外键；
- 历史表有 `view_time` 索引。

### 练习 2：检查收藏状态（课件第 3 页）

`GET /api/favorite/check?newsId=1`，返回 `{ "isFavorite": true }`。

### 练习 3：添加与取消收藏（课件第 6、9 页）

添加：请求体传 `newsId`；取消：查询参数传 `newsId`，删不到要 404。

### 练习 4：收藏列表（课件第 12–14 页）

分页返回 `list`、`total`、`hasMore`，列表项包含新闻字段 + `favoriteId` + `favoriteTime`。

### 练习 5：清空收藏（课件第 16、17 页）

返回消息里带上实际删除条数，如"清空了 3 条记录"。

### 练习 6：浏览历史四接口（课件第 41–51 页）

添加（存在则更新时间）、列表（按 `view_time` 倒序）、删除单条（按历史 id）、清空。

### 练习 7（思考题）

1. 为什么收藏表需要 `UNIQUE(user_id, news_id)`，浏览历史表却不需要？
2. 收藏列表为什么不能用 `scalars().all()` 取值？

---

## 九、练习参考答案

### 练习 1

```python
class Favorite(Base):
    __tablename__ = "favorite"

    __table_args__ = (
        UniqueConstraint("user_id", "news_id", name="user_news_unique"),
        Index("fk_favorite_user_idx", "user_id"),
        Index("fk_favorite_news_idx", "news_id"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
    news_id: Mapped[int] = mapped_column(Integer, ForeignKey("news.id"), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now, nullable=False)


class History(Base):
    __tablename__ = "history"

    __table_args__ = (
        Index("fk_history_user_idx", "user_id"),
        Index("fk_history_news_idx", "news_id"),
        Index("idx_view_time", "view_time"),
    )

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("user.id"), nullable=False)
    news_id: Mapped[int] = mapped_column(Integer, ForeignKey("news.id"), nullable=False)
    view_time: Mapped[datetime] = mapped_column(DateTime, default=datetime.now, nullable=False)
```

### 练习 2

```python
@router.get("/check")
async def check_favorite(
        news_id: int = Query(..., alias="newsId"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    is_fav = await favorite.is_news_favorite(db, user.id, news_id)
    return success_response(message="检查收藏状态成功",
                            data=FavoriteCheckResponse(isFavorite=is_fav))
```

### 练习 3

```python
@router.post("/add")
async def add_favorite(data: FavoriteAddRequest,
                       user: User = Depends(get_current_user),
                       db: AsyncSession = Depends(get_db)):
    result = await favorite.add_news_favorite(db, user.id, data.news_id)
    return success_response(message="添加收藏成功", data=result)


@router.delete("/remove")
async def remove_favorite(news_id: int = Query(..., alias="newsId"),
                          user: User = Depends(get_current_user),
                          db: AsyncSession = Depends(get_db)):
    ok = await favorite.remove_news_favorite(db, user.id, news_id)
    if not ok:
        raise HTTPException(status_code=404, detail="收藏记录不存在")
    return success_response(message="删除收藏成功")
```

### 练习 4

```python
@router.get("/list")
async def get_favorite_list(
        page: int = Query(1, ge=1),
        page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    rows, total = await favorite.get_favorite_list(db, user.id, page, page_size)
    items = [{
        **news.__dict__,
        "favorite_time": favorite_time,
        "favorite_id": favorite_id,
    } for news, favorite_time, favorite_id in rows]
    data = FavoriteListResponse(
        list=items, total=total, hasMore=total > page * page_size)
    return success_response(message="获取收藏列表成功", data=data)
```

### 练习 5

```python
@router.delete("/clear")
async def clear_favorite(user: User = Depends(get_current_user),
                         db: AsyncSession = Depends(get_db)):
    count = await favorite.remove_all_favorites(db, user.id)
    return success_response(message=f"清空了{count}条记录")
```

### 练习 6

```python
@router.post("/add")
async def add_history(data: HistoryAddRequest,
                      user: User = Depends(get_current_user),
                      db: AsyncSession = Depends(get_db)):
    result = await history.add_history(db, user.id, data.news_id)
    return success_response(message="添加成功", data=result)


@router.get("/list")
async def get_history_list(
        page: int = Query(1, ge=1),
        page_size: int = Query(10, ge=1, le=100, alias="pageSize"),
        user: User = Depends(get_current_user),
        db: AsyncSession = Depends(get_db)):
    rows, total = await history.get_history_list(db, user.id, page, page_size)
    has_more = total > page * page_size
    items = [HistoryNewsItemResponse.model_validate({
        **news.__dict__, "view_time": view_time, "history_id": history_id,
    }) for news, view_time, history_id in rows]
    return success_response(data=HistoryListResponse(
        list=items, total=total, hasMore=has_more))


@router.delete("/delete/{history_id}")
async def delete_history(history_id: int,
                         user: User = Depends(get_current_user),
                         db: AsyncSession = Depends(get_db)):
    ok = await history.delete_history(db, user.id, history_id)
    if not ok:
        raise HTTPException(status_code=404, detail="历史记录不存在")
    return success_response(message="删除成功")


@router.delete("/clear")
async def clear_history(user: User = Depends(get_current_user),
                        db: AsyncSession = Depends(get_db)):
    await history.clear_history(db, user.id)
    return success_response(message="清空成功")
```

### 练习 7

1. 收藏是"表态"，同一篇新闻对同一用户只应存在一次，重复收藏没有意义，所以用唯一约束在数据库层挡住；浏览历史是"时间线"，语义上允许重复记录，但为了列表整洁，代码用"存在则更新时间"实现等效唯一。  
2. 因为 `select(News, Favorite.created_at, Favorite.id)` 返回的是**元组**，`scalars()` 只会取每行的第一个元素，`favorite_time` / `favorite_id` 会丢。

---

## 十、最容易混淆的知识点

| 容易混淆 | 正确认识 |
| --- | --- |
| `Index` 与 `UniqueConstraint` | 前者只加速，后者强制唯一；`favorite` 两个都要 |
| 收藏与历史的时间字段 | `favorite.created_at` vs `history.view_time`，别名别抄错 |
| `scalars().all()` 与 `all()` | 单表用前者，多列用后者（返回元组） |
| `join` 的条件 | 必须是 `Favorite.news_id == News.id`（关联键），不是 `Favorite.id == News.id` |
| `where` 漏 `user_id` | 会查到/删到**别人**的数据，这是最严重的越权 bug |
| `rowcount` | 删除/更新后判断是否命中，0 就要 404 |
| 收藏删除用查询参数，历史删除用路径参数 | 接口文档如此规定，别写反 |
| `hasMore` 两种算法 | `(offset+len)<total` 与 `total > page*page_size`，同一项目统一一种 |
| `**news.__dict__` | 会带上 `_sa_instance_state`；规范做法是 Pydantic `model_dump()` |
| `created_at` 默认值 | 收藏表用 `datetime.now`，历史表也用 `datetime.now`（原代码曾用 `utcnow`，注意时区一致） |
| 重复收藏 | 直接 `add` 会触发 `IntegrityError`，应先 `check` |
| 清空接口 | `where(user_id)` 必须保留，否则清的是全站数据 |

### 课件示例与项目代码的整理说明

1. **`delete_history` 的过滤字段**：项目代码路由参数是 `history_id`，而 CRUD 用 `History.news_id` 过滤，语义不一致。本笔记按接口文档修正为 `History.id == history_id`。
2. **`hasMore` 算法**：课件第 14 页写"是否有更多"，项目用 `total > page * page_size`；第三章用的是 `(offset + len) < total`。两种都对，本笔记并列说明。
3. **`Favorite.created_at` 的默认值**：原代码混用 `datetime.utcnow` 与 `datetime.now`，会导致收藏时间与浏览时间差 8 小时；本笔记统一为 `datetime.now`。
4. **`**news.__dict__` 的隐患**：会把 SQLAlchemy 内部字段带进字典，本笔记注明规范写法（`NewsItemBase.model_validate(news).model_dump()`）。
5. **清空收藏的消息**：课件写 `f"清空{xx}条收藏记录"`，项目写 `f"清空了{count}条记录"`，含义一致。

以上属于**正确性与一致性修正**，不改变原课件知识点。

---

## 十一、闭卷自测：10 分钟检验掌握情况

1. 收藏表和浏览历史表的字段有什么区别？为什么收藏需要唯一约束？
2. `UniqueConstraint("user_id", "news_id")` 和 `Index("user_id")` 分别做什么？
3. 收藏的 5 个接口分别是什么？哪些是写操作？
4. `check` 接口的 `where` 条件为什么必须同时包含 `user_id` 和 `news_id`？
5. 添加收藏时重复收藏会发生什么？怎么避免？
6. 取消收藏为什么要判断 `rowcount`？
7. `select(News, Favorite.created_at.label("favorite_time"))` 返回的是什么？怎么取值？
8. `join` 的第二个参数写什么？写错了会怎样？
9. 收藏列表的响应为什么要用 `NewsItemBase` 继承？
10. 浏览历史"添加"为什么要先查是否存在？
11. `scalars().all()` 和 `all()` 的区别？
12. 清空接口最危险的写法是什么？

**达标标准**：能不看笔记写出两张表模型 + 9 个接口 + 两个联表查询，并用 `/docs` 走通"收藏 → 查状态 → 列表 → 取消 → 清空"。

**参考答案**（先自己回答，再对照）：

1. 收藏用 `created_at`，历史用 `view_time`；收藏是"表态"只能有一条，所以加唯一约束，历史是"时间线"。  
2. 前者是组合唯一约束（防重复），后者是普通索引（加速按用户查询）。  
3. `check`(GET)、`add`(POST)、`remove`(DELETE)、`list`(GET)、`clear`(DELETE)；写操作是 add / remove / clear。  
4. 只查 `news_id` 得到的是"是否有人收藏过"，业务要的是"**我**是否收藏过"。  
5. 撞 `user_news_unique` 抛 `IntegrityError`，被全局异常处理器转成 400；应在 `add` 前先 `is_news_favorite` 判断。  
6. 删 0 条说明该收藏不存在（或不属于当前用户），应返回 404 而不是成功。  
7. 返回元组列表 `[(News 对象, 收藏时间), ...]`，用 `for news, favorite_time in rows` 解包，或 `row[0]`、`row[1]`。  
8. 写关联键 `Favorite.news_id == News.id`；写成 `Favorite.id == News.id` 会拼出错误的笛卡尔结果。  
9. 让收藏项、历史项的新闻字段完全一致，避免两处维护两套字段导致前端对不上。  
10. 因为"看过"同一篇只保留一条记录、刷新时间即可，否则列表会被同一篇新闻刷屏。  
11. 前者返回单对象列表（只取第一列），后者返回元组列表（多列时用）。  
12. `delete(Favorite)` 不加 `where(Favorite.user_id == user_id)`——会把全站收藏清空。

---

## 十二、一页速查

### 两张表模型模板

```python
class Favorite(Base):
    __tablename__ = "favorite"
    __table_args__ = (
        UniqueConstraint("user_id", "news_id", name="user_news_unique"),
        Index("fk_favorite_user_idx", "user_id"),
        Index("fk_favorite_news_idx", "news_id"),
    )
    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), nullable=False)
    news_id: Mapped[int] = mapped_column(ForeignKey("news.id"), nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.now)
```

### 联表查询模板

```python
query = (
    select(
        News,
        Favorite.created_at.label("favorite_time"),
        Favorite.id.label("favorite_id"),
    )
    .join(Favorite, Favorite.news_id == News.id)
    .where(Favorite.user_id == user_id)
    .order_by(Favorite.created_at.desc())
    .offset((page - 1) * page_size).limit(page_size)
)
rows = (await db.execute(query)).all()     # 元组列表
```

### 删除与计数模板

```python
# 删除单条 / 清空：都用 rowcount 判断
stmt = delete(Favorite).where(Favorite.user_id == user_id, Favorite.news_id == news_id)
result = await db.execute(stmt)
await db.commit()
ok = result.rowcount > 0

# 统计总量
total = (await db.execute(select(func.count(Favorite.id))
                          .where(Favorite.user_id == user_id))).scalar_one()
```

### 接口一览

| 接口 | 方法 | 参数位置 | 关键 ORM |
| --- | --- | --- | --- |
| `/api/favorite/check` | GET | Query `newsId` | `select().where(user_id, news_id)` |
| `/api/favorite/add` | POST | Body `newsId` | `add()` + `refresh()` |
| `/api/favorite/remove` | DELETE | Query `newsId` | `delete().rowcount` |
| `/api/favorite/list` | GET | Query `page`/`pageSize` | `join` + `label` + `offset/limit` |
| `/api/favorite/clear` | DELETE | — | `delete().where(user_id)` |
| `/api/history/add` | POST | Body `newsId` | 存在→更新时间；不存在→`add()` |
| `/api/history/list` | GET | Query `page`/`pageSize` | `join` + 按 `view_time` 倒序 |
| `/api/history/delete/{history_id}` | DELETE | Path | `delete().where(user_id, id)` |
| `/api/history/clear` | DELETE | — | `delete().where(user_id)` |

### 核心知识结构

```text
收藏 / 浏览历史
├── 两张关联表
│   ├── favorite  user_id + news_id + created_at （唯一约束）
│   └── history   user_id + news_id + view_time  （时间索引）
├── 认证
│   └── 全部接口 Depends(get_current_user)
├── 写操作
│   ├── add   → add() / 存在则更新时间
│   ├── remove → delete() + rowcount
│   └── clear → delete().where(user_id) + rowcount 条数
└── 读操作
    ├── join(关联表, 关联键)
    ├── label() 起别名 → 元组解包
    ├── order_by(时间.desc())
    └── total + hasMore
```

### 复习顺序建议

- **第 1 遍（理解）**：看一至三节，把收藏 5 个接口搞明白。
- **第 2 遍（啃难点）**：第四节联表查询，自己写一遍 `select(News, ...).join(...)`。
- **第 3 遍（迁移）**：把收藏那套照搬到浏览历史，对比第六节的差异表。
- **以后快速回顾**：优先看第四、六、十节。

> **本章边界**：课件围绕"用户的收藏与浏览历史"讲解了关联表、唯一约束、联表查询与分页，**没有**系统讲解多对多关系的 `relationship()` 双向导航、`joinedload`/`selectinload` 解决 N+1、批量删除的性能、软删除（`is_deleted`）、收藏分组与标签、浏览历史的去重策略（按天聚合），也没有讲 `ON DELETE CASCADE` 的实际效果。这些内容后续单独整理，不要与本章混在一起。
