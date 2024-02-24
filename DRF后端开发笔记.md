# DRF后端开发笔记

## 1.环境准备/项目创建

### 环境安装

```python
# django
pip install django==4.2 Pillow  
# 数据库
pip install mysqlclient django-redis redis
# drf相关组件
pip install djangorestframework djangorestframework-simplejwt
```

### 项目创建

```python
# 创建项目
django-admin startproject PRO_NAME
# 创建应用
django-admin startapp APP_NAME
```



## 2.相关配置

### settings

* 可以在项目根目录下创建settings文件夹，用来存放开发和上线用的两套settings（dev/prod）

* 在settings中注册创建的app

* 注释crf验证中间件

* 注册跨域中间件

  ```python
  MIDDLEWARE = [
  	...
      # 跨域中间件
      'corsheaders.middleware.CorsMiddleware',
  ]
  
  # 允许所有用户进行跨域请求
  CORS_ORIGIN_ALLOW_ALL = True
  
  # 指定自定义用户类
  AUTH_USER_MODEL = 'users.User'
  ```

### 数据库

* mysql

  ```sql
  mysql -u root -p
  # 输入密码
  show databases;
  create database DB_NAME charset=utf8;
  ```

  ```python
  DATABASES = {
      'default': {
          'ENGINE': 'django.db.backends.mysql',
          'NAME': DB_NAME,
          'USER': 'root',
          'PASSWORD': 'root',
          'PORT': 3306,
          'HOST': 'localhost',
      }
  }
  ```

  

## 3.公共字段

在项目根目录创建common文件加存放公共字段

```python
# db.py
from django.db import models


class BaseModel(models.Model):
    """抽象模型基类"""
    create_time = models.DateTimeField(auto_now_add=True, verbose_name='创建时间')
    update_time = models.DateTimeField(auto_now=True, verbose_name='更新时间')
    is_delete = models.BooleanField(default=False, verbose_name='删除标记')

    class Meta:
        abstract = True
        verbose_name_plural = "公共字段模型"
        db_table = 'BaseTable'
   
```



## 4.用户模块

* models

```python
# .users.models.py
from django.db import models
from common.db import BaseModel
from django.contrib.auth.models import AbstractUser

# Create your models here.


class User(AbstractUser, BaseModel):
    """用户模型"""
    mobile = models.CharField(verbose_name="手机号码", default='', max_length=11)
    avatar = models.ImageField(verbose_name="用户头像", blank=True, null=True)

    class Meta:
        db_table = 'users'
        verbose_name = '用户表'


class Addr(models.Model):
    """收获地址"""
    user = models.ForeignKey(verbose_name="所属用户", to='User', on_delete=models.CASCADE)
    phone = models.CharField(verbose_name="收货手机", max_length=11)
    name = models.CharField(verbose_name="联系人", max_length=32)
    province = models.CharField(verbose_name="省份", max_length=32)
    city = models.CharField(verbose_name="城市", max_length=32)
    region = models.CharField(verbose_name="区县", max_length=32)
    address = models.CharField(verbose_name="详细地址", max_length=32)
    is_default = models.BooleanField(verbose_name="是否为默认收货地址", default=False)

    class Meta:
        db_table = 'addr'
        verbose_name = "收获地址表"


class Area(models.Model):
    """省市区县地址模型"""
    pid = models.IntegerField(verbose_name="上级id")
    name = models.CharField(verbose_name="地区名", max_length=32)
    level = models.CharField(verbose_name="地域等级", max_length=1)

    class Meta:
        db_table = 'area'
        verbose_name = "地区表"


class VerifCode(models.Model):
    """验证码模型"""
    mobile = models.CharField(verbose_name="手机号码", max_length=11)
    code = models.CharField(verbose_name="验证码", max_length=6)
    create_time = models.DateTimeField(auto_now_add=True, verbose_name='创建时间')

    class Meta:
        db_table = 'verifcode'
        verbose_name = "验证码表"
```

* token实现

```python
# settings.py
# 1.注册
INSTALLED_APPS = [
	...
    'rest_framework_simplejwt',
]

# 2.DRF配置鉴权方式
REST_FRAMEWORK = {
    # 配置鉴权方式
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}

SIMPLE_JWT = {
    # token相关配置
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),  # 访问令牌效期
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),  # 刷星令牌效期
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,

    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'VERIFYING_KEY': None,
    'AUDIENCE': None,
    'ISSUER': None,

    'AUTH_HEADER_TYPES': ('Bearer',),
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
}
```

### 登录

#### 简单登录

```python
# .apps.users.view.py
class LoginView(TokenObtainPairView):
    """用户登录"""

    def post(self, request: Request, *args, **kwargs) -> Response:
        serializer = self.get_serializer(data=request.data)

        try:
            serializer.is_valid(raise_exception=True)
        except TokenError as e:
            raise InvalidToken(e.args[0])
        # 自定义登陆成功之后返回结果
        result = serializer.validated_data
        result['id'] = serializer.user.id
        result['mobile'] = serializer.user.mobile
        result['email'] = serializer.user.email
        result['username'] = serializer.user.username

        result['token'] = result.pop('access')

        return Response(result, status=status.HTTP_200_OK)
```

#### 多字段登录

* 重写authentication

```python
# .common.authenticaton.py
from django.db.models import Q
from rest_framework import serializers

from apps.users.models import User


class Authentication(ModelBackend):
    """自定义登录认证类"""

    def authenticate(self, request, username=None, password=None, **kwargs):
        """支持使用手机h号/邮箱/用户名登录"""
        try:
            user = User.objects.get(Q(username=username) | Q(mobile=username) | Q(email=username))
        except:
            raise serializers.ValidationError({'msg': "未找到该用户！"})

        # 判断密码
        if user.check_password(password):
            return user
        else:
            raise serializers.ValidationError({'msg': "密码错误"})
```

* 配置认证方式

  ```python
  # .settings
  # 使用自定义的认证类进行身份认证
  AUTHENTICATION_BACKENDS = [
      'common.authentication.Authentication',
  ]
  ```

  





