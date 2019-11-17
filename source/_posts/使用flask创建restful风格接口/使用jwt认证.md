---
title: "3. 使用jwt认证"
date: 2019-11-03T00:00:00+08:00
categories:
- 使用flask创建restful风格接口
---

完成接口目录和业务层级的布局之后，就可以开心的实现接口逻辑，主要围绕着数据库的增删改查实现，但是事实上通过接口所有人都可以访问，修改，删除数据库中的数据，这样十分危险，必须提供一种方式标识出访问者的身份，根据身份确定访问者是否可以访问，修改，删除对应的数据。

<!-- more -->

> ***项目地址：<https://github.com/defpis/build_restful_api_with_flask>***

本节实现的目标：

* 创建User表，提供用户注册，用户登录等接口，包含最基本的数据库操作
* 先实现使用jwt认证，授权将在之后介绍
* 参数解析
* 模型序列化

依赖安装

```bash
pipenv install flask_sqlalchemy # 数据库orm
pipenv install flask-migrate # 数据库迁移
pipenv install marshmallow-sqlalchemy # 模型序列化
pipenv install flask-marshmallow # 模型序列化
pipenv install webargs # 参数解析
pipenv install flask-jwt-extended # jwt认证
```

在extensions.py中实例化插件

```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from flask_marshmallow import Marshmallow

db = SQLAlchemy()
session = db.session
migrate = Migrate()
jwt = JWTManager()
ma = Marshmallow()
```

之后在create_app中初始化

```python
...
from .extensions import db, migrate, jwt, ma


def create_app(env: str) -> Flask:
		...
    
    # extensions
    db.init_app(app)
    migrate.init_app(app, db)
    jwt.init_app(app)
    ma.init_app(app)
    
    ...
    
    return app
```

创建base.py定义抽象基类，抽离公用字段

> app/api_v1/models/base.py

```python
from sqlalchemy.ext.declarative import declared_attr
from app.extensions import db
from app.utils import get_now, camel_to_underline, get_attr_on_g


class Base(db.Model):
    __abstract__ = True

    app_id = db.Column(db.Integer(), default=get_attr_on_g('app_id'))
    creator_id = db.Column(db.Integer(), default=get_attr_on_g('user_id'))
    created_time = db.Column(db.DateTime(), default=get_now)
    modifier_id = db.Column(db.Integer(), onupdate=get_attr_on_g('user_id'))
    last_updated_time = db.Column(db.DateTime(), onupdate=get_now)
    is_deleted = db.Column(db.Boolean(), default=False)

    # noinspection PyMethodParameters
    @declared_attr
    def __tablename__(cls) -> str:
        return camel_to_underline(cls.__name__)
```

然后定义用户模型user.py

>app/api_v1/models/user.py

```python
from werkzeug.security import generate_password_hash, check_password_hash
from app.extensions import jwt
from app.errors import UserNotExisted
from .base import Base, db


class User(Base):
    id = db.Column(db.Integer(), primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)

    @property
    def password(self) -> None:
        raise AttributeError('password is not a readable attribute')

    @password.setter
    def password(self, password: str) -> None:
        self.password_hash = generate_password_hash(password)

    def verify_password(self, password: str) -> bool:
        return check_password_hash(self.password_hash, password)

    @staticmethod
    @jwt.user_loader_callback_loader
    def get_current_user(user_id):
        user = User.query.filter_by(id=user_id, is_deleted=False).first()
        if not user:
            raise UserNotExisted()
        return user
```

在app/api_v1/models/\_\_init\_\_.py中导出模型

> app/api_v1/models/\_\_init\_\_.py

```python
from .user import User
```

修改configs.py，添加数据库配置

```python
class SqlalchemyConfig:
    SQLALCHEMY_DATABASE_URI = f"sqlite:///{ROOT_PATH}/db.sqlite"
    SQLALCHEMY_TRACK_MODIFICATIONS = True


class DevelopmentConfig(BasicConfig, SqlalchemyConfig):
    ...
```

代码编写完成，开始创建数据库表，在项目根目录下执行

```bash
flask db init
flask db migrate
flask db upgrade
```

通过PyCharm自带的工具连接sqlite查看表

