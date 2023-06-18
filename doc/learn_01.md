# 安装

需要安装fastAPI及服务器运行组件uvicorn. 命令如下

```bash
pip install "fastapi[all]"
```

# 简单使用


编写简易代码

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
async def root():
    return {"message": "Hello World"}
```

长得和flask的差不多, 这里采用异步的方式. 实际上即使这里不用async, 使用uvicorn开启服务后也是异步的执行. 只不过使用async后, 代码内能编写支持await操作的请求, 从而在更细的粒度上优化异步执行效率. 换言之, 如果单个路由函数不使用async, uvicorn开启服务后会将整个函数体当做整体作为异步执行, 但使用async后, 我们可以更加精细地调控异步执行范围, 不只局限于整个函数体.

使用uvicorn执行代码如下

```bash
uvicorn learn_01:app --reload --port 4399 --host 0.0.0.0
```

这里的port和host都指定了一下, 因为在服务端, 本地想要访问还是要指定port和host的. 而后, 就可以在任意客户端通过`host:port/`访问该API了.

同时, fastapi还提供了API的文档接口, 地址为`host:port/docs`或者`host:port/redoc`. 二者的画风不一样, 但都提供了接口的可视化界面.

最后, uvicorn在`--reload`模式下会根据代码更新动态地更新API服务. 因此, 使用fastapi很方便进行调试.

# 路径参数

## 指定参数类型

如下

```python
@app.get("/item/{item_id}")
async def read_item(item_id:int):
    return {"itemid": item_id}
```

这里指定了`item_id`为`int`类型, 这时候输入`host:port/item/123`不会有错, 因为类型匹配. 然而, 诸如`host:port/item/str`或者`host:port/item/4.2`之类的, 会因为类型不匹配而返回一个错误的json提示, 要求最后一个是`int`类型.

这里还允许例外, 即看似一个路由包含了另一个路由, 这在通用匹配下是有可能发生的. 但只要声明顺序正确, 还是能够被区分开.

## 拓扑序

```python
@app.get("/users/me")
async def read_user_me():
    return {"user_id": "the current user"}


@app.get("/users/{user_id}")
async def read_user(user_id: str):
    return {"user_id": user_id}
```

上述代码中, 访问`host:port/users/me`的时候会有不一样的返回值, 其他的则会有各自的返回模板. 这两个路由函数交换顺序不会导致报错, 但此时由于通用模板在前面, 因此`read_user`的路由将永远无法触发.

## enum

```python
from enum import Enum

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet = "resnet"
    lenet = "lenet"

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name is ModelName.alexnet:
        return {"model_name": ModelName.alexnet.value, "message": "Deep Learning FTW!"}

    if model_name.value == "lenet":
        return {"model_name": ModelName.resnet, "message": "LeCNN all the images"}

    return {"model_name": model_name.value, "message": "Have some residuals"}
```

有时候我们只需要特定的几个输入, 如这里只需要`alexnet, resnet, lenet`三个输入, 则可以使用enum将枚举种类先包装, 这个类将会用于判定`model_name`. 若不属于该类, 则会和之前一样被判定为type error.

## 包含路径

有时候参数内会有路径, 如`/files/images/01.jpg`. 其中包含斜杠`/`, 此前的方案会有问题, 因此需要如下

```python
@app.get("/files/{file_path:path}")
async def read_file(file_path: str):
    return {"file_path": file_path}
```
这里可以看到在`get`内和之前不一样, 指定了是path. 这样内部的斜杠`/`会被忽略.

# 查询参数

所谓查询参数, 指的是get请求`?`跟着的一连串参数, 就构成上和前面差不太多, 如下

```python
fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
async def read_item(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

这里指定了默认的`skip`和`limit`值, 倘若不指定, 且查询参数里又没有对应的值, 则会报错. 有时候我们需要某些参数空缺, 就会如下

```python
@app.get("/items/{item_id}")
async def read_item(item_id: str, q: str | None = None, short: bool = False):
    item = {"item_id": item_id}
    if q:
        item.update({"q": q})
    if not short:
        item.update(
            {"description": "This is an amazing item that has a long description"}
        )
    return item
```

在上述情况下, `q`参数可以为一个字符串, 若空缺则默认为`None`. 同时, 这里指定的`short`为`bool`类型, 但实际中`short`为`1, true, on, yes`及其大小写都可以被正确识别.