# FastAPI-Basics｜后端开发基础学习笔记

用于整理后端开发入门阶段的学习笔记、可运行示例与复习资料。目前覆盖 **MySQL 8 基础 SQL**、**FastAPI 基础**与 **FastAPI 进阶（中间件、依赖注入、ORM）**，文档为 Markdown 格式，可在 GitHub 或 Typora 阅读。

## 目录结构


```text
FastAPI-Basics/
├── 01-database/                      # 数据库基础
│   └── mysql/
│       └── SQL基础核心教程_Typora版.md
├── 02-backend/                       # Python Web 后端
│   └── fastapi/
│       ├── FastAPI_第一章_复习回顾笔记.md
│       └── FastAPI_第二章_进阶笔记.md
└── README.md
```

数据库相关笔记放到 `01-database/`，API 框架、后端开发相关内容放到 `02-backend/`。尚未学习的内容不提前创建空目录。

## 学习资料导航

| 分类 | 笔记 | 核心内容 |
| --- | --- | --- |
| **01 · 数据库 / MySQL** | [MySQL 8 基础核心教程：从读懂 SQL 到独立写查询](01-database/mysql/SQL基础核心教程_Typora版.md) | `SELECT`、增删改查、`GROUP BY`、`JOIN`、子查询、事务、练习与排错 |
| **02 · 后端 / FastAPI** | [FastAPI 第一章：基础入门复习与回顾](02-backend/fastapi/FastAPI_第一章_复习回顾笔记.md) | 同步与异步、路由、Path/Query/Body、Pydantic 校验、响应类型及异常处理 |
| **02 · 后端 / FastAPI** | [FastAPI 第二章：中间件、依赖注入与 ORM](02-backend/fastapi/FastAPI_第二章_进阶笔记.md) | 中间件与执行顺序、`Depends` 依赖注入、SQLAlchemy 异步引擎与建表、会话依赖项 `yield`、增删改查、聚合与分页 |

**建议学习顺序**：先掌握 SQL 的常用查询与事务，再学习 FastAPI 的请求处理和数据校验，然后进入 SQLAlchemy 异步 ORM、依赖注入与增删改查，之后逐步接触表关联、Alembic 迁移、PostgreSQL、Redis、Docker 等工程能力。

## FastAPI 快速开始

在项目目录下创建 Python 虚拟环境并安装依赖：

```bash
python -m venv .venv
# Windows PowerShell: .\.venv\Scripts\Activate.ps1
# Linux / macOS: source .venv/bin/activate
python -m pip install fastapi uvicorn
```

新建 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello World"}
```

开发环境启动：

```bash
python -m uvicorn main:app --reload
```

- 接口：<http://127.0.0.1:8000/>
- Swagger UI：<http://127.0.0.1:8000/docs>
- OpenAPI JSON：<http://127.0.0.1:8000/openapi.json>

> `--reload` 仅用于本地开发；数据库练习请使用笔记中的独立测试库，不要直接在生产库执行修改语句。

## 复习建议

1. 先不看答案，独立说出概念与代码执行流程。
2. 运行笔记示例，并修改参数、查询条件来观察结果。
3. 对易错知识点建立对照表，间隔几天重新完成练习。

## 参考资料

- 黑马程序员《FastAPI 基础到实战》
- Datawhale：[wonderful-sql](https://github.com/datawhalechina/wonderful-sql)
- [MySQL 8.0 参考手册](https://dev.mysql.com/doc/refman/8.0/en/)