![user table.jpg](https://i.loli.net/2019/11/09/14slZRKJLgTH5qF.jpg)

数据库表生成好了，开始编写注册逻辑

在resources下新建一个auth目录存放认证相关接口

> app/api_v1/resources

```text
.
├── __init__.py
├── auth
│   ├── __init__.py
│   ├── login.py
│   └── register.py
└── demo
    ├── __init__.py
    └── greet.py
```

先创建注册接口（相关依赖查看项目代码自己修复）

> app/api_v1/resources/auth/register.py

```python
from typing import Dict, Any
from flask_marshmallow import Schema
from flask_restful import Resource
from marshmallow import INCLUDE
from app.core import parse, fields
from app.extensions import session
from app.api_v1.schemas import UserSchema


class RegisterSchema(Schema):
    username = fields.String(required=True, error_messages={'required': '用户名是必要的'})
    password = fields.String(required=True, error_messages={'required': '密码是必要的'})


class RegisterResource(Resource):
    schema = UserSchema(exclude=('password_hash',))

    @parse(RegisterSchema)
    def get(self, data: Dict) -> Any:
        return self._register(data)

    @parse(RegisterSchema)
    def post(self, data: Dict) -> Any:
        return self._register(data)

    def _register(self, data: Dict) -> Dict:
        user = self.schema.load(data, unknown=INCLUDE)
        session.add(user)
        session.commit()
        return self.schema.dump(user)
```

然后创建登录接口

> app/api_v1/resources/auth/login.py

```python
from typing import Dict, Any
from flask_restful import Resource
from flask_jwt_extended import create_access_token, create_refresh_token
from flask_marshmallow import Schema
from app.core import parse, fields
from app.errors import UserNotExisted, PasswordInvalid
from app.api_v1.models import User
from app.api_v1.schemas import UserSchema


class AuthenticateMixin(object):
    @staticmethod
    def authenticate(username: str, password: str) -> User:
        user = User.query.filter_by(username=username, is_deleted=False).first()
        if not user:
            raise UserNotExisted()
        if not user.verify_password(password):
            raise PasswordInvalid()
        return user

    @staticmethod
    def obtain_token(user: User) -> Dict:
        return dict(
            access_token=create_access_token(identity=user.id),
            refresh_token=create_refresh_token(identity=user.id)
        )


class LoginSchema(Schema):
    username = fields.String(required=True, error_messages={'required': '用户名是必要的'})
    password = fields.String(required=True, error_messages={'required': '密码是必要的'})


class LoginResource(Resource, AuthenticateMixin):
    schema = UserSchema(exclude=('password_hash',))

    @parse(LoginSchema)
    def get(self, data: Dict) -> Any:
        return self._login(data)

    @parse(LoginSchema)
    def post(self, data: Dict) -> Any:
        return self._login(data)

    def _login(self, data: Dict) -> Any:
        user = self.authenticate(data['username'], data['password'])
        return dict(
            **self.schema.dump(user),
            **self.obtain_token(user)
        )
```

需要定义UserSchema来序列化和反序列化，绑定当前的session对象，使之load回来的对象是orm的实例

> app/api_v1/schemas/user.py

```python
from app.extensions import ma, session
from ..models import User


class UserSchema(ma.ModelSchema):
    class Meta:
        model = User
        sqla_session = session
```

最后依次往上注册路由

> app/api_v1/resources/auth/\_\_init\_\_.py

```python
from app.core import Router
from . import login, register

router = Router(decorators=[])
router.url(
    (('/login', '/obtain_token'), login.LoginResource, {"endpoint": "login"}),
    ('/register', register.RegisterResource, {"endpoint": "register"}),
)
```

> app/api_v1/resources/\_\_init\_\_.py

```python
from app.core import Router
from . import demo, auth

router = Router(decorators=[])
router.url(
    ('/demo', demo.router),
    ('/auth', auth.router),
)
```

参数解析使用到了库webargs，在app.core中定义了模块fields.py和parser.py

> app/core/fields.py

```python
from marshmallow.fields import *
from webargs.fields import Nested  # type: ignore
```

> app/core/parser.py

```python
from typing import Dict, Callable
from functools import partial
from marshmallow import Schema
from marshmallow.exceptions import ValidationError
from webargs.flaskparser import parser
from flask import Request
from app.errors import ValidateError


@parser.error_handler
def handle_error(error: ValidationError, req: Request, schema: Schema, status_code: int, headers: Dict) -> None:
    raise ValidateError(**error.messages) from error  # type: ignore


parse: Callable = partial(parser.use_args, locations=('querystring', 'json'))
```

> app/core/\_\_init\_\_.py

```python
...
from .parser import parse
from . import fields
```

自定义异常在app/errors.py中

> app/errors.py

```python
from typing import Dict, Type
from flask import Flask, jsonify
from .log import log_error


class CustomErrorHandler(object):
    def __init__(self):
        self.custom_errors = []

    def __call__(self, error_class: Type['CustomError']):
        self.custom_errors.append(error_class)
        return error_class

    def init_app(self, app: Flask):
        for error_class in self.custom_errors:
            error_handler = log_error(error_class.generate_response)
            app.errorhandler(error_class)(error_handler)


custom_error_handler = CustomErrorHandler()


@custom_error_handler
class CustomError(Exception):
    code = 500
    message = "自定义错误"
    errors: Dict = {}

    def __init__(self, **kwargs):
        self.errors = kwargs

    def generate_response(self):
        resp_dict = {
            'message': self.message
        }
        if self.errors:
            resp_dict['errors'] = self.errors
        return jsonify(resp_dict), self.code


@custom_error_handler
class UserNotExisted(CustomError):
    code = 404
    message = "用户不存在或已删除"


@custom_error_handler
class PasswordInvalid(CustomError):
    code = 400
    message = "密码不合法"


@custom_error_handler
class ValidateError(CustomError):
    code = 400
    message = "参数不合法"
```

需要在create_app中注册

```python
...
from .errors import custom_error_handler


def create_app(env: str) -> Flask:
		...
    
    custom_error_handler.init_app(app)

    return app
```

还需要配置一下jwt

```python
class JwtConfig:
    JWT_SECRET_KEY = "you-will-never-guess"
    JWT_ERROR_MESSAGE_KEY = "message"
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(days=1) # access_token有效期1天
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=30) # refresh_token有效期30天
    JWT_COOKIE_CSRF_PROTECT = False
    JWT_CSRF_IN_COOKIES = True
    JWT_ACCESS_COOKIE_PATH = "/"
    JWT_REFRESH_COOKIE_PATH = "/"
    JWT_ACCESS_CSRF_HEADER_NAME = "X-CSRF-TOKEN"
    JWT_REFRESH_CSRF_HEADER_NAME = "X-CSRF-TOKEN"
    JWT_TOKEN_LOCATION = ["headers", "cookies", "query_string", "json"]
    JWT_CSRF_METHODS = ["POST", "PUT", "PATCH", "DELETE"]
    JWT_BLACKLIST_ENABLED = False

class DevelopmentConfig(BasicConfig, SqlalchemyConfig, JwtConfig):
    ...
```

测试注册接口

![Xnip2019-11-09_14-27-59.jpg](https://i.loli.net/2019/11/09/IaqRrSf2VTeObB5.jpg)

测试登录接口

![Xnip2019-11-09_14-28-21.jpg](https://i.loli.net/2019/11/09/kXOznHTZYjwby2G.jpg)

在我们的demo接口上添加jwt_required装饰器验证jwt

> app/api_v1/schemas/\_\_init\_\_.py

```python
from app.core import Router
from . import demo, auth
from flask_jwt_extended import jwt_required

router = Router(decorators=[])
router.url(
    ('/demo', demo.router, [jwt_required]),
    ('/auth', auth.router),
)
```

访问<http://127.0.0.1:5000/v1/demo/hello>

```json
{
  "request_id": "7c9cbc53-bd25-46ab-9a17-cbb1c7dd91c0",
  "message": "Missing Authorization Header"
}
```

为此接口添加之前登录返回的token，再次请求成功获取结果

![Xnip2019-11-09_14-35-34.jpg](https://i.loli.net/2019/11/09/JCVPdkQqZwDWeth.jpg)

下一节将介绍使用flask_sqlalchemy完成复杂查询。
