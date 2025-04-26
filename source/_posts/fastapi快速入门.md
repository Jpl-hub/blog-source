---
title: fastapi快速入门
tags: ["FastAPI","Python"]

categories: [技术文档]
---


## 为什么选FastAPI？

先贴一个官方文档

[fastapi官方文档]: https://fastapi.tiangolo.com/zh/

主要原因有这么几点：

- **速度快**：基于ASGI异步框架，性能直逼Go语言
- **开发快**：自动生成API文档
- **类型提示**：用Python的类型注解自动验证参数
- **现代化**：支持异步编程，默认就是async/await

## 环境准备

先装FastAPI和uvicorn（一个高性能的ASGI服务器）：

```bash
pip install fastapi uvicorn
```

## 第一个FastAPI应用

直接上代码

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

保存为`main.py`，然后运行：

```bash
uvicorn main:app --reload
```

打开浏览器访问 `http://localhost:8000`，你就能看到返回的JSON了。

同时，访问 `http://localhost:8000/docs`，你会看到自动生成的API文档！这是FastAPI自带的功能。

## 路由和请求处理

FastAPI的路由设计很直观，支持所有标准的HTTP方法：

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}

@app.post("/items/")
async def create_item(name: str, price: float, is_offer: bool = None):
    item = {
        "name": name,
        "price": price,
        "is_offer": is_offer
    }
    return item
```

注意：这里用了类型提示（item_id: int），FastAPI会自动做参数验证。如果有人传了个字符串给item_id，FastAPI会直接返回422错误，连写验证代码都省了。

## 请求体和数据模型

实际开发中，我们经常要处理复杂的数据结构。FastAPI配合Pydantic使用，能让数据处理变得异常简单：

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: str = None
    price: float
    tax: float = None

@app.post("/items/")
async def create_item(item: Item):
    item_dict = item.dict()
    if item.tax:
        price_with_tax = item.price + item.tax
        item_dict.update({"price_with_tax": price_with_tax})
    return item_dict
```

这里的Item是一个Pydantic模型，它不仅定义了数据结构，还自带数据验证功能。客户端发来的JSON会自动解析成Item对象。

## 查询参数和路径参数

在FastAPI中，处理URL参数和查询参数都很直观：

```python
@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(user_id: int, item_id: str, q: str = None, short: bool = False):
    item = {"item_id": item_id, "owner_id": user_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update(
            {"description": "这是一个很长的描述，在short为True时会被省略"}
        )
    return item
```

在这个例子中：

- `user_id`和`item_id`是路径参数
- `q`和`short`是查询参数，而且都有默认值

## 错误处理

FastAPI提供了优雅的错误处理机制：

```python
from fastapi import HTTPException

@app.get("/items/{item_id}")
async def read_item(item_id: str):
    if item_id not in items:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item": items[item_id]}
```

用`HTTPException`抛出HTTP错误，FastAPI会自动返回相应的错误响应。

## 依赖注入

FastAPI的依赖注入系统很强大，可以帮你管理共享的资源和认证：

```python
from fastapi import Depends

async def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/")
async def read_users(db: DBSession = Depends(get_db)):
    users = db.query(User).all()
    return users
```

这里的`get_db`函数会在每次请求时自动调用，并在请求结束后清理资源。

## 中间件

想在请求处理前后做点什么？用中间件：

```python
@app.middleware("http")
async def add_process_time_header(request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

这个中间件会给每个响应添加一个处理时间的响应头。

## 异步支持

FastAPI原生支持异步操作，特别适合I/O密集型应用：

```python
@app.get("/async-items/")
async def read_items():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
    return response.json()
```

记住：如果你的函数里有IO操作（比如数据库查询、HTTP请求），最好用异步的方式。

## 部署建议

开发阶段用`uvicorn`就够了，但生产环境建议这样部署：

1. 使用gunicorn作为进程管理器：

   ```bash
   gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
   ```

2. 配合Nginx做反向代理：

   ```nginx
   server {
       listen 80;
       server_name example.com;
       
       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   ```

