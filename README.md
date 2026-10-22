# FastAPI

FastAPI 学习与实践仓库，用于整理基础知识、复习笔记和后续项目代码。

## 学习文档

| 章节 | 内容 | 文档 |
| --- | --- | --- |
| 第一章：FastAPI 基础入门 | 同步与异步、环境搭建、路由、路径参数、查询参数、请求体、Pydantic 数据校验、响应类型、`response_model` 与异常处理 | [复习与回顾笔记](docs/FastAPI_第一章_复习回顾笔记.md) |

笔记采用 Markdown 编写，可直接在 GitHub 或 Typora 中阅读。包含知识梳理、示例代码、易错点和复习练习。

## 快速开始

在项目目录下创建 Python 虚拟环境并安装依赖：

```bash
python -m venv .venv
# Windows PowerShell：.venv\Scripts\Activate.ps1
# Linux / macOS：source .venv/bin/activate
python -m pip install fastapi uvicorn
```

编写 `main.py`：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello World"}
```

启动开发服务：

```bash
python -m uvicorn main:app --reload
```

- 接口地址：<http://127.0.0.1:8000/>
- 交互式 API 文档：<http://127.0.0.1:8000/docs>
- OpenAPI 文档：<http://127.0.0.1:8000/openapi.json>

> `--reload` 用于本地开发阶段，不应直接用于生产部署。
